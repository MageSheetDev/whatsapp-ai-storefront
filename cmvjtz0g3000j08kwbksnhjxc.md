---
title: "Adobe Commerce to Looker Studio: SUM(grand_total) Is Not Revenue"
seoTitle: "Adobe Commerce Revenue in Looker Studio: Net, Not Gross"
seoDescription: "SUM(grand_total) counts cancelled orders, refunds and mixed currencies. A Sheets row for net revenue, plus a store count check."
datePublished: 2026-10-22T17:44:22.285Z
cuid: cmvjtz0g3000j08kwbksnhjxc
slug: adobe-commerce-to-looker-studio-sum-grand-total-is-not-revenue
canonical: https://magesheet.com/blog/adobe-commerce-google-sheets-looker-studio
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/64094f02-044e-48a9-9083-7695b55eba06.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/0ab9cf10-da8e-4ed0-bf1b-369e1d5f98e4.jpg
tags: google-sheets, google-apps-script, adobe-commerce, looker-studio, magesheet

---

The Adobe Commerce to Looker Studio guide on the MageSheet blog ends with a dashboard step: set `grand_total` to a currency and check that the aggregation is Sum. I ran the guide's Apps Script against a fake store, and the column it tells Looker Studio to sum is not revenue. It includes orders the store has since cancelled, it counts refunded money as sold, and for a store that sells in more than one currency it adds euros to dollars.

None of that comes from Looker Studio, which I did not open for this. It comes from what the Orders tab holds. The paging and the `updated_at` sync are already covered in [my earlier post on Magento order sync](https://magesheet.com/blog/magento-google-sheets-automation), so I do not repeat them. This post is about the row itself: which columns a revenue dashboard can trust, and two checks that fail loudly when the tab drifts from the store.

## What the recipe leaves in the sheet

The guide's `importOrders()` appends one row per order, reads `grand_total` with `Number()`, and moves its cursor to the `created_at` of the last row it wrote. I copied that function from the page unchanged and ran it in Node against a fake Adobe Commerce API that honours the same `searchCriteria` filter. Two pending orders came in on the first run. Then I cancelled the first order in the fake store, added a third, and ran it again.

The sheet kept the first order as `pending`. The cursor filters on `created_at > cursor`, so an order that was already read never comes back, and a status change does not move `created_at`. The third order arrived as expected. The `grand_total` column, the one the guide has Looker Studio sum, came to 242: 100 USD, 92 EUR and 50 USD.

The run shows the first of four problems. The other three come from reading what the row contains:

- **The status is frozen at import time.** A dashboard that filters on status shows what the order looked like the first time the script saw it.
- **The amount is the whole order.** Nothing in the row says that part of it was cancelled or refunded.
- **The currency is the customer's.** The row carries `grand_total` and `order_currency_code`. Magento's `OrderInterface` also declares `base_grand_total` and `base_currency_code`, plus a `base_to_order_rate`. I read the pairing from the names. The interface's own comments only say "Gets the grand total for the order."
- **The day is UTC.** `created_at` is a UTC string. A store in Istanbul that sells at 22:30 UTC on 30 September made that sale on 1 October.

Here is what I changed, column by column.

| Dashboard question | Source recipe column | Report column |
| --- | --- | --- |
| How much did we sell? | `grand_total`, the whole order | `net`: `base_grand_total` minus cancelled and refunded |
| In which currency? | `grand_total`, in `order_currency_code` | `base_grand_total`, in `base_currency_code` |
| On which day? | `created_at`, UTC | store day, in the store's timezone |
| Is it still a sale? | `status` as it was at import | `state` at the last refresh, plus a `counts as sale` flag |

## One row per order, with the money already decided

The row below is what `toReportRow_` returns. I wrote it to replace the row-building part of the sync: same `increment_id` in column A, so an upsert by order number still works, but twelve columns instead of eight.

```javascript
// The money rules below are my choices, not Adobe's definition.
const NOT_A_SALE = ['canceled'];   // add 'pending_payment' if unpaid
                                   // orders should not count

function cents_(v, field, id) {
  if (v === null || v === undefined || v === '') {
    throw new Error('order ' + id + ': ' + field + ' is missing');
  }
  const n = Number(v);
  if (!isFinite(n)) {
    throw new Error('order ' + id + ': ' + field + ' = ' + v);
  }
  return Math.round(n * 100);
}

// OrderInterface declares these two as float|null. I read null as 0.
function optCents_(v, field, id) {
  return v === null || v === undefined ? 0 : cents_(v, field, id);
}

function storeDay_(createdAtUtc, tz) {
  const iso = String(createdAtUtc).replace(' ', 'T') + 'Z';
  const d = new Date(iso);
  if (isNaN(d.getTime())) {
    throw new Error('bad created_at: ' + createdAtUtc);
  }
  return Utilities.formatDate(d, tz, 'yyyy-MM-dd');
}

// One row per order. Columns: id, store day, created_at (UTC),
// status, state, base currency, gross, canceled, refunded, net,
// counts as sale, check.
function toReportRow_(o, tz) {
  const id = String(o.increment_id);
  const gross = cents_(o.base_grand_total, 'base_grand_total', id);
  const canc = optCents_(o.base_total_canceled, 'canceled', id);
  const refd = optCents_(o.base_total_refunded, 'refunded', id);
  const sale = NOT_A_SALE.indexOf(o.state) === -1;
  const net = sale ? Math.max(0, gross - canc - refd) : 0;
  return [
    id,
    storeDay_(o.created_at, tz),
    o.created_at,
    o.status,
    o.state,
    o.base_currency_code,
    gross / 100,
    canc / 100,
    refd / 100,
    net / 100,
    sale,
    canc + refd > gross ? 'REVIEW' : '',
  ];
}
```

Several decisions in there are mine, and a store owner may want different ones.

**Cents, not floats.** `cents_` turns every amount into whole cents with `Math.round`. Amounts can arrive as numbers or as strings such as `108.0000`, and `cents_` takes both. In JavaScript `0.29 * 100` is `28.999999999999996`, which a truncating conversion would turn into 28.

**Missing is an error, null is zero.** `base_grand_total` missing throws with the order number in the message, because a revenue row without a total is worse than a failed run. `base_total_canceled` and `base_total_refunded` are declared `float|null` in `OrderInterface`, so I read null as "nothing cancelled or refunded". That reading is mine; the interface does not say it.

`state`**, not** `status`**.** I understand `status` to be the label a store can rename, so I filter on `state`, and `NOT_A_SALE` starts as `['canceled']` only. If your store takes orders before payment, adding `pending_payment` is a business decision, not a code one.

**Clamp, and say so.** `net` is gross minus cancelled minus refunded, never below zero. If cancelled plus refunded is more than gross, the row gets `REVIEW` in the last column instead of a silent guess. I do not know whether those two fields can ever overlap on a real order. The flag exists so that you find out.

**No customer email in the row.** A revenue dashboard does not need it, and the tab is what the people with dashboard access can read through Looker Studio.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/7b92b383-e63c-4998-91f4-42c089b2a8ff.png align="center")

## Two checks that fail loudly

A dashboard that is wrong looks exactly like one that is right, so the tab needs a number it can be compared with. The store has one: every search response carries a total count of the matches. `SearchResultsInterface` in the Magento source declares `getTotalCount()`, and I read the JSON key as `total_count`. The function asks for the orders created up to a cutoff with `pageSize=1`. Adobe's search documentation (I read it on 22 October 2026; the page shows no date) says that without a `pageSize` "the system returns all matches", so the page size is not optional.

```javascript
const COL = { ID: 1, CREATED: 3, NET: 10, SALE: 11, N: 12 };

// Plain-text format first, or Sheets turns '000000123' into 123
// and the created_at string into a date.
function prepareSheet_(sheet, rowCount) {
  [COL.ID, COL.CREATED].forEach(function (c) {
    sheet.getRange(2, c, rowCount, 1).setNumberFormat('@');
  });
}

function apiTotalCount_(cutoffUtc) {
  const p = PropertiesService.getScriptProperties();
  const f = 'searchCriteria[filter_groups][0][filters][0]';
  const q = [
    f + '[field]=created_at',
    f + '[value]=' + encodeURIComponent(cutoffUtc),
    f + '[condition_type]=lteq',
    'searchCriteria[pageSize]=1',
  ].join('&');
  const token = p.getProperty('AC_TOKEN');
  const res = UrlFetchApp.fetch(
    p.getProperty('AC_BASE') + '/rest/V1/orders?' + q,
    {
      headers: { Authorization: 'Bearer ' + token },
      muteHttpExceptions: true,
    }
  );
  if (res.getResponseCode() !== 200) {
    throw new Error('total_count ' + res.getResponseCode());
  }
  const n = JSON.parse(res.getContentText()).total_count;
  if (typeof n !== 'number') throw new Error('no total_count');
  return n;
}

// Distinct ids, so a re-read order is not counted twice.
function reconcile_(rows, apiCount, cutoffUtc) {
  const ids = Object.create(null);
  rows.forEach(function (r) {
    if (String(r[COL.CREATED - 1]) <= cutoffUtc) {
      ids[String(r[COL.ID - 1])] = true;
    }
  });
  const have = Object.keys(ids).length;
  return { have: have, want: apiCount, ok: have === apiCount };
}
```

Three details matter. The cutoff in `runChecks` below is one hour in the past, so an order placed while the check runs does not count as missing. `reconcile_` counts distinct order numbers, because a sync that re-reads changed orders can legitimately see one twice. And `prepareSheet_` sets plain text on the order number and `created_at` before the first write. Without it, a `created_at` string can come back from `getValues()` as a `Date`, and the string comparison against the cutoff quietly stops matching. I did not test how Sheets parses either value, so the format call is a precaution, and a `Date` that slips through makes the check fail instead of pass.

## Net per day, and the check that runs

The last block adds up what the dashboard should show, in whole cents, per store day and currency, and wraps both checks in one function for a daily trigger.

```javascript
// Sheets may hand the store day back as a Date, not as text.
function dayKey_(v, sheetTz) {
  return Object.prototype.toString.call(v) === '[object Date]'
    ? Utilities.formatDate(v, sheetTz, 'yyyy-MM-dd')
    : String(v);
}

// What the dashboard should show: net per store day and currency.
function dailyNet_(rows, sheetTz) {
  const out = Object.create(null);
  rows.forEach(function (r) {
    if (r[COL.SALE - 1] !== true) return;
    const key = dayKey_(r[1], sheetTz) + ' ' + r[5];
    out[key] = (out[key] || 0) + Math.round(r[COL.NET - 1] * 100);
  });
  return out;
}

function runChecks() {
  const sheet = SpreadsheetApp.getActive().getSheetByName('Orders');
  const last = sheet.getLastRow();
  const rows = last < 2
    ? []
    : sheet.getRange(2, 1, last - 1, COL.N).getValues();
  const cutoff = Utilities.formatDate(
    new Date(Date.now() - 3600 * 1000), 'UTC', 'yyyy-MM-dd HH:mm:ss'
  );
  const res = reconcile_(rows, apiTotalCount_(cutoff), cutoff);
  const tz = SpreadsheetApp.getActive().getSpreadsheetTimeZone();
  if (!res.ok) {
    throw new Error('Orders tab has ' + res.have +
      ' orders up to ' + cutoff + ' UTC, the store has ' + res.want);
  }
  return dailyNet_(rows, tz);
}
```

If the counts differ, `runChecks` throws with both numbers in the message, so the failure email from the trigger already says how many orders are missing. If they match, it returns the totals to compare with the Looker Studio scorecard.

In Looker Studio, point the data source at the Orders tab and sum the `net` column in the store's base currency. Use `counts as sale` as a filter whenever the chart counts orders, because a cancelled order has a net of 0 but is still a row. Google's data freshness page, last updated 2026-10-20, lists 15 minutes as the default for Google Sheets and 1, 4 or 12 hours as the alternatives.

## What I tested, and what I did not

The three blocks ran in Node against fakes of `SpreadsheetApp`, `UrlFetchApp`, `PropertiesService` and `Utilities`: 57 checks on five made-up orders and their edge cases, covering null and empty amounts, the 21:00 UTC day boundary in Istanbul, partial and full refunds, a missing order and an extra one. Then I broke the code on purpose in twenty places, for example by summing `grand_total` instead of `base_grand_total`, dropping the cutoff, or skipping the text format. Nineteen failed at least one check on the first run. The twentieth, `pageSize=100` instead of `pageSize=1`, survived, because my test looked for the text `pageSize=1` and the longer one contains it. I tightened the check, and now all twenty are caught.

The amounts in the example are fixtures I made up, with an invented exchange rate between a 92 EUR order and a 100 USD base amount. They show the arithmetic, not a real store.

What a fake cannot show, and I have not measured:

- A real Adobe Commerce response. I read the field names from `OrderInterface` in the `2.4-develop` branch of the magento/magento2 repository on 22 October 2026. That file lists names and types, not what the values mean, and I did not call a live store.
- The `total_count` key. I read the name of the getter in the Magento source, not a response.
- The order states. `canceled`, `closed` and `pending_payment`, and the idea that `status` can be renamed while `state` cannot, come from my memory of Magento, not from a page I read for this post.
- Whether `base_total_refunded` includes shipping and tax.
- `Utilities.formatDate`. The fake uses Node's `Intl`, while Google's reference says it follows Java's `SimpleDateFormat`. They can disagree around daylight-saving changes.
- How Sheets and Looker Studio treat the columns once the rows are written. I never opened Looker Studio for this, and the guide's claim that it guesses `created_at` as text is not something I checked.

## Pitfalls

**A refund lands on the day of the order, not the day of the refund.** The row has one date. A refund issued in November for an October order lowers October. That is right for "what did October's orders earn" and wrong for "what did we pay out in November". If you need both, that is a second tab built from credit memos.

**Two tabs do not fix a frozen status.** The guide suggests one tab per order and one per line item when you need both shapes. That works, but each tab keeps the status it had at import unless the sync re-reads changed orders.

`REVIEW` **rows are not corrected for you.** They are counted in net at the clamped value. Look at them before you send the dashboard to anyone.

**The store day needs the store's timezone.** The sync passes an IANA name such as `Europe/Istanbul` to `toReportRow_`. Keep it in a script property next to the token, and change it in the same place if the store changes it.

## The takeaway

A revenue number on a dashboard is only as good as the column under it. The row should carry the base-currency amount, what was cancelled and refunded, the store's calendar day, and a flag for whether it still counts. A count taken from the store itself tells you when the tab has drifted. The original recipe, with the three routes and the BigQuery cut-off, is in the [Adobe Commerce to Looker Studio guide on the MageSheet blog](https://magesheet.com/blog/adobe-commerce-google-sheets-looker-studio).

Built by the MageSheet team.