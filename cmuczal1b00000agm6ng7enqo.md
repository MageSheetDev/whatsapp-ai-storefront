---
title: "Two Invoices, One Number: Allocate Invoice Numbers From the Log, Not a Cell"
seoTitle: "Google Sheets Invoice Numbers: Allocate Them From the Log"
seoDescription: "Reading the invoice number from a cell gives you duplicates and gaps. Allocate it from the log and reserve it before you send."
datePublished: 2026-09-22T17:59:14.721Z
cuid: cmuczal1b00000agm6ng7enqo
slug: google-sheets-invoice-numbering-apps-script
canonical: https://magesheet.com/blog/google-sheets-invoice-template
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/57db579c-b886-44c4-8a41-3612bdd0ce69.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/91e1522a-23d5-4593-8ceb-5d2a3e7eddfc.jpg
tags: javascript, automation, google-apps-script, googlesheets, magesheet

---

A Google Sheets invoice template does one job well: it prints. The header, the line items, the four formulas at the bottom. The automation people bolt on afterwards reads the invoice number out of cell `B4`, exports the sheet as a PDF, emails it, and appends a row to a `Log` tab:

```javascript
const invoiceNo = sheet.getRange('B4').getValue();
// ... export, email ...
ss.getSheetByName('Log').appendRow([
  invoiceNo, clientName, new Date(), dueDate, total, 'Unpaid', ''
]);
```

That works until the day someone clicks the menu item twice because the first click seemed slow. Two PDFs go out under `INV-2026-0042`, for two different clients, and the log has two rows claiming the same number. Nothing errors. You find out when a client forwards a payment reference that matches somebody else's invoice.

The cell is the problem. `B4` is a display: it shows the number the last person typed. A sequence needs a counter, and a counter needs to be the thing that decides, not the thing that shows.

## Why the cell cannot be the counter

Three separate failures come out of the same shortcut, and none of them announce themselves.

**The number is read, never claimed.** Between reading `B4` and appending the log row, the script exports a PDF and sends mail. Two runs that overlap read the same value. So do two people working from two copies of the sheet.

**The log is written last.** If the script dies after `GmailApp.sendEmail` — a timeout, a quota refusal on the row append, a browser tab closed on a slow export — the client has an invoice and your ledger does not. The row you never wrote is the row you would have chased for payment.

**Re-running is indistinguishable from a new invoice.** Nothing in the sheet records that this particular draft has already gone out, so the second click bills the client again.

There is a fourth reason that has nothing to do with code. In most VAT and sales-tax regimes, issued invoice numbers have to form an unbroken sequence you can account for, including the ones you cancelled. A cell that anybody can overtype cannot make that promise. (Check the rule for your own jurisdiction — the shape of the requirement varies, the need for an auditable sequence does not.)

## The log allocates, the cell displays

Flip the direction. The `Log` tab already holds every number you have issued, so let it hand out the next one. That is a pure function, and pure functions can be tested without a spreadsheet anywhere near them:

```javascript
var INVOICE_NO = /^([A-Z]{2,6})-(\d{4})-(\d+)$/;

function parseInvoiceNo(value) {
  var text = String(value == null ? '' : value).trim().toUpperCase();
  var m = INVOICE_NO.exec(text);
  if (!m) return null;
  return {
    prefix: m[1], year: Number(m[2]),
    seq: Number(m[3]), width: m[3].length
  };
}

function formatInvoiceNo(p) {
  var seq = String(p.seq);
  while (seq.length < p.width) seq = '0' + seq;
  return p.prefix + '-' + p.year + '-' + seq;
}

// Every number the log has ever held decides the next one. A cancelled
// invoice keeps its number: the sequence steps past it, never over it.
function nextInvoiceNo(issued, issueDate, opts) {
  var prefix = (opts && opts.prefix) || 'INV';
  var width = (opts && opts.width) || 4;
  var year = issueDate.getFullYear();
  var highest = 0;

  for (var i = 0; i < issued.length; i++) {
    var raw = issued[i];
    if (raw === '' || raw === null || raw === undefined) continue;
    var parts = parseInvoiceNo(raw);
    if (!parts) {
      throw new Error('Unreadable invoice number in the log: ' + raw);
    }
    if (parts.prefix !== prefix || parts.year !== year) continue;
    if (parts.seq > highest) highest = parts.seq;
    if (parts.width > width) width = parts.width;
  }

  return formatInvoiceNo({
    prefix: prefix, year: year, seq: highest + 1, width: width
  });
}
```

Three decisions in there are worth naming, because each one replaces a bug I would otherwise have shipped.

It compares `parts.seq`, a number, instead of taking the maximum string. `INV-2026-10000` sorts below `INV-2026-9999` in any lexical comparison, so a string maximum re-issues `10000` on your ten-thousandth invoice and every one after it. A test with those two rows fails instantly; a live sheet would not fail for years.

It filters on `parts.year`, so 1 January restarts at `0001` rather than continuing at `0043`. Sequences that run per calendar year are the common case, and the rollover is the one day of the year nobody tests by hand.

It throws on a number it cannot read rather than skipping it. A skipped row is a row whose number you are about to hand out for the second time, and `INV-26-7` typed by a hurried human is exactly the row that gets skipped. Refusing to guess is the whole point of a ledger.

## Reserve the number before you send anything

Allocation on its own is not enough: the number has to be written to the log **before** the PDF exists, under a lock, with something that identifies this draft so a second click finds the first click's work.

```javascript
var LOG = 'Log';
var COL = { no: 1, client: 2, issued: 3, due: 4, total: 5,
            status: 6, draftId: 7 };

function reserveInvoiceNo(ss, draft) {
  var lock = LockService.getDocumentLock();
  if (!lock.tryLock(30000)) {
    throw new Error('Another invoice is being issued right now.');
  }
  try {
    var log = ss.getSheetByName(LOG);
    var rows = log.getDataRange().getValues().slice(1);

    for (var i = 0; i < rows.length; i++) {
      if (rows[i][COL.draftId - 1] === draft.id) {
        return { invoiceNo: rows[i][COL.no - 1],
                 status: rows[i][COL.status - 1],
                 row: i + 2 };
      }
    }

    var issued = rows.map(function (r) { return r[COL.no - 1]; });
    var invoiceNo = nextInvoiceNo(issued, draft.issueDate);

    log.appendRow([invoiceNo, draft.clientName, draft.issueDate,
                   draft.dueDate, draft.total, 'Reserved', draft.id]);
    SpreadsheetApp.flush();

    return { invoiceNo: invoiceNo, status: 'Reserved',
             row: log.getLastRow() };
  } finally {
    lock.releaseLock();
  }
}
```

`SpreadsheetApp.flush()` before the lock is released is not decoration. Apps Script batches writes and sends them when it suits; release the lock with the append still pending and the next run reads a log that does not contain the number you just allocated. The lock is only worth what the write inside it is worth.

The `draftId` column turns the whole operation into something you can retry. The invoice sheet carries a draft id in a cell, written once when the draft is started, and it stays there until the invoice goes out. A second click finds the reserved row and stops.

| Status | What it means | Who sets it |
| --- | --- | --- |
| `Reserved` | number allocated, nothing sent yet | `reserveInvoiceNo` |
| `Unpaid` | PDF filed, email accepted, awaiting payment | `generateInvoice` |
| `Paid` | payment matched | you, or a bank-import script |
| `Void` | cancelled; the number stays, the amount goes to 0 | you |

A `Reserved` row that is still sitting there tomorrow is a send that failed halfway. That is a report you can run. The version that writes the log last has no such row, and no such report.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/bccf3017-2b61-4876-bc04-69edd3573d22.png align="center")

## The send, rewritten around the reservation

```javascript
function generateInvoice() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName('Invoice');
  var ui = SpreadsheetApp.getUi();

  var draft = readDraft(sheet);
  var res = reserveInvoiceNo(ss, draft);
  if (res.status !== 'Reserved') {
    ui.alert('This draft already went out as ' + res.invoiceNo + '.');
    return;
  }

  sheet.getRange('B4').setValue(res.invoiceNo);
  SpreadsheetApp.flush();

  var pdf = exportSheetAsPdf(ss, sheet)
    .setName('Invoice-' + res.invoiceNo + '.pdf');
  getOrCreateFolder('Invoices').createFile(pdf);

  GmailApp.sendEmail(draft.clientEmail, 'Invoice ' + res.invoiceNo,
    'Hi ' + draft.clientName + ',\n\n' +
    'Please find invoice ' + res.invoiceNo + ' attached for ' +
    draft.totalText + '. Thank you for your business.\n\n- Billing',
    { attachments: [pdf], name: 'Billing' });

  ss.getSheetByName(LOG).getRange(res.row, COL.status)
    .setValue('Unpaid');
  sheet.getRange('B10').setValue('');
  ui.alert('Invoice ' + res.invoiceNo + ' sent to ' +
           draft.clientEmail);
}
```

The `flush()` after `setValue('B4')` matters for a reason that is easy to miss: the PDF is not produced by the script. `exportSheetAsPdf` fetches `docs.google.com/.../export` over HTTP, and that request is served from the file as Google has it stored. A value still sitting in the script's pending write queue is not in that file yet, so the PDF can arrive carrying the previous invoice number while the log carries the new one. The number reaches the client through the export, not through your variable.

`readDraft` is where the second quiet bug lives:

```javascript
function readDraft(sheet) {
  var id = sheet.getRange('B10').getValue();
  if (!id) {
    id = Utilities.getUuid();
    sheet.getRange('B10').setValue(id);
  }
  return {
    id: id,
    clientName: sheet.getRange('B7').getValue(),
    clientEmail: String(sheet.getRange('B8').getValue()).trim(),
    issueDate: new Date(),
    dueDate: sheet.getRange('B6').getValue(),
    total: sheet.getRange('D33').getValue(),
    totalText: sheet.getRange('D33').getDisplayValue()
  };
}

// Unchanged from the template's own sketch: the export endpoint renders
// the stored file, and the folder lookup is created once and reused.
function exportSheetAsPdf(ss, sheet) {
  var url = 'https://docs.google.com/spreadsheets/d/' + ss.getId() +
    '/export?format=pdf&gid=' + sheet.getSheetId() +
    '&portrait=true&fitw=true&gridlines=false';
  var res = UrlFetchApp.fetch(url, {
    headers: { Authorization: 'Bearer ' + ScriptApp.getOAuthToken() }
  });
  return res.getBlob();
}

function getOrCreateFolder(name) {
  var it = DriveApp.getFoldersByName(name);
  return it.hasNext() ? it.next() : DriveApp.createFolder(name);
}
```

`getValue()` on a currency cell returns the raw number, so an email built from it reads "attached for 425.01" where the PDF says "$425.01", and sometimes "attached for 425.0100000000001". `getDisplayValue()` returns the string the cell shows, formatting and currency symbol included. The log keeps the raw number for arithmetic; the email quotes what the client is looking at.

## Pitfalls

*   **The printed column does not always add up to** `=SUM()`**.** With `=B12*C12` unrounded down the column, 7.35 hours at 42.50 displays as 312.38 and 2.65 hours displays as 112.63, while `=SUM` adds the full-precision values to 425.00 exactly. The client reads two lines that total 425.01 and a subtotal of 425.00. Round at the line, not only at the display: `=ROUND(B12*C12, 2)`, and every total built from those rounded values.
    
*   **A cancelled invoice keeps its number.** Deleting the row to "tidy up" makes the next allocation reuse that number, which is precisely the thing the sequence exists to prevent. Set the status to `Void` and the amount to zero.
    
*   `Reserved` **rows need a sweep.** A send that failed after reservation leaves a row that will never become `Unpaid`. Look for them; each one is either an invoice to resend or a number to void.
    
*   **Two prefixes, two sequences.** Credit notes are not invoices. `nextInvoiceNo(issued, date, { prefix: 'CN' })` keeps `CN-2026-0004` from colliding with `INV-2026-0042`, and the log can hold both.
    
*   **Mail quota is a hard stop, not a slowdown.** Google publishes a daily recipient limit per account for Apps Script mail — the number depends on your account type, and it is worth reading off the quota page for the plan you are actually on rather than assuming. Past it, `GmailApp.sendEmail` throws, and with the reservation written first that failure leaves a `Reserved` row instead of an unrecorded invoice.
    
*   **Bulk runs hit the six-minute wall.** Issuing a month of invoices in one loop puts an export and a send inside every iteration. That is the case for [splitting the run across triggers](https://magesheet.com/blog/apps-script-6-minute-limit); the reservation makes a resumed run safe, because a draft that already has a row is skipped rather than re-sent.
    
*   **The log is a ledger, so treat it like one.** Append rows and change statuses; do not rewrite history in place. An invoice row is a record of something that left the building, and an edited record cannot be reconciled against a bank statement.
    
*   **A Docs-template PDF changes nothing about the order.** If you fill a Google Docs template with `{{PLACEHOLDER}}` tags instead of exporting the sheet — the approach in [this Magento invoice walkthrough](https://magesheet.com/blog/magento-automated-invoice-google-docs) — the number still has to be claimed from the log before the document is built, for the same reason: the file is produced from values that must already be final.
    

## The one thing worth taking away

An invoice number is not a value on the invoice. It is a claim on a sequence, and the only thing that can grant it is the record that knows what has already been granted. Read it from a cell and you are asking a display to enforce an accounting rule; allocate it from the log, write the row before the PDF exists, and every one of the failures above turns into a row with the wrong status — visible, recoverable, and countable.

The template this started from, with the field list, the formula cheat-sheet and the point where a static sheet stops paying for itself, is on the [MageSheet blog](https://magesheet.com/blog/google-sheets-invoice-template).