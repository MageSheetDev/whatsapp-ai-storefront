---
title: "QuickBooks to Google Sheets: The Deleted Invoice Still in Your Receivables"
seoTitle: "QuickBooks to Google Sheets Sync: 3 Apps Script Fixes"
seoDescription: "Apps Script fixes for a QuickBooks to Google Sheets sync: deleted invoices in receivables, 5-minute retries, a stale CDC cursor."
datePublished: 2026-11-19T06:36:38.429Z
cuid: cmwn6g5oo000002qk04kpc8t4
slug: quickbooks-to-google-sheets-the-deleted-invoice-still-in-your-receivables
canonical: https://magesheet.com/blog/quickbooks-xero-google-sheets-sync
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/507f7577-16cf-4f17-b05f-f2c0e0548401.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/0a3702dc-a50c-412b-b480-f7c5bf8e6df6.jpg
tags: accounting, google-sheets, quickbooks, google-apps-script, magesheet

---

The [QuickBooks and Xero to Google Sheets guide on the MageSheet blog](https://magesheet.com/blog/quickbooks-xero-google-sheets-sync?utm_source=hashnode&utm_medium=social&utm_campaign=quickbooks-xero-google-sheets-sync) syncs a ledger into a spreadsheet with about 150 lines of Apps Script: an OAuth connection, a fetch wrapper that retries, an incremental sync through QuickBooks' Change Data Capture (CDC) endpoint, and an upsert that writes each invoice to its own row. I ran that code in Node against a mock of Apps Script and a fake QuickBooks that answers the way Intuit's reference says it does, and it did five things the text does not promise.

The first one is the one an accountant finds. Ten invoices sync, five of them open, and the Balance column adds up to 2,750. Someone deletes invoice 1007 in QuickBooks, which had 570 open. The next run marks the row DELETED, exactly as designed, and the Balance column still adds up to 2,750. The real open balance is 2,180.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/bb77b119-8d25-4067-aa7b-816f1e8bc7ef.gif align="center")

Below are the five results side by side, then three replacement blocks. The invoices, customers and amounts are invented, and the services are mocked, so this shows what the logic does, not what Intuit's or Google's servers do. The section near the end lists what I could not test.

| Input (invented data) | Original code | With the three blocks |
| --- | --- | --- |
| 10 invoices, 5 open (2,750), then #1007 with 570 open is deleted | Row marked DELETED, Balance column still adds up to 2,750 | Balance of #1007 set to 0, column adds up to 2,180, total of 570 kept |
| First run: 1 invoice and 3 deletions in the 29-day window | 4 rows, 3 of them an Id, blank cells and DELETED | 1 row, 3 deletions counted as skipped |
| QuickBooks answers 429 to every request | 5 requests, 300 s asleep in one call, then an error; Invoices + Payments: 600 s | 2 requests, 60 s, then an error that says so; Invoices + Payments: 60 s in total |
| Stored cursor is 45 days old | Sent to QuickBooks unchanged | No request, one email, cursor left in place |
| 1,000 changes waiting, one run every 15 minutes for a day | 96 failed runs, cursor never set | 0 failed runs, one email, nothing written |

## Why the original does this

The guide gets the hard parts right: the cursor is the time the run started, saved only after a successful write, and rows are upserted by ID, so a failed run re-reads the same window and does no harm. The five results come from three smaller assumptions around that core.

1. **A deleted invoice is only a status change.** Intuit's reference says deleted objects come back in the CDC response with little more than `"status": "Deleted"`, an `Id` and `MetaData`. The original maps that to a row of `null` values, and `null` means "keep the existing cell". That keeps the number and the customer for the audit trail, which is the point, but it also keeps the Balance. Any `SUM` over that column, any pivot table and any dashboard built on the tab now counts money that nobody owes. When a deletion arrives for an invoice the sheet never had, for example one created and deleted inside the first 29-day window, the same mapping appends a row with an ID, blank cells and DELETED.
2. **Retries assume the run has time to wait.** `qboGet` sleeps 60 seconds on every 429, up to five attempts. That follows Intuit's advice for one request, but an Apps Script execution stops at 6 minutes. One call that keeps getting 429 sleeps 300 seconds and then throws anyway. A trigger that syncs Invoices and then Payments sleeps 600 seconds in theory and is killed by Google before the second call is done.
3. **The cursor is trusted as it is.** The guide's text says to run the backfill again if the sync was off for more than a month, because CDC looks back at most 30 days. The code does not check: it sends whatever timestamp is stored. And when CDC returns 1,000 changes, the code throws, which is the right reaction once, but the cursor never moves, so every following run throws the same error. At one run every 15 minutes, that is 96 failed executions a day.

None of this needs a bad actor. It needs a bookkeeper deleting a duplicate invoice, a slow afternoon at Intuit, and a holiday.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/a6665eca-2b47-4413-a465-c7d46598beef.png align="center")

## Block 1: a deleted invoice keeps its history, not its balance

`invoiceRow_` maps one CDC object to the nine columns of the Invoices tab: Id, number, date, customer, total, balance, due date, status, last updated. For a deletion it still writes `null` into the number, date, customer, total and due date, so those cells keep what they had, but it writes `0` into Balance. A deleted invoice is owed by nobody, and the TotalAmt column still shows what it was for.

`upsertById` is the original function with two changes. It takes an optional `skipIfNew` test, and `syncInvoices` passes `isDeletedRow_`, so a deletion for an ID the sheet has never seen is counted and dropped instead of written as an empty row. It also returns how many rows it added, updated and skipped, which is what you want in a log line.

```javascript
const STATUS_COL = 7;   // 0-based: Id, Doc, Date, Customer, Total,
                        // Balance, Due, Status, Updated

function invoiceRow_(inv) {
  const updated = inv.MetaData ? inv.MetaData.LastUpdatedTime : '';
  if (inv.status === 'Deleted') {
    // null = keep the existing cell (number, customer, total stay
    // for the audit trail). Balance is forced to 0: nobody owes a
    // deleted invoice, so it must drop out of SUM(Balance).
    return [inv.Id, null, null, null, null, 0, null,
      'DELETED', updated];
  }
  return [
    inv.Id,
    inv.DocNumber || '',
    inv.TxnDate || '',
    inv.CustomerRef ? inv.CustomerRef.name : '',
    inv.TotalAmt || 0,
    inv.Balance || 0,
    inv.DueDate || '',
    'ACTIVE',
    updated
  ];
}

function isDeletedRow_(row) {
  return row[STATUS_COL] === 'DELETED';
}

function upsertById(sheetName, rows, skipIfNew) {
  const stats = { added: 0, updated: 0, skipped: 0 };
  if (!rows.length) return stats;
  const sh = SpreadsheetApp.getActive().getSheetByName(sheetName);
  const width = rows[0].length;
  const lastRow = sh.getLastRow();
  const existing = lastRow > 1
    ? sh.getRange(2, 1, lastRow - 1, width).getValues()
    : [];
  const indexById = new Map(
    existing.map((r, i) => [String(r[0]), i]));

  rows.forEach(row => {
    const id = String(row[0]);
    const i = indexById.get(id);
    if (i === undefined) {
      // A deletion for a row we never wrote: nothing to flag.
      if (skipIfNew && skipIfNew(row)) { stats.skipped++; return; }
      indexById.set(id, existing.length);
      existing.push(row.map(v => v === null ? '' : v));
      stats.added++;
    } else {
      existing[i] = row.map((v, c) => v === null ? existing[i][c] : v);
      stats.updated++;
    }
  });
  if (existing.length) {
    sh.getRange(2, 1, existing.length, width).setValues(existing);
  }
  return stats;
}
```

With the same ten invoices and the same deletion, the Balance column now adds up to 2,180, and the TotalAmt column still adds up to the full 5,550, because the row for 1007 keeps its 570 total. The first-run case writes one row instead of four.

If you would rather keep the original balance visible, the other honest option is to leave the cell alone and build every receivables figure with `SUMIFS(F:F, H:H, "ACTIVE")`. That works as long as everyone who adds a chart later knows about it. Zeroing the balance makes the plain `SUM` correct, which is the formula people actually type.

## Block 2: one wait budget for the whole run

`retryWait_` decides how long to wait before the next attempt, and whether to wait at all. A 429 still waits 60 seconds, as Intuit asks. A 5xx waits 1, 2, 4, 8 and 16 seconds. Anything else, including a second 401, a 400 for a retired minor version or a bad query, is not retried. The new rule is the budget: all sleeps in one execution together may not pass 90 seconds. `waitedMs_` is a global, and Apps Script starts every execution with fresh globals, so the budget is shared by every `qboGet` call in that run and resets on the next trigger.

```javascript
const QBO_BASE = 'https://quickbooks.api.intuit.com/v3/company/';
const QBO_MINOR_VERSION = 75;
const WAIT_BUDGET_MS = 90 * 1000;  // all sleeps in one execution
let waitedMs_ = 0;                 // globals reset every execution

function retryWait_(code, attempt, waitedMs) {
  let wait = -1;
  if (code === 429) wait = 60000;              // Intuit: wait 60s
  if (code >= 500) wait = 1000 * Math.pow(2, attempt);
  if (wait < 0) return -1;                     // not retryable
  if (waitedMs + wait > WAIT_BUDGET_MS) return -1;  // next trigger
  return wait;
}

function qboGet(path, params) {
  const service = getQboService();
  if (!service.hasAccess()) {
    throw new Error('QuickBooks is not connected.');
  }
  const realm = PropertiesService.getUserProperties()
    .getProperty('QBO_REALM_ID');
  const all = Object.assign(
    { minorversion: QBO_MINOR_VERSION }, params);
  const query = Object.entries(all)
    .map(([k, v]) => k + '=' + encodeURIComponent(v)).join('&');
  const url = QBO_BASE + realm + path + '?' + query;

  for (let attempt = 0; attempt < 5; attempt++) {
    const res = UrlFetchApp.fetch(url, {
      headers: { Authorization: 'Bearer ' + service.getAccessToken(),
                 Accept: 'application/json' },
      muteHttpExceptions: true
    });
    const code = res.getResponseCode();
    if (code === 200) return JSON.parse(res.getContentText());
    if (code === 401 && attempt === 0) { service.refresh(); continue; }
    const wait = retryWait_(code, attempt, waitedMs_);
    if (wait < 0) {
      throw new Error('QuickBooks ' + code + ' on ' + path +
        ' (waited ' + Math.round(waitedMs_ / 1000) + 's): ' +
        res.getContentText().slice(0, 300));
    }
    Utilities.sleep(wait);
    waitedMs_ += wait;
  }
  throw new Error('QuickBooks: gave up after 5 attempts on ' + path);
}
```

With Intuit answering 429 to everything, one call now makes two requests, sleeps 60 seconds and throws an error that says how long it waited. A trigger that tries Invoices and then Payments sleeps 60 seconds in total, not 600, because the second call finds the budget spent and gives up at once. Five 503s in a row still get their full 1 + 2 + 4 + 8 + 16 = 31 seconds.

Giving up sooner is safe here because of the guide's own cursor design. Nothing is written and the cursor is not saved when `qboGet` throws, so the next trigger, 15 minutes later, reads the same window. Waiting five minutes inside one execution buys nothing that the next run does not get for free. The 90 seconds is my choice, not a number from Intuit or Google. It leaves room for a backfill page or a report call in the same run.

## Block 3: check the cursor before you use it

`cursorProblem_` looks at the stored cursor before any request goes out. An empty cursor is fine: the first run reads the last 29 days, as in the original. A cursor that cannot be parsed, one that is more than five minutes in the future, or one older than 29 days is a problem. Intuit allows 30 days. The spare day keeps a run that starts late, or a request that is retried, inside the window.

`alertOnce_` mails the script owner once per problem. It stores the message in User Properties and does not send the same text twice, which is why the messages are built from values that do not change from run to run: the stored cursor, not the number of days, and "1000+ changes", not the exact count. A message with the day count in it would arrive every morning. When a run succeeds, `syncInvoices` deletes the stored alert, so the next problem is reported again.

```javascript
const CDC_MAX = 1000;          // per CDC request (Intuit reference)
const MAX_LOOKBACK_DAYS = 29;  // CDC allows 30; keep a day spare

function cursorProblem_(since, nowMs) {
  if (!since) return '';                  // first run: 29-day window
  const t = Date.parse(since);
  if (isNaN(t)) return 'unreadable cursor "' + since + '"';
  if (t > nowMs + 5 * 60e3) return 'cursor is in the future';
  if (nowMs - t > MAX_LOOKBACK_DAYS * 864e5) {
    // Same text every run, so alertOnce_ mails once, not daily.
    return 'cursor ' + since + ' is past the CDC window';
  }
  return '';
}

function alertOnce_(key, message) {
  const props = PropertiesService.getUserProperties();
  if (props.getProperty(key) === message) return false;  // sent
  MailApp.sendEmail({
    to: Session.getEffectiveUser().getEmail(),
    subject: 'QuickBooks sync paused: ' + key,
    body: message + '\nRun the paged backfill, then the sync ' +
      'resumes on its own.'
  });
  props.setProperty(key, message);
  return true;
}

function syncInvoices() {
  const props = PropertiesService.getUserProperties();
  const now = Date.now();
  const saved = props.getProperty('QBO_INVOICES_SINCE');
  const problem = cursorProblem_(saved, now);
  if (problem) return alertOnce_('QBO_INVOICES_ALERT', problem);

  const since = saved ||
    new Date(now - MAX_LOOKBACK_DAYS * 864e5).toISOString();
  const runStartedAt = new Date(now).toISOString();  // before call
  const data = qboGet('/cdc',
    { entities: 'Invoice', changedSince: since });
  const pages = data.CDCResponse[0].QueryResponse || [];
  const invoices = pages.flatMap(p => p.Invoice || []);
  if (invoices.length >= CDC_MAX) {
    return alertOnce_('QBO_INVOICES_ALERT',
      CDC_MAX + '+ changes since ' + (saved || 'the first run') +
      ' (one CDC request returns at most ' + CDC_MAX + ')');
  }

  upsertById('Invoices', invoices.map(invoiceRow_), isDeletedRow_);
  props.setProperty('QBO_INVOICES_SINCE', runStartedAt);
  props.deleteProperty('QBO_INVOICES_ALERT');  // healthy again
  return true;
}
```

With a cursor 45 days old, the original sends it to Intuit unchanged. The new version makes no request, sends one email and leaves the cursor where it is, and 96 runs in a day still send one email. With 1,000 changes waiting, the original fails 96 times a day and the cursor is never set. The new version sends one email, fails zero times, writes nothing and waits for the backfill.

The guide describes the backfill: a paged `SELECT * FROM Invoice STARTPOSITION n MAXRESULTS 1000`, continued on a follow-up trigger when the history is long. When it finishes, it should set `QBO_INVOICES_SINCE` to the time the backfill started, and the next scheduled run picks up from there.

## Pitfalls

- **Intuit's 1,000 is per request, not per entity.** The reference says "A given CDC request returns a maximum of 1000 objects". The original's comment says each entity is capped at 1,000. If you ask for `Invoice,Payment` in one call, the limit is shared, so check the total. The blocks above ask for one entity per call, which keeps the check simple.
- **Zero is a valid balance, so is a zero total.** A voided invoice in QuickBooks stays active with a total of 0. `invoiceRow_` uses `inv.TotalAmt || 0`, which keeps that 0. Do not "fix" it to skip falsy values, or voids disappear.
- **The status column is a position.** `STATUS_COL = 7` points at the eighth value that `invoiceRow_` returns. If you add or reorder columns there, change the constant too, or `isDeletedRow_` reads the wrong field and deletions are written as empty rows again.
- **Do not log the response body somewhere shared.** The error message includes up to 300 characters of Intuit's response. For a 400 that is a fault description. For other endpoints it can be customer data. Keep the execution log as private as the ledger.
- **One alert key per entity.** `alertOnce_` dedupes by key. If you add Payments with the same `QBO_INVOICES_ALERT` key, a payments alert can hide an invoices alert. Give each its own key.

## How this goes stale

- **A holiday.** The trigger is paused for five weeks and the cursor passes the 30-day window. The original keeps sending the old date. Block 3 stops and asks for a backfill on the first run after you come back.
- **Month-end batches.** A bookkeeper imports 1,200 invoices on the last day of the month. The first sync after that crosses the 1,000 limit. With the original, every run fails until someone reads the error. With Block 3, one email explains it.
- **A retired minor version.** The guide pins minor version 75 because Intuit retired 1 to 74 in August 2025. When 75 is retired too, the answer is a 4xx. Block 2 does not retry it, so the error reaches the log in seconds instead of after five attempts.
- **The refresh token's maximum lifetime.** Intuit caps it, and a person has to click Connect again. None of these blocks changes that. Wire the sync into an alert, as the guide says, so the day it stops is the day someone hears about it.

## What I tested, and what I did not

Measured, in Node with mocked services:

- 83 tests, all passing, also under `TZ=UTC`, `TZ=Asia/Istanbul` and `TZ=America/Los_Angeles`. Six of them pin the original code's behaviour, the five results in the table above. The other 77 test the three blocks.
- 56 deliberate breakages of the new code, for example keeping the deleted balance, dropping the skip test, letting the budget stop accumulating or building the alert text from the day count. All 56 made at least one test fail, and they failed in 39 different tests, so the breakages are not all caught by one assertion.
- The code blocks in this post were extracted back out of the article and run against the same 83 tests.

Not measured:

- A live QuickBooks company or Intuit's sandbox. The fake QuickBooks returns the CDC shape from Intuit's reference page, including the deleted-object sample. I have not seen what Intuit returns for a `changedSince` older than 30 days. The reference says only that it must be "within 30 days of today".
- The OAuth layer. `getQboService`, the token refresh and the lock from the guide are replaced by a stub that always has access. The refresh race the guide describes was not simulated.
- Google Sheets itself. The mock stores values as given. I have not measured how Sheets converts a document number like `00123` or a date string on `setValues`.
- Whether receiving a payment changes the invoice's `LastUpdatedTime` in CDC. The blocks assume a changed balance arrives as a changed invoice, which is how the guide is written. I did not confirm it against a live company.
- The Xero half of the guide. Xero has its own limits (60 calls a minute per organisation) and its own incremental header. The budget idea in Block 2 carries over, but I did not test it against Xero.
- Intuit's `Retry-After` header. Block 2 waits the fixed 60 seconds the guide uses.

## The takeaway

A sync that marks deletions is only half done until the columns people add up agree with the ledger. Zero the balance of a deleted invoice, give the whole run one wait budget instead of five minutes per call, and check the cursor before trusting it, so that a holiday or a month-end batch ends in one email rather than 96 failures a day. The connection, the backfill, the P&L flattener and the Xero differences are in the original [QuickBooks and Xero sync guide on the MageSheet blog](https://magesheet.com/blog/quickbooks-xero-google-sheets-sync?utm_source=hashnode&utm_medium=social&utm_campaign=quickbooks-xero-google-sheets-sync). What the alert email should do once it is sent is covered in [the error-handling post](https://dev.to/hayrullahkar/apps-script-error-handling-the-failed-invoice-your-ledger-skips-forever-3bpe).

---

*Drafted with AI help; the code was run and tested, and every number comes from those tests.*