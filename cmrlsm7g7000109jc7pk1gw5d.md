---
title: "Build a Gumroad Lead Machine in Google Sheets: Ping Webhook, License Check &amp; AI Upsell"
seoTitle: "Build a Gumroad Lead Machine in Google Sheets (Apps Script)"
seoDescription: "Turn Gumroad micro-purchases into qualified B2B leads: an Apps Script Ping webhook, license verification, and AI-personalized upsell emails."
datePublished: 2026-07-15T08:03:08.224Z
cuid: cmrlsm7g7000109jc7pk1gw5d
slug: build-a-gumroad-lead-machine-in-google-sheets-ping-webhook-license-check-amp-ai-upsell
canonical: https://magesheet.com/blog/gumroad-lead-generation-machine
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/68d41faf-993f-481e-a378-f40359ffb1b5.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/d43f3762-cb0b-4585-8817-1f8120205516.jpg
tags: google-sheets, google-apps-script, gumroad, magesheet

---

A traditional B2B funnel asks a stranger to fill a form, wait for a sales rep, and sit through a call — and it typically converts around 0.1–0.5%. I stopped running that. Instead I sell a small $29 product on Gumroad and let the purchase itself qualify the lead.

This post is the Apps Script that turns every Gumroad sale into a logged, license-verified lead with an AI-personalized follow-up email — no CRM, no Zapier, about 70 lines you can copy.

## Why free lead magnets and demo forms both fail

A free download gets you emails from people who spent nothing, so they're not qualified. A demo form gets you almost nobody, because the friction is huge. A small paid "micro-commitment" sits in the middle: the buyer proved intent with their wallet, and you now have their email, the exact product they bought, and a valid license — everything you need to follow up with the right offer.

Gumroad handles the checkout. Google Sheets is the lead database. Apps Script is the glue: catch the sale, verify it's real, and send the next step.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/6e4161ff-18b4-4712-bf84-7dcd9de0778e.png align="center")

  
  
Make a sheet named `Leads` and add this so the code and the columns stay in sync:

```javascript
// Leads columns: timestamp | saleId | email | product | amount | license | status
const COL = { TS:1, SALE_ID:2, EMAIL:3, PRODUCT:4, AMOUNT:5, LICENSE:6, STATUS:7 };
```

In Gumroad, enable the ping under Settings → Advanced → "Ping", and point it at your Web App URL with a secret token in the query string: `https://script.google.com/.../exec?token=my-secret`.

## 1\. Catch the sale: the Ping webhook

The one thing to get right: **Gumroad posts** `application/x-www-form-urlencoded`**, not JSON.** So you read `e.parameter`, not `JSON.parse(e.postData.contents)`. This trips up everyone who copied a JSON webhook example.

```javascript
function doPost(e) {
  const p = e.parameter;   // Gumroad is form-encoded — use e.parameter, NOT JSON.parse

  const expected = PropertiesService.getScriptProperties().getProperty('GUMROAD_TOKEN');
  if (expected && p.token !== expected) {
    return ContentService.createTextOutput('unauthorized');
  }

  const sale = saleToRow(p);
  if (!sale.saleId) return ContentService.createTextOutput('ignored');

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Leads');

  // Gumroad can resend a ping — dedupe by sale_id.
  const seen = sheet.getRange(2, COL.SALE_ID, Math.max(sheet.getLastRow() - 1, 1), 1)
                    .getValues().flat();
  if (seen.indexOf(sale.saleId) !== -1) return ContentService.createTextOutput('duplicate');

  sheet.appendRow([new Date(), sale.saleId, sale.email, sale.product, sale.amount, sale.license, sale.status]);
  return ContentService.createTextOutput('ok');
}

function saleToRow(p) {
  return {
    saleId:  p.sale_id || '',
    email:   p.email || '',
    product: p.product_name || '',
    amount:  Number(p.price || 0) / 100,   // Gumroad sends price in cents
    license: p.license_key || '',
    status:  'new'
  };
}
```

## 2\. Prove it's real: verify the license

A ping is just an HTTP POST — anyone who finds your URL can fake one. Before you provision anything valuable, verify the license against Gumroad's API. It returns the real purchase record, or nothing.

```javascript
function verifyLicense(productPermalink, licenseKey) {
  const res = UrlFetchApp.fetch('https://api.gumroad.com/v2/licenses/verify', {
    method: 'post',
    muteHttpExceptions: true,
    payload: {
      product_permalink: productPermalink,
      license_key: licenseKey,
      increment_uses_count: 'false'   // just checking — don't burn a use
    }
  });
  const data = JSON.parse(res.getContentText());
  return data.success ? data.purchase : null;   // buyer + product details, or null
}
```

## 3\. Follow up: the AI-personalized upsell

A time trigger (Triggers → Add trigger → `sendUpsells`, every 5 minutes) emails each new lead once, then marks it so it never double-sends. The message is written per-product by `gpt-4o-mini` — under $0.01 per email — and falls back to a plain template if no API key is set.

```javascript
function sendUpsells() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Leads');
  const last = sheet.getLastRow();
  if (last < 2) return;

  const rows = sheet.getRange(2, 1, last - 1, COL.STATUS).getValues();
  rows.forEach((row, i) => {
    if (row[COL.STATUS - 1] !== 'new') return;   // only fresh leads
    const email = row[COL.EMAIL - 1];
    const product = row[COL.PRODUCT - 1];
    const message = personalize(product);        // one API call
    GmailApp.sendEmail(email, `Your next step after ${product}`, message, { name: 'Hayrullah at MageSheet' });
    sheet.getRange(i + 2, COL.STATUS).setValue('emailed');
  });
}

function personalize(product) {
  const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
  if (!apiKey) {
    return `Thanks for grabbing ${product}. Reply to this email and I'll send the advanced setup.`;
  }
  const res = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', {
    method: 'post', contentType: 'application/json',
    headers: { Authorization: 'Bearer ' + apiKey }, muteHttpExceptions: true,
    payload: JSON.stringify({
      model: 'gpt-4o-mini', temperature: 0.7,
      messages: [{ role: 'user',
        content: `Write a friendly 3-sentence follow-up email to someone who just bought "${product}". ` +
                 `Offer a paid done-for-you upgrade. No subject line, no placeholders.` }]
    })
  });
  if (res.getResponseCode() !== 200) {
    return `Thanks for grabbing ${product}. Happy to help you take it further — just reply.`;
  }
  return JSON.parse(res.getContentText()).choices[0].message.content;
}
```

I unit-tested `saleToRow` before shipping — the price field is the trap, because Gumroad reports it in cents (`2900` means $29.00), and a missing field must default cleanly instead of writing `NaN` into the sheet.

## Pitfalls I hit (so you don't)

*   **Gumroad is form-encoded, not JSON.** Read `e.parameter.email`, not `JSON.parse(e.postData.contents)`. This is the single most common mistake.
    
*   `doPost(e)` **can't read HTTP headers.** Apps Script exposes query/form params only, so you can't check a signature header — put a secret `?token=` in the ping URL and compare `e.parameter.token`.
    
*   **Never trust the ping alone.** Verify the license before provisioning anything a spoofed POST could steal.
    
*   **Deduplicate.** Gumroad resends pings on timeout; the `sale_id` check stops duplicate leads.
    
*   **Gmail send limits.** ~100 emails/day on consumer accounts, ~1,500/day on Workspace. Above that, queue and throttle.
    
*   **Gumroad takes ~10%.** Price your micro-commitment so the fee still leaves the offer worth running.
    

## Wrap-up

That's an autonomous lead machine in about 70 lines: a Gumroad checkout becomes a logged, verified lead with a personalized upsell — all inside a Google Sheet, with the micro-commitment doing your lead qualification for you.

The production version — ad attribution back to the sale, license provisioning, and a multi-step follow-up sequence — is written up on the [MageSheet blog](https://magesheet.com/blog/gumroad-lead-generation-machine).

Built by the MageSheet team.