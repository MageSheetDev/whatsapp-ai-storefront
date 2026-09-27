---
title: "An Empty Reminder Inbox Is Not an Empty Pipeline"
seoTitle: "Google Sheets CRM Template: Columns, Formulas, Follow-Ups"
seoDescription: "Build a free sales-pipeline CRM in Google Sheets: exact columns, copy-able formulas, and Apps Script that emails your follow-ups daily."
datePublished: 2026-09-27T14:39:33.874Z
cuid: cmujxd1wg00000agm8tgy3n3c
slug: an-empty-reminder-inbox-is-not-an-empty-pipeline
canonical: https://magesheet.com/blog/google-sheets-crm-template
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/30e30856-cf01-4390-a874-ac8863e7e367.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/39e60f10-2928-4e8e-89e0-3e6580eb45ec.jpg
tags: google-sheets, crm, automation-testing, google-apps-script, magesheet

---

A daily follow-up digest is the first automation worth adding to a spreadsheet CRM. A trigger fires at eight, a script reads the deals tab, and anything due or overdue lands in your inbox. Mine ran for weeks before I noticed it had stopped telling me anything.

Nothing had broken in a way that produces an error. No failed execution, no quota warning, no red in the trigger log. The script ran every morning, read the sheet, decided there was nothing due, and returned. An inbox with no reminder in it looks exactly the same whether the pipeline is quiet or the scan is blind.

That is the part worth engineering. Not the filter, the silence.

## Three ways the obvious filter goes quiet

The sketch version of this is short, and each line of it is a place the scan can fail without saying so.

```javascript
// The illustrative version. Every line here has a silent failure mode.
const due = rows.filter(r => {
  const stage = String(r[STAGE]);
  const followUp = r[FOLLOWUP];
  if (stage === 'Won' || stage === 'Lost') return false;
  if (!(followUp instanceof Date)) return false;
  const d = new Date(followUp);
  d.setHours(0, 0, 0, 0);
  return d <= today;
});
if (due.length === 0) return;
```

**The stage comparison is exact.** A dropdown keeps new rows clean, but rows typed before the dropdown existed, rows pasted from another sheet, and rows where someone left a trailing space all miss it. `'Won ' === 'Won'` is `false`, so a closed deal keeps getting chased. That one is at least visible, because you get reminded about a deal you already won.

`instanceof Date` **throws away anything that is not a real date cell.** Paste a column from another sheet and you get text. Type into a column that was formatted as plain text and you get text. Import a CSV and you get text. Every one of those rows is dropped from the scan with no trace. If the whole column is text, the filter drops every row, `due.length` is zero, and the script returns without sending anything. That is the silent total failure: the digest never fires again and the inbox looks like a quiet month.

**A blank follow-up date is treated as nothing to do.** It is the opposite. A deal that is open and has no next step is the one most likely to go cold, and this line guarantees it is never mentioned.

There is also a timezone seam in the same block. `new Date()` builds midnight in the script's timezone, while the dates in the sheet belong to the spreadsheet's timezone. When those differ, every comparison is off by a day, in a direction nobody notices until a deal is chased one day late.

## Read the day, not the object

The fix to most of it is one function that turns whatever is in the cell into a calendar day, in the sheet's timezone, and that distinguishes "no date" from "not a date".

```javascript
var CLOSED = ['won', 'lost'];

function normaliseStage(value) {
  return String(value == null ? '' : value)
    .replace(/\s+/g, ' ')
    .trim()
    .toLowerCase();
}

function isClosed(stageValue) {
  return CLOSED.indexOf(normaliseStage(stageValue)) !== -1;
}

/**
 * yyyy-mm-dd read in the SHEET's timezone.
 * null   = the cell is empty
 * undefined = the cell holds something that is not a date
 */
function toDateKey(value, offsetMinutes) {
  var off = offsetMinutes || 0;
  if (value === null || value === undefined) return null;
  if (typeof value === 'string' && value.trim() === '') return null;
  var ms;
  if (value instanceof Date) {
    ms = value.getTime();
  } else if (typeof value === 'number' && isFinite(value)) {
    // A raw serial from an unformatted cell: days since 1899-12-30.
    ms = (value - 25569) * 86400000;
  } else if (typeof value === 'string') {
    var t = value.trim();
    var iso = /^(\d{4})-(\d{2})-(\d{2})/.exec(t);
    if (iso) return iso[1] + '-' + iso[2] + '-' + iso[3];
    ms = Date.parse(t);
  } else {
    return undefined;
  }
  if (isNaN(ms)) return undefined;
  var shifted = new Date(ms + off * 60000);
  var y = shifted.getUTCFullYear();
  var m = ('0' + (shifted.getUTCMonth() + 1)).slice(-2);
  var d = ('0' + shifted.getUTCDate()).slice(-2);
  return y + '-' + m + '-' + d;
}
```

Comparing `'2026-09-01' <= '2026-09-27'` as strings is exact, needs no clock, and cannot drift by an hour. The offset comes from the spreadsheet rather than the script, which closes the seam rather than moving it.

## Every row lands in exactly one state

The filter above answers one question, in or out. The scan needs five answers, because three of them are things you want to hear about.

```javascript
function classifyRow(row, cols, todayKey, offsetMinutes) {
  if (isClosed(row[cols.stage])) return { state: 'closed' };
  var key = toDateKey(row[cols.followUp], offsetMinutes);
  if (key === null) return { state: 'no_date' };
  if (key === undefined) return { state: 'unreadable' };
  if (key <= todayKey) return { state: 'due', key: key };
  return { state: 'future', key: key };
}

function buildDigest(rows, cols, todayKey, offsetMinutes) {
  var out = { due: [], no_date: [], unreadable: [], future: 0, closed: 0 };
  for (var i = 0; i < rows.length; i++) {
    var r = classifyRow(rows[i], cols, todayKey, offsetMinutes);
    if (r.state === 'due') out.due.push({ row: rows[i], key: r.key });
    else if (r.state === 'no_date') out.no_date.push(rows[i]);
    else if (r.state === 'unreadable') out.unreadable.push(rows[i]);
    else if (r.state === 'future') out.future++;
    else out.closed++;
  }
  out.due.sort(function (a, b) {
    return a.key < b.key ? -1 : a.key > b.key ? 1 : 0;
  });
  out.scanned = rows.length;
  return out;
}

/** Silence is a claim. Only make it when the scan actually succeeded. */
function shouldSend(digest) {
  if (digest.due.length) return 'due';
  if (digest.unreadable.length) return 'unreadable';
  if (digest.no_date.length) return 'no_date';
  if (digest.scanned === 0) return 'empty_sheet';
  if (digest.future === 0 && digest.closed === digest.scanned) {
    return 'all_closed';
  }
  return '';
}
```

The counts are the point. A digest that says five due, two with no next step, and one date it could not read is a different message from an empty inbox, and it costs one extra pass over an array you already have in memory.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/dc53c12a-3500-4e72-a680-d3684fdc1ff6.png align="center")

## The Apps Script around it

```javascript
var SHEET_NAME = 'Deals';
var COLS = { contact: 0, email: 2, stage: 4, value: 5, followUp: 8, owner: 9 };

function sendFollowUpReminders() {
  var ss = SpreadsheetApp.getActive();
  var sheet = ss.getSheetByName(SHEET_NAME);
  if (!sheet) throw new Error('No sheet named ' + SHEET_NAME);

  var rows = sheet.getDataRange().getValues();
  rows.shift();

  // The offset that matters is the SHEET's, not the script's.
  var offset = -new Date().getTimezoneOffset();
  var todayKey = toDateKey(new Date(), offset);
  var digest = buildDigest(rows, COLS, todayKey, offset);
  var reason = shouldSend(digest);
  if (!reason) return;

  var lines = digest.due.map(function (d) {
    return '- ' + d.row[COLS.contact] + ' (' + d.row[COLS.stage] + ', '
      + d.row[COLS.value] + ') due ' + d.key
      + (d.row[COLS.owner] ? ' owner: ' + d.row[COLS.owner] : '');
  });
  if (digest.no_date.length) {
    lines.push('', digest.no_date.length + ' open deals have no next step.');
  }
  if (digest.unreadable.length) {
    lines.push('', digest.unreadable.length
      + ' rows have a follow-up cell I could not read.');
  }
  lines.push('', 'Scanned ' + digest.scanned + ' rows.');

  GmailApp.sendEmail(
    Session.getEffectiveUser().getEmail(),
    digest.due.length + ' follow-ups due (' + reason + ')',
    lines.join('\n'));
}
```

`Session.getEffectiveUser()` rather than `getActiveUser()` is deliberate. Under a time-driven trigger there is no active user in the sense the name suggests, and on some account types `getActiveUser().getEmail()` returns an empty string, which turns into a send failure at the last line of a job that otherwise worked.

The closing `Scanned N rows` line is the cheapest health check available. If that number drops to zero, or stops matching the sheet, you find out from the email rather than from a customer.

## Pitfalls

**Trusting the trigger log.** A time-driven trigger that completes successfully proves the script ran, not that it read anything. Every failure above is a successful execution. The scanned count in the message body is the signal the log cannot give you.

**Normalising the stage but not the list.** `isClosed` lowercases the cell, so the constant it compares against has to be lowercase too. Writing `CLOSED = ['Won', 'Lost']` reintroduces exactly the bug the function was added to remove, and no test that only uses clean data will catch it.

**Taking the offset from the script.** `new Date().getTimezoneOffset()` inside Apps Script gives the script's timezone. If the spreadsheet is set to a different one, read the sheet's with `SpreadsheetApp.getActive().getSpreadsheetTimeZone()` and convert, or set both to the same value and write that down somewhere. The failure is one day of drift, in the direction nobody checks.

**Sending one digest for a team.** Grouping by owner is the obvious next step, and it changes the silence rule with it: an owner with nothing due should still hear from the scan occasionally, or an owner whose rows all became unreadable gets the quiet inbox all over again, one person at a time.

Five states, one pass, and the rule that silence has to be earned. That is the difference between a reminder you trust and a reminder you stop noticing.

The template these columns and formulas come from, including the conditional-format rule that flags overdue rows before any of this is automated, is on the [MageSheet blog](https://magesheet.com/blog/google-sheets-crm-template).