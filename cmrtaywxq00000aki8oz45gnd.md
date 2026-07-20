---
title: "Build a WhatsApp Storefront in Google Sheets: OpenAI Function-Calling Tools (No Website)"
seoTitle: "Build a WhatsApp Storefront in Google Sheets (No Website)"
seoDescription: "Turn WhatsApp into a mini storefront with Google Sheets and OpenAI function calling — three Apps Script tools to browse products and log orders."
datePublished: 2026-07-20T14:11:17.452Z
cuid: cmrtaywxq00000aki8oz45gnd
slug: whatsapp-storefront-google-sheets-openai
canonical: https://magesheet.com/blog/whatsapp-mini-storefront-google-sheets
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/06b04901-c278-4b10-bdee-48461f330a06.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/4d487c32-82d7-478b-9b4a-10c81df9dd81.jpg
tags: google-sheets, whatsapp, google-apps-script, openai, magesheet

---

A café owner asked me for an online store. She didn't want Shopify, a domain, a theme, or a $90/month bill — she wanted customers to message her on WhatsApp, ask what's available, and place an order. That's not a website. That's three functions and a spreadsheet.

Here's the whole thing: an AI that browses your catalog and logs orders over chat, backed by two Google Sheets tabs, for under $20/month. It's about 200 lines of Apps Script, and the three tools that do the work are below — verbatim.

## Why function calling is the right tool here

A keyword bot ("reply 1 for menu") falls apart the moment a customer types "what pastries do you have under 50?". A language model handles that phrasing easily — but on its own it can only *talk*. To actually read your catalog and write an order, it needs **tools**: functions it's allowed to call.

OpenAI's function calling is exactly that. You hand GPT a list of tool schemas; when it decides it needs data, it returns a `tool_call` naming the tool and its arguments; your Apps Script runs the matching function and hands the result back. Three tools cover a storefront: list products, look up one product, log an order.

You need two tabs. **Products**: `sku | name | description | price | currency | stock | active`. **Orders**: `orderId | phone | sku | name | qty | price | total | currency | notes | status | timestamp`. That's the entire database.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/60ef5cd8-b719-4c21-85ee-f9b82ae05895.jpg align="center")

## Tool 1: read the catalog

The model calls this when a customer wants to browse. It skips anything inactive or out of stock, and treats a blank stock cell as "unlimited" (handy for services and made-to-order items).

```javascript
function toolGetProducts(phone, args, lead) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Products');
  if (!sheet || sheet.getLastRow() < 2) {
    return { products: [], note: 'No products configured yet.' };
  }

  const data = sheet.getDataRange().getValues();
  const products = [];
  for (let i = 1; i < data.length; i++) {
    const [sku, name, desc, price, currency, stock, active] = data[i];
    if (active !== true) continue;
    if (stock !== '' && Number(stock) <= 0) continue;
    products.push({
      sku: String(sku),
      name: String(name),
      description: String(desc || ''),
      price: Number(price),
      currency: String(currency || ''),
      in_stock: stock === '' ? null : Number(stock)
    });
  }
  return { count: products.length, products };
}
```

## Tools 2 & 3: look up a product, then log the order

`toolGetProduct` is a case-insensitive SKU lookup. `toolLogOrder` is the one that touches money, so it validates first, re-reads the product for the authoritative price (never trusting a price the model might invent), computes the total itself, and stamps a unique order id.

```javascript
function toolGetProduct(phone, args, lead) {
  const sku = args && args.sku ? String(args.sku).trim() : '';
  if (!sku) return { error: 'sku is required' };

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Products');
  const data = sheet.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    if (String(row[0]).toLowerCase() === sku.toLowerCase()) {
      return {
        sku: String(row[0]),
        name: String(row[1]),
        description: String(row[2] || ''),
        price: Number(row[3]),
        currency: String(row[4] || ''),
        in_stock: row[5] === '' ? null : Number(row[5]),
        active: row[6] === true
      };
    }
  }
  return { error: 'Product ' + sku + ' not found' };
}

function toolLogOrder(phone, args, lead) {
  const sku = args && args.sku ? String(args.sku).trim() : '';
  const qty = args && args.quantity ? Number(args.quantity) : 0;
  const notes = (args && args.notes) ? String(args.notes) : '';

  if (!sku || !qty || qty < 1) return { error: 'sku and quantity required' };

  const product = toolGetProduct(phone, { sku }, lead);
  if (product.error) return product;

  const total = product.price * qty;                       // server computes it, not the model
  const orderId = 'ORD-' + Date.now().toString(36).toUpperCase();

  SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Orders').appendRow([
    orderId, phone, product.sku, product.name, qty,
    product.price, total, product.currency, notes,
    'pending', new Date().toISOString()
  ]);

  return {
    success: true, order_id: orderId, sku: product.sku,
    product_name: product.name, quantity: qty, total: total,
    currency: product.currency, status: 'pending'
  };
}
```

## The glue: expose the tools and dispatch the model's call

This is what turns three functions into a storefront. You register each tool with the schema the model sees, and route whatever it calls to the matching handler. The `description` on each schema is doing real work — it's how GPT knows *not* to call `log_order` until the customer has actually confirmed.

```javascript
const TOOLS = {
  get_products: {
    handler: toolGetProducts,
    schema: {
      name: 'get_products',
      description: 'List the active, in-stock products this shop sells.',
      parameters: { type: 'object', properties: {} }
    }
  },
  get_product: {
    handler: toolGetProduct,
    schema: {
      name: 'get_product',
      description: 'Look up one product by its SKU.',
      parameters: {
        type: 'object',
        properties: { sku: { type: 'string' } },
        required: ['sku']
      }
    }
  },
  log_order: {
    handler: toolLogOrder,
    schema: {
      name: 'log_order',
      description: 'Log a confirmed order. Only call AFTER the customer confirms the item and quantity.',
      parameters: {
        type: 'object',
        properties: {
          sku: { type: 'string' },
          quantity: { type: 'number' },
          notes: { type: 'string' }
        },
        required: ['sku', 'quantity']
      }
    }
  }
};

// When GPT returns a tool call, run the matching handler and feed the JSON back.
function runToolCall(toolCall, phone, lead) {
  const tool = TOOLS[toolCall.name];
  if (!tool) return { error: 'unknown tool ' + toolCall.name };
  let args = {};
  try { args = JSON.parse(toolCall.arguments || '{}'); }
  catch (e) { return { error: 'bad tool arguments' }; }
  return tool.handler(phone, args, lead);
}
```

The webhook that receives the WhatsApp message (via Twilio) and the loop that calls GPT with `TOOLS` are the same pattern I covered in [Build a WhatsApp Sales Inbox in Google Sheets](https://magesheet.com/blog/automating-whatsapp-sales-google-sheets) — a `doPost` that returns `200` fast and does the model round-trip on the message text. Drop these tools into that loop and you have a store.

I unit-tested the tool logic before shipping — the cases that matter are a blank stock cell reading as unlimited, an out-of-stock item never reaching the catalog, a case-insensitive SKU, and an order with a missing or zero quantity being rejected instead of written.

## The economics

For a shop doing under ~500 messages and ~100 orders a month, the running cost breaks down to roughly $5–10 Twilio, $2–5 OpenAI on GPT-4o-mini, and $0–6 Google — call it under $20/month. A Shopify-plus-app stack for the same thing runs $80–140/month combined. The bigger win is that the orders live in *your* Drive, not a vendor's database.

## When NOT to use this

*   **Thousands of SKUs, or a catalog that changes hourly.** This shines at 5–30 stable products; past that you want real inventory sync and a database.
    
*   **Variants, tax rules, and international shipping.** The model will happily log an order; it won't handle size/color matrices or VAT correctly.
    
*   **Taking payment in-chat.** This logs a *pending* order. Bolt on a Stripe payment link if you need money to move; don't fake a checkout.
    
*   **Letting the model set prices.** Note that `toolLogOrder` re-reads the price from the sheet. Never trust a total the model calculated — recompute it server-side, always.
    

## Wrap-up

An online store, for a lot of small sellers, is three functions: list the catalog, look up an item, log the order — wired to GPT through function calling, backed by two spreadsheet tabs. No website, no monthly platform fee, and the data stays yours.

The production version — Stripe payment links, stock decrement on order, and a merchant dashboard — is written up on the [MageSheet blog](https://magesheet.com/blog/whatsapp-mini-storefront-google-sheets).

Built by the MageSheet team.