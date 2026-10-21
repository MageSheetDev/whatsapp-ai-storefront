---
title: "One Text Amount in getValues() Breaks the Whole Commission Total"
seoTitle: "Apps Script getValues(): Text Cells Break Commission Sums"
seoDescription: "A text amount in getValues() turns a commission sum into 49 million. Parse each cell to cents and throw on what you cannot read."
datePublished: 2026-10-21T21:26:56.782Z
cuid: cmvimhe6d000108l8bxq83xfv
slug: one-text-amount-in-getvalues-breaks-the-whole-commission-total
canonical: https://magesheet.com/blog/commission-tracker-app-comparison
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/778127a8-7dd1-4a49-8b01-bb743688a8d9.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/4ccc09b6-b087-4ecf-a20b-4c0c821e948e.jpg
tags: javascript, automation, google-sheets, google-apps-script, magesheet

---

A commission statement is the sum of a rep's deal amounts times the rate. The loop that computes it is four lines long, and it can return a figure wrong by millions without raising an error. In my comparison of [commission tracker apps and spreadsheets](https://magesheet.com/blog/commission-tracker-app-comparison), silent failure was the spreadsheet risk I weighed most. This is one concrete case, in the step where amounts leave the sheet.

## The trap

`getValues()` returns whatever each cell holds. From Google's Range reference (page last updated 2026-08-19 UTC, read 2026-10-21): "The values may be of type Number, Boolean, Date, or String, depending on the value of the cell. Empty cells are represented by an empty string in the array."

So an amount column is an array of numbers until one cell contains text. Then the obvious loop stops adding:

```javascript
// The version that looks right: it only works while every cell
// in the amount column is a real number.
function commissionNaive(rows, repId, rate) {
  let total = 0;
  rows.forEach(r => {
    if (r[1] === repId) total += r[2];
  });
  return total * rate;
}
```

Take one rep with `700`, the text `'300'` and `250`. JavaScript computes `700 + '300'` as the string `'700300'`, then `'700300' + 250` as `'700300250'`, and the multiplication turns that back into 49,021,017.50 instead of 87.50. A blank cell in the middle does the same: `700`, `''`, `250` give 49,017.50. No exception, no log line. A text amount like `'1,200.50'` fails more visibly: the total becomes `NaN`.

## The fix: parse every amount, and refuse what you cannot read

This version turns each cell into integer cents. Numbers pass through. Text such as `'1,200.50'`, `'$1,200.50'` or `' 300 '` goes through a strict pattern. Anything else throws, naming the sheet row:

```javascript
const RATE_BP = 700;  // 7.00% in basis points (1 bp = 0.01%)

// "1,200.50", "$1,200.50" or " 300 " -> cents. Anything
// else throws, with the sheet row, instead of guessing.
function toCents_(v, sheetRow) {
  if (typeof v === 'number') return Math.round(v * 100);
  const s = String(v).trim().replace('$', '');
  const ok = /^-?(\d{1,3}(,\d{3})+|\d+)(\.\d+)?$/.test(s);
  if (!ok) {
    throw new Error(
      'Row ' + sheetRow + ': not an amount: ' + JSON.stringify(v)
    );
  }
  return Math.round(Number(s.replace(/,/g, '')) * 100);
}

// rows = Deals data without the header: [dealId, repId, amount]
function statementFor(rows, repId) {
  let cents = 0;
  rows.forEach((r, i) => {
    if (r[1] !== repId) return;
    cents += toCents_(r[2], i + 2);   // +2: header + 1-based
  });
  return {
    repId: repId,
    saleCents: cents,
    commissionCents: Math.round(cents * RATE_BP / 10000)
  };
}
```

Three choices are deliberate. Unreadable cells throw instead of counting as zero: a stop at `Row 3: not an amount: "TBD"` costs ten minutes, a deal quietly left out costs a rep's trust. The pattern accepts only US-style grouping, so `'1.200,50'` throws; loosened, stripping commas would turn it into 1.2005 with no warning. And other reps' rows are skipped before parsing, so one bad cell does not block everyone, though it stays unnoticed until that rep's turn.

Cents also remove float drift: `19.99 * 100` is `1998.9999999999998` before rounding. Rounding the commission itself per entry is a different problem, covered in [commission rate changes and the append-only ledger](https://magesheet.com/blog/commission-rule-versioning-append-only-ledger).

The part that touches the spreadsheet:

```javascript
function runStatement() {
  const rows = SpreadsheetApp.getActive()
    .getSheetByName('Deals')
    .getDataRange().getValues().slice(1);
  const s = statementFor(rows, 'ayse');
  Logger.log('%s: sales %s, commission %s',
    s.repId, (s.saleCents / 100).toFixed(2),
    (s.commissionCents / 100).toFixed(2));
}
```

`.slice(1)` drops the header, so `i + 2` is the real sheet row.

## What I tested, and what I did not

I ran the blocks in Node with a mocked `SpreadsheetApp` and `Logger`: 35 checks, 35 passed. They cover the naive loop's two wrong totals above, number and text amounts, negative refunds, European and misplaced-comma strings, blanks, error strings, booleans, dates, another rep's bad cell, row numbers and the header. I then broke the code in 12 places, such as loosening the pattern, skipping bad cells or flooring the commission. All 12 failed at least one check. I also re-ran the checks on code extracted back out of this post.

What I did not measure:

*   I did not run this against a live spreadsheet. The mock returns strings where I put strings; it shows what the code does with text, not how Sheets decides a cell is text.
    
*   I have not reproduced how a number becomes text (a CSV or CRM paste, a plain-text cell format, a leading apostrophe). Those are likely causes, not measured ones.
    
*   I did not test `getDisplayValues()`. Google's reference says it returns strings that follow the cell's formatting and locale.
    
*   The 7% rate and every amount are example data.
    

## Pitfalls

**Do not fix it with** `Number(v)` **alone.** `Number('')` is 0, so a blank amount disappears, and `Number('1,200.50')` is `NaN`.

## Takeaway

When a script sums a column read with `getValues()`, convert each cell yourself and throw on what you cannot parse. The wider comparison, including when a commission tracker app is the better buy, is in the [MageSheet blog post on commission tracker apps](https://magesheet.com/blog/commission-tracker-app-comparison).

Built by the MageSheet team.