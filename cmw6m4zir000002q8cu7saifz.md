---
title: "Outgrown Google Sheets? Count the Slow Formulas Before You Decide"
seoTitle: "Outgrown Google Sheets? Audit Its Formulas With Apps Script"
seoDescription: "An Apps Script audit that counts volatile functions, open ranges and IMPORTRANGE per tab, plus three fixes to a locked recordSale()."
datePublished: 2026-11-07T16:23:46.083Z
cuid: cmw6m4zir000002q8cu7saifz
slug: outgrown-google-sheets-formula-audit-apps-script
canonical: https://magesheet.com/blog/outgrown-google-sheets-signs
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/a46494d4-3c7b-483f-9bb1-e6668ed88b70.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/26a3322c-623a-405a-bb6e-f8cd1ac025f0.jpg
tags: javascript, automation, google-sheets, google-apps-script, magesheet

---

Our guide to [outgrowing Google Sheets](https://magesheet.com/blog/outgrown-google-sheets-signs) lists six signs that a business needs a real system. Sign 3 is the one people reach for first: the file is slow, scripts time out, a filter takes eight seconds. It is also the easiest one to get wrong, because "the sheet feels slow" is an opinion, and an opinion is a weak reason to start a custom software project.

The guide's own advice is to fix the sheet first, because the soft limits arrive long before the hard ones. So before anyone books a scoping call, I wanted a number. I wrote one Apps Script function that reads every formula in the file, counts the patterns that make a sheet slow, and writes the result to a tab called Sheet health. While testing it, I also ran the guide's Sign 1 code (the locked `recordSale()` function) against a fake sheet, and three ordinary inputs got through that should not have.\
\\

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/e0f6debd-7e21-40e3-baa6-4bdcfc9d61b6.gif align="center")

## Why "it feels slow" is not enough

There are two ways a sheet hits a wall, and they need different answers.

The hard limits are published. Google's Drive limits page puts a spreadsheet at up to 20 million cells. One Apps Script run stops at 6 minutes ([our 6-minute limit guide](https://magesheet.com/blog/apps-script-6-minute-limit) covers the workarounds). Those numbers almost never decide anything for a small business.

The soft limits are the formulas. Google's own help page on Sheets performance names three patterns:

- **Volatile functions.** In Google's words, "TODAY(), RAND(), RANDBETWEEN(), and NOW() are volatile functions". Google says they refresh frequently to stay up to date, so every cell that uses one keeps recalculating.
- **Open ranges.** Google's example is `SUM(A:A)`, which makes Sheets read every row in column A, against `SUM(A1:A10)`, which reads ten.
- **IMPORTRANGE.** Each one fetches data from another file, so the page describes it as a round trip.

Our guide also lists `INDIRECT()` as volatile. Google's page does not. I count `INDIRECT` and `OFFSET` in their own column, so the report does not claim more than Google says.

None of this tells you whether a sheet has outgrown its shape. It tells you whether a slow sheet is slow because of how it is built. That is the question you need to answer before Sign 3 can count.

## The scanner: one formula at a time

The first block is plain JavaScript with no Google services, so all of it can be tested outside Sheets. `scanFormula()` takes one formula string and returns five counts.

```javascript
// Names Google lists as volatile on its performance help page.
const VOLATILE = ['NOW', 'TODAY', 'RAND', 'RANDBETWEEN'];
// Not on Google's list, but they re-read whatever they point at.
const INDIRECT_LIKE = ['INDIRECT', 'OFFSET'];

// Blank out "string literals" and 'quoted sheet names' so a
// formula like ="Call at 10:30" or 'A:A notes'!B2 is not misread.
function stripLiterals(f) {
  return String(f)
    .replace(/"(?:[^"]|"")*"/g, '""')
    .replace(/'(?:[^']|'')*'!/g, "S!");
}

function countCalls(f, names) {
  const re = new RegExp(
    '(?<![A-Z0-9_.])(' + names.join('|') + ')\\s*\\(', 'gi');
  return (f.match(re) || []).length;
}

// A:A, $B:$D, A2:A, Sales!C5:C, i.e. ranges with no end row.
const OPEN_RANGE = new RegExp(
  '\\$?[A-Z]{1,3}(\\$?\\d+)?:\\$?[A-Z]{1,3}' +
  '(?![A-Z0-9_$(])', 'gi');

function scanFormula(formula) {
  if (!formula) return null;
  const f = stripLiterals(formula);
  return {
    volatile: countCalls(f, VOLATILE),
    indirect: countCalls(f, INDIRECT_LIKE),
    importrange: countCalls(f, ['IMPORTRANGE']),
    arrayformula: countCalls(f, ['ARRAYFORMULA']),
    openRanges: (f.match(OPEN_RANGE) || []).length,
  };
}
```

The part that took the most tests is `stripLiterals()`. Without it, `="Call NOW() at 10:30"` counts as a volatile call and an open range, and a tab named `'A:A notes'` counts as a whole-column reference. With it, text in double quotes and sheet names in single quotes are blanked out before any counting starts.

Here is what the scanner counts and what it does not:

| Pattern | Counted as | Example | Not counted |
| --- | --- | --- | --- |
| Volatile | `volatile` | `TODAY()`, `NOW()`, `RAND()`, `RANDBETWEEN()` | `SNOW()`, `MY.NOW()`, `NOW()` inside quotes |
| Indirect | `indirect` | `INDIRECT("A"&B1)`, `OFFSET(A1,0,0,5)` | (Google does not list these as volatile) |
| Cross-file | `importrange` | `IMPORTRANGE("id","Orders!A:H")` | the range written inside the quotes |
| Array | `arrayformula` | `ARRAYFORMULA(B2:B*C2:C)` |  |
| Open range | `openRanges` | `A:A`, `$B:$D`, `A2:A`, `Sales!C:C` | `A1:A10`, `A:B10`, whole rows like `1:1` |

`countCalls()` uses a lookbehind, `(?<![A-Z0-9_.])`, instead of matching the character before the function name. My first version matched that character, and a test caught the result: in `INDIRECT(OFFSET(...))` the opening bracket belongs to the first call, so the second call was never counted. Apps Script's current runtime is V8, the same engine family as Node, where I tested it.

## The audit: every tab, one report

The second block walks the tabs, sends every formula through the scanner, and writes one row per tab. It also writes four lines for the whole file.

```javascript
const CELL_CAP = 20000000; // Google's per-spreadsheet cell limit
const KEYS = [
  'volatile', 'indirect', 'importrange', 'arrayformula', 'openRanges',
];

function summarize(tabs) {
  const rows = tabs.map(t => {
    const row = { name: t.name, grid: t.maxRows * t.maxCols,
      formulas: 0 };
    KEYS.forEach(k => { row[k] = 0; });
    t.formulas.forEach(line => line.forEach(cell => {
      const hit = scanFormula(cell);
      if (!hit) return;
      row.formulas++;
      KEYS.forEach(k => { row[k] += hit[k]; });
    }));
    return row;
  });
  const grid = rows.reduce((sum, r) => sum + r.grid, 0);
  return { rows, grid, capShare: grid / CELL_CAP };
}

function auditSheetHealth() {
  const ss = SpreadsheetApp.getActive();
  const tabs = ss.getSheets()
    .filter(sh => sh.getName() !== 'Sheet health')
    .map(sh => ({
      name: sh.getName(),
      maxRows: sh.getMaxRows(),
      maxCols: sh.getMaxColumns(),
      formulas: sh.getDataRange().getFormulas(),
    }));
  const report = summarize(tabs);
  const file = DriveApp.getFileById(ss.getId());
  const out = ss.getSheetByName('Sheet health') ||
    ss.insertSheet('Sheet health');
  out.clearContents();
  const head = ['Tab', 'Grid cells', 'Formulas'].concat(KEYS);
  const body = report.rows.map(r =>
    [r.name, r.grid, r.formulas].concat(KEYS.map(k => r[k])));
  out.getRange(1, 1, 1, head.length).setValues([head]);
  out.getRange(2, 1, body.length, head.length).setValues(body);
  out.getRange(body.length + 3, 1, 4, 2).setValues([
    ['Grid cells, whole file', report.grid],
    ['Share of 20M cell cap', report.capShare],
    ['Named editors + viewers', file.getEditors().length +
      file.getViewers().length],
    ['General access', String(file.getSharingAccess())],
  ]);
}
```

`summarize()` is pure, so the totals are tested with invented tabs. `auditSheetHealth()` is the part that touches Google: it reads `getMaxRows()`, `getMaxColumns()` and `getFormulas()`, then writes the report.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/d624e301-3aea-4f11-b627-3f6a9c5d87c6.png align="center")

Two of the file-level lines are there for Sign 2 (staff who can open every tab). `getEditors()` and `getViewers()` list the people who were added by name, and Google's reference says `getViewers()` includes commenters. Neither list shows people who reach the file through a link. That is what `getSharingAccess()` is for. If the General access line says ANYONE_WITH_LINK, the named list undercounts, and the master file is not private the way the guide's Sign 2 fix needs it to be.

## Three inputs the Sign 1 code let through

Our guide fixes Sign 1, people overwriting each other, with a `recordSale()` function that takes a script lock before it changes stock. The lock is right. What I tested was the rest of the function, with a fake `SpreadsheetApp` and a three-row Stock tab. These three calls behaved differently from what the code intends:

| Call | Original `recordSale()` | Fixed version |
| --- | --- | --- |
| `recordSale("1001", 2)`, SKU sent as text by a web app | Unknown SKU 1001 | stock 5 becomes 3 |
| `recordSale(1001, -3)` | stock 5 becomes 8 | Quantity must be a whole number above 0 |
| `recordSale("A-7", 2)`, stock cell holds the text "twelve" | writes `NaN` into the cell | Stock for A-7 is not a number |

The first one matters most, because it is how the function will actually be called. The guide pairs this function with a form or a web app, and those pass every value as text. `getValues()` returns a typed SKU like 1001 as a number, and `===` never treats the text "1001" as equal to the number 1001. The sale fails as "Unknown SKU" even though the row is right there.

The second one is a sale of -3 that adds three units to stock. The third one writes `NaN` into the sheet, because `"twelve" < 2` is false in JavaScript, so the stock check passes, and then `"twelve" - 2` is not a number.

The fixed version keeps the lock exactly as it was and adds three checks:

```javascript
function recordSale(sku, qty) {
  const key = String(sku).trim();
  const n = Number(qty);
  if (!Number.isInteger(n) || n < 1) {
    throw new Error('Quantity must be a whole number above 0');
  }
  const lock = LockService.getScriptLock();
  lock.waitLock(10000); // throws if another sale holds it for 10s
  try {
    const sheet = SpreadsheetApp.getActive().getSheetByName('Stock');
    const rows = sheet.getDataRange().getValues();
    const i = rows.findIndex(r => String(r[0]).trim() === key);
    if (i < 1) throw new Error('Unknown SKU ' + key);
    const onHand = rows[i][2];
    if (typeof onHand !== 'number') {
      throw new Error('Stock for ' + key + ' is not a number');
    }
    if (onHand < n) throw new Error('Only ' + onHand + ' left');
    sheet.getRange(i + 1, 3).setValue(onHand - n);
    SpreadsheetApp.flush(); // write before releasing the lock
  } finally {
    lock.releaseLock();
  }
}
```

The quantity check runs before `waitLock()`, so a bad request is rejected without making the next sale wait. One test checks that no lock is taken for -3. The stock check is strict on purpose: an empty cell or a number stored as text throws an error instead of guessing. A clear error at the till is better than a wrong number in the sheet.

## Pitfalls

- **SKUs with leading zeros.** If someone types 00123 into an ordinary cell, Sheets stores the number 123. `String()` then gives "123", and a request for "00123" still fails. Format the SKU column as plain text before entering data. The last test in my suite pins this down as a known limit.
- **The scanner reads formulas from the data range only.** `getDataRange()` stops at the last row and column that hold content. Conditional formatting rules, data validation and named ranges also hold formulas, and the audit does not read them.
- **Open ranges inside text.** `IMPORTRANGE("id","Orders!A2:H")` asks the other file for an open range, but the range is written as text, so `stripLiterals()` hides it. The audit counts the IMPORTRANGE call, not the range inside it.
- **Whole-row ranges.** `1:1` is not counted. I left it out because I have not seen it cause trouble in the sheets I have worked on, and that is not a measurement.
- **Permissions.** `getEditors()` and `getViewers()` return empty lists if the person running the script cannot edit the file. Run the audit as the owner. `DriveApp` also asks for wider Drive access the first time, which some admins will question.

## How it drifts after you set it up

A clean report is a snapshot, and sheets do not stay clean.

1. **A new tab copies an old habit.** Someone builds a summary with `SUMIF(Orders!D:D, ...)` because that is what the last summary did. The open-range count goes up one tab at a time.
2. **Helper columns pick up TODAY().** "Days since order" is a common column, and it is volatile in every row it fills. In my invented example, one helper column put 1,200 volatile calls into a single tab.
3. **The grid grows faster than the data.** Imports and pasted exports add thousands of empty rows and columns. Grid cells is the number to watch here, not row count.
4. **Sharing widens quietly.** A link shared "just for this week" stays shared. Run the audit on a monthly trigger and compare the General access line with last month's.

## What I tested, and what I did not

Measured: `test.js` runs 60 checks against the code in this post: 34 for the scanner, 7 for the summary and 19 for `recordSale()`, the original and the fixed version side by side. All 60 pass in UTC and in America/Los_Angeles. I then broke the code on purpose in 21 different ways (removed the literal stripping, dropped RAND from the list, moved the quantity check inside the lock, and so on), and the tests failed for all 21. The mutation run also exposed two pieces of dead code, which I deleted, and the nested-call bug described above, which I fixed.

Not measured:

- I have not run `auditSheetHealth()` in a live spreadsheet. `SpreadsheetApp` and `DriveApp` do not exist outside Google, so the code that calls them was reviewed against Google's reference, not executed.
- Google's limits page gives the 20 million figure but does not say whether empty grid cells count toward it. I report `getMaxRows() × getMaxColumns()` because that is the number you can shrink by deleting empty rows. The "share of cap" line is a guide, not Google's own count.
- I have not timed how much any of these patterns slows a real sheet. The counts tell you where to look. They do not say how many seconds each one costs.
- I ran the lookbehind regex in Node, not inside Apps Script itself.
- All workbook figures in the demo are invented.

## What the report can and cannot decide

The audit measures Sign 3 and part of Sign 2. The other four signs in the guide (people overwriting each other, field staff on phones, no record of who changed what, branches with their own copies) are about how people work, and no formula scan finds them.

So the report answers one narrow question. If the Sheet health tab shows hundreds of volatile calls and open ranges, fix those first and look again in a week. If it is clean and the sheet is still slow, or if the other signs keep coming up every week, the sheet really has hit its limits.

The full checklist for those other five signs, and how to score them, is in [the MageSheet guide to outgrowing Google Sheets](https://magesheet.com/blog/outgrown-google-sheets-signs).