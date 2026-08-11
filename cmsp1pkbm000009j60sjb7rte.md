---
title: "Magento Order Sync in Apps Script: Page-Based Pagination Drops Orders"
seoTitle: "Magento Order Sync: Fix Skipped Rows in Apps Script"
seoDescription: "Page-based searchCriteria pagination duplicates and drops Magento orders mid-sync. Keyset cursors, an overlap window, and tested Apps Script code."
datePublished: 2026-08-11T19:20:42.300Z
cuid: cmsp1pkbm000009j60sjb7rte
slug: magento-order-sync-keyset-pagination
canonical: https://magesheet.com/blog/magento-google-sheets-automation
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/67c38a8d-2d9c-4ab3-8f6e-e600d5e91ac1.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/29a43b4b-2eb9-4f64-b578-14b761609e80.jpg
tags: ecommerce, magento, google-apps-script, googlesheets, magesheet

---

Point a `searchCriteria[currentPage]` loop at a Magento store that is still taking orders, and the Sheet ends up disagreeing with the store. Some orders land twice, some never land at all. Nothing throws. The API returns 200 on every call, the script logs a clean finish, and the only way to find out is to count rows against the admin.

I hit this building an order sync, and the fix was not a retry or a bigger page size. It was giving up on page numbers entirely.

## The offset is a row count, not a row

`currentPage` is an offset. `currentPage=2` with `pageSize=100` means "skip 100 rows, then take 100". That is only correct if the result set holds still while you walk it, and on a live store it does not.

Work through the arithmetic. Sort newest-first, request page 1, and read rows 1 to 100. Two orders are placed before your page-2 request. Every existing row is now two positions further down the list. The two rows you already read at the bottom of page 1 sit at positions 101 and 102, which is exactly where page 2 starts. **You read them again.** That is the duplicate.

The hole works the other way. Sort oldest-first on `updated_at` and run the same loop while statuses are changing. A row you already passed gets updated, jumps to the end of the sort, and vacates its slot. Everything behind it shifts one position toward the start, so the row that was at position 201 is now at 200 — inside the page you just finished. **You never see it.**

Both failures need the result set to move under you, which is the normal state of a production store. Neither raises an error, because from the API's side every request was valid.

## Keyset pagination: make the position a value

Stop asking for an offset. Ask for "the next N orders after the last `entity_id` I actually wrote". Concurrent inserts cannot move that position, because it is a value in the data rather than a count of rows.

Here is the foundation: one fetch helper that checks the response code before parsing, and one query builder.

```javascript
function magentoGet_(path) {
  const p = PropertiesService.getScriptProperties();
  const base = p.getProperty('MAGENTO_BASE_URL');
  const token = p.getProperty('MAGENTO_TOKEN');
  const res = UrlFetchApp.fetch(base + '/rest/V1/' + path, {
    method: 'get',
    headers: { Authorization: 'Bearer ' + token },
    muteHttpExceptions: true,
  });
  const code = res.getResponseCode();
  if (code === 429 || code >= 500) {
    throw new Error('retryable ' + code + ': ' + res.getContentText());
  }
  if (code >= 300) {
    throw new Error('magento ' + code + ': ' + res.getContentText());
  }
  return JSON.parse(res.getContentText());
}

function magentoQuery_(groups, opts) {
  const p = [];
  groups.forEach(function (filters, g) {
    filters.forEach(function (f, i) {
      const k = 'searchCriteria[filter_groups][' + g +
                '][filters][' + i + ']';
      p.push(k + '[field]=' + encodeURIComponent(f.field));
      p.push(k + '[value]=' + encodeURIComponent(f.value));
      p.push(k + '[condition_type]=' + encodeURIComponent(f.cond));
    });
  });
  if (opts.sortField) {
    p.push('searchCriteria[sortOrders][0][field]=' + opts.sortField);
    p.push('searchCriteria[sortOrders][0][direction]=' + opts.sortDir);
  }
  p.push('searchCriteria[pageSize]=' + opts.pageSize);
  p.push('searchCriteria[currentPage]=1');
  return p.join('&');
}
```

`currentPage` is pinned to 1 and never changes. That is the whole trick: every request asks for the first page of a *different* filter, not the next page of the same one.

The sync loop moves the cursor instead.

```javascript
const PAGE_SIZE = 100;
const RUN_BUDGET_MS = 4.5 * 60 * 1000;

// A billing name starting with = + - or @ becomes a live formula
// the moment setValues writes it. Prefix it instead.
function safeText_(v) {
  const s = v == null ? '' : String(v);
  return /^[=+\-@]/.test(s) ? "'" + s : s;
}

function orderToRow_(o) {
  return [
    safeText_(o.increment_id),
    o.created_at,
    o.updated_at,
    o.status,
    safeText_(o.customer_email),
    Number(o.grand_total),
    o.order_currency_code,
  ];
}

function nextCursor_(items, current) {
  return items.reduce(function (max, o) {
    const id = Number(o.entity_id);
    return id > max ? id : max;
  }, current);
}

function syncNewOrders() {
  const props = PropertiesService.getScriptProperties();
  const sheet = SpreadsheetApp.getActive().getSheetByName('Orders');
  const started = Date.now();
  let cursor = Number(props.getProperty('ORDER_CURSOR') || 0);

  while (Date.now() - started < RUN_BUDGET_MS) {
    const query = magentoQuery_(
      [[{ field: 'entity_id', value: cursor, cond: 'gt' }]],
      { sortField: 'entity_id', sortDir: 'ASC', pageSize: PAGE_SIZE }
    );
    const items = magentoGet_('orders?' + query).items || [];
    if (!items.length) return;

    const rows = items.map(orderToRow_);
    sheet.getRange(sheet.getLastRow() + 1, 1, rows.length, rows[0].length)
      .setValues(rows);
    SpreadsheetApp.flush();

    cursor = nextCursor_(items, cursor);
    props.setProperty('ORDER_CURSOR', String(cursor));
    if (items.length < PAGE_SIZE) return;
  }
}
```

Two details in that loop carry the reliability.

**The write happens before the cursor moves.** `setValues`, then `flush`, then `setProperty`. If the run dies between them — quota, timeout, a bad gateway from the store — the next run re-reads the batch it already wrote. Re-reading one page is a bounded, fixable problem. Advancing the cursor first and then failing to write is a permanent hole.

**The loop watches the clock.** `RUN_BUDGET_MS` leaves headroom under the 6-minute execution ceiling so the run exits on its own terms with a saved cursor, rather than being killed mid-`setValues`.

## New orders are only half of it

A cursor on `entity_id` finds orders that did not exist before. It will never find an order that was placed yesterday and shipped today, because updating an order does not change its `entity_id`.

The second pass filters on `updated_at` and upserts. It needs two things the naive version gets wrong.

```javascript
function parseMagentoUtc_(s) {
  return new Date(String(s).replace(' ', 'T') + 'Z');
}

function toMagentoUtc_(d) {
  return d.toISOString().slice(0, 19).replace('T', ' ');
}

// Re-read a safety overlap so commits that landed late are not missed.
function windowStart_(lastRunUtc, overlapSec) {
  const t = parseMagentoUtc_(lastRunUtc).getTime() - overlapSec * 1000;
  return toMagentoUtc_(new Date(t));
}

function planUpsert_(index, rows, idCol) {
  const updates = [];
  const appends = [];
  const seen = Object.create(null);
  rows.forEach(function (row) {
    const id = String(row[idCol]);
    if (Object.prototype.hasOwnProperty.call(seen, id)) {
      updates.push({ row: seen[id], values: row });
      return;
    }
    if (Object.prototype.hasOwnProperty.call(index, id)) {
      seen[id] = index[id];
      updates.push({ row: index[id], values: row });
    } else {
      appends.push(row);
    }
  });
  return { updates: updates, appends: appends };
}

// increment_id -> sheet row number. Column A must be formatted as
// plain text, or Sheets hands back 123 instead of '000000123'.
function readIdIndex_(sheet) {
  const last = sheet.getLastRow();
  if (last < 2) return {};
  const ids = sheet.getRange(2, 1, last - 1, 1).getValues();
  const index = Object.create(null);
  ids.forEach(function (r, i) {
    if (r[0] !== '') index[String(r[0])] = i + 2;
  });
  return index;
}

function syncChangedOrders() {
  const props = PropertiesService.getScriptProperties();
  const sheet = SpreadsheetApp.getActive().getSheetByName('Orders');
  const lastRun = props.getProperty('ORDER_WATERMARK') ||
    toMagentoUtc_(new Date(Date.now() - 864e5));
  const since = windowStart_(lastRun, 120);
  const runStartedUtc = toMagentoUtc_(new Date());

  const query = magentoQuery_(
    [[{ field: 'updated_at', value: since, cond: 'gteq' }]],
    { sortField: 'updated_at', sortDir: 'ASC', pageSize: PAGE_SIZE }
  );
  const items = magentoGet_('orders?' + query).items || [];
  const plan = planUpsert_(readIdIndex_(sheet), items.map(orderToRow_), 0);

  plan.updates.forEach(function (u) {
    sheet.getRange(u.row, 1, 1, u.values.length).setValues([u.values]);
  });
  if (plan.appends.length) {
    sheet.getRange(sheet.getLastRow() + 1, 1, plan.appends.length,
      plan.appends[0].length).setValues(plan.appends);
  }
  SpreadsheetApp.flush();
  props.setProperty('ORDER_WATERMARK', runStartedUtc);
}
```

**The window starts before the last run, not at it.** A transaction can be stamped with an `updated_at` of 14:03:22 and commit a second or two later. If your next run asks for `updated_at >= 14:03:23`, that row is already invisible. The 120-second overlap re-reads a small band on every run, and the upsert makes re-reading free.

**The watermark is the run's start time, not its finish time.** Save the finish time and every order that changed while the run was in flight falls into the gap between the two runs and is never picked up.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/48c92888-60b8-4111-a208-60b6ba7fcd85.jpg align="center")

## Pitfalls

**Leaving out** `pageSize` **does not give you a default page.** Adobe's REST documentation is explicit: "If the `pageSize` is not specified, the system returns all matches." Not the first 20 — all of them. On a store with tens of thousands of orders that is a single response Apps Script has to hold in memory and `JSON.parse` in one go. Always set it.

**Two filters in one** `filter_groups` **is an OR, not an AND.** The same docs: "To perform a logical OR, specify multiple `filters` within a `filter_groups`. To perform a logical AND, specify multiple `filter_groups`." Put `status=processing` and `updated_at gteq X` side by side in group 0 and you get every processing order ever placed, plus everything updated since X. It looks like a working sync until you check the row count.

**Magento date fields are UTC; the admin grid shows store time.** Copy a timestamp off the admin screen into a filter and the window shifts by your UTC offset. For a store on `+03:00` that is a three-hour hole on every run, in the direction that loses orders rather than duplicating them. Build the string from a `Date` with `toISOString`, as `toMagentoUtc_` does above, and never from a formatted local string.

`increment_id` **is a string with leading zeros.** `000000123` is not `123`. If you build the upsert index by reading a Sheets column that was never formatted as text, Sheets hands you the number `123`, no row matches, and every changed order appends a fresh duplicate. Format the column as plain text and compare with `String()` on both sides.

**A customer name can be a formula.** A billing name that starts with `=`, `+`, `-` or `@` becomes a live formula the moment `setValues` writes it, and `=HYPERLINK(...)` in a name field is a real thing storefronts receive. That is what `safeText_` in the sync loop above is for: it prefixes an apostrophe so the cell stays text. Run every customer-supplied field through it, not just the name.

**Rate limits are a separate problem.** A sync that behaves against 200 orders will meet `429` and transient `5xx` at 5,000. `magentoGet_` above marks those as retryable but does not retry; the backoff, jitter and dead-letter handling belong in the fetch layer, which I covered separately in [the hidden cost of UrlFetchApp](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries).

## What this buys you

I pulled the pure logic out of the Apps Script services — the query builder, the cursor, the UTC conversions, the upsert planner — and ran it in Node: 37 assertions, covering the OR-versus-AND grouping, a cursor that refuses to move backwards, an overlap window that crosses midnight, and an `increment_id` of `000000123` correctly refusing to match `123`.

The rule that survives every store size is the one at the top: **never let your read position be a row count.** Page numbers describe where you are in a list that Magento is free to reorder between two HTTP calls. An `entity_id` describes a row.

The production version — two passes on one trigger, retry and backoff in the fetch layer, and the full Magento-to-Sheets playbook it sits inside — is on the [MageSheet blog](https://magesheet.com/blog/magento-google-sheets-automation).

Built by the MageSheet team.