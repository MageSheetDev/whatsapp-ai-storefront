---
title: "Build a Real-Time Inventory Dashboard in Google Sheets: Reorder Points &amp; Low-Stock Alerts"
seoTitle: "Build a Real-Time Inventory Dashboard in Google Sheets"
seoDescription: "Build a real-time inventory dashboard in Google Sheets with Apps Script: reorder-point math, available-vs-allocated stock, and low-stock alerts."
datePublished: 2026-07-17T16:56:08.366Z
cuid: cmrp6jctg00000ajl4lmjfo5n
slug: build-a-real-time-inventory-dashboard-in-google-sheets-reorder-points-amp-low-stock-alerts
canonical: https://magesheet.com/blog/ecommerce-realtime-inventory-dashboard
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/a721cb2f-7a46-472b-9299-32fd05d69f3a.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/7aec3729-58b4-46e3-84c4-ee0485d696bc.jpg
tags: ecommerce, automation, google-sheets, google-apps-script, magesheet

---

A marketing campaign works, 150 units sell in a day — and 40 were in stock. Now you have 110 angry customers, refunds, and a marketplace penalty for overselling. The campaign that was supposed to make money just cost you money and trust.

The fix isn't a $200/month inventory app. It's a Google Sheet that knows your reorder point per product, tells the difference between stock you *have* and stock you can actually *sell*, and emails you before you run out. This post builds it, with the code.

## Why manual stock tracking guarantees a stockout

Across a handful of SKUs you can eyeball it. Across a few hundred, you can't — each product has a different sales rate and a different supplier lead time, so "when do I reorder this one?" is a different answer for every row. Do that math by hand and you will be late on something.

There's one formula that removes the guessing — the reorder point:

> **ROP = (average daily sales × supplier lead time in days) + safety stock**

If a product sells ~10/day, your supplier takes 14 days, and you keep 20 units of buffer: `10 × 14 + 20 = 160`. When available stock hits 160, you reorder — and the shipment lands just as you'd otherwise run dry.  
  

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/86e9d8c9-a664-48ba-b643-0efe07ec624b.jpg align="center")

Set up an `Inventory` sheet with these columns and a matching constant:

```javascript
// Inventory columns:
// sku | name | onHand | allocated | avgDailySales | leadTimeDays |
// safetyStock | available | rop | status
const INV = {
  SKU: 1, NAME: 2, ON_HAND: 3, ALLOC: 4, AVG: 5, LEAD: 6,
  SAFETY: 7, AVAIL: 8, ROP: 9, STATUS: 10,
};
```

## 1\. The math: reorder point and the stock that actually matters

Three small functions carry the whole dashboard. The one people skip is `availableStock` — the difference between what's on the shelf and what's already promised to open orders. Overselling happens when you reorder against *on-hand* instead of *available*.

```javascript
function reorderPoint(avgDailySales, leadTimeDays, safetyStock) {
  return avgDailySales * leadTimeDays + safetyStock;
}

function availableStock(onHand, allocated) {
  return onHand - allocated;   // what you can sell, not what's on the shelf
}

function stockStatus(onHand, allocated, rop) {
  const avail = availableStock(onHand, allocated);
  if (avail <= 0) return 'OUT';
  if (avail <= rop) return 'REORDER';
  return 'OK';
}
```

100 units on hand with 90 already allocated to unshipped orders is not "100 in stock" — it's 10. `stockStatus` reads the available number, so it flags REORDER while the shelf still looks full.

## 2\. Refresh the whole dashboard in one pass

This recomputes available, ROP, and status for every row and writes them back in a single `setValues` call — fast, and gentle on Apps Script's quotas. Run it on an hourly trigger, or fire it from the Magento webhook that updates `onHand` and `allocated`.

```javascript
function refreshDashboard() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Inventory');
  const last = sheet.getLastRow();
  if (last < 2) return;

  const range = sheet.getRange(2, 1, last - 1, INV.STATUS);
  const rows = range.getValues();

  rows.forEach(r => {
    const onHand = Number(r[INV.ON_HAND - 1]) || 0;
    const alloc  = Number(r[INV.ALLOC - 1]) || 0;
    const rop = reorderPoint(
      Number(r[INV.AVG - 1]) || 0,
      Number(r[INV.LEAD - 1]) || 0,
      Number(r[INV.SAFETY - 1]) || 0
    );
    r[INV.AVAIL - 1]  = availableStock(onHand, alloc);
    r[INV.ROP - 1]    = rop;
    r[INV.STATUS - 1] = stockStatus(onHand, alloc, rop);
  });

  range.setValues(rows);   // one write for the whole sheet
}
```

Reading and writing the whole range once (instead of cell by cell in a loop) is the difference between a dashboard that refreshes in a second and one that trips the 6-minute limit at a few thousand SKUs.

## 3\. Get told before you run out

A dashboard nobody looks at doesn't prevent a stockout. A daily trigger on `sendLowStockAlert` emails whoever handles purchasing the moment anything crosses into REORDER or OUT.

```javascript
function sendLowStockAlert() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Inventory');
  const last = sheet.getLastRow();
  if (last < 2) return;

  const low = sheet.getRange(2, 1, last - 1, INV.STATUS).getValues()
    .filter(r => r[INV.STATUS - 1] === 'REORDER' || r[INV.STATUS - 1] === 'OUT')
    .map(r => `${r[INV.STATUS - 1]}  ${r[INV.SKU - 1]}  ${r[INV.NAME - 1]}  ` +
              `(available ${r[INV.AVAIL - 1]}, reorder point ${r[INV.ROP - 1]})`);

  if (!low.length) return;   // nothing to nag about today

  MailApp.sendEmail({
    to: PropertiesService.getScriptProperties().getProperty('ALERT_EMAIL'),
    subject: `⚠ ${low.length} product(s) need reordering`,
    body: 'Reorder these now:\n\n' + low.join('\n'),
  });
}
```

I unit-tested the math before shipping — the ROP worked example lands on 160, and the status boundaries (available exactly at the ROP, exactly at zero, and oversold into negatives) all behave.

## Pitfalls

*   **Reorder against *available*, not *on-hand*.** The allocated column is what stops overselling; skip it and the dashboard lies to you while the shelf looks full.
    
*   **Keep** `avgDailySales` **fresh.** A number set once goes stale — recompute it from a rolling window (e.g. the last 14–30 days) so a seasonal spike actually moves the reorder point.
    
*   **Read/write the range once.** Cell-by-cell `getValue`/`setValue` in a loop is the classic way to hit the 6-minute execution cap on a big catalog.
    
*   **Dedupe the Magento webhook.** Sync events can arrive twice or out of order; key updates by SKU and ignore stale ones.
    
*   **Know the ceiling.** Sheets stays responsive into the thousands of SKUs and low-to-mid six-figure monthly orders; past that, keep the sheet as the dashboard and move the source of truth to a database.
    

## Wrap-up

A real-time inventory dashboard is three ideas: a reorder point per product, the discipline to reorder against available (not on-hand) stock, and an alert that reaches you before the shelf is empty. That's a few dozen lines of Apps Script on a sheet you already own — not a $200/month subscription.

The production version — Magento two-way sync, per-warehouse stock, and supplier-aware lead times — is written up on the [MageSheet blog](https://magesheet.com/blog/ecommerce-realtime-inventory-dashboard).

Built by the MageSheet team.