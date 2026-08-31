---
title: "AppSheet's 2,500-Row Ceiling: Audit Your Sheet Before You Pick a Plan"
seoTitle: "AppSheet Row Limits: Audit Your Sheet Before You Upgrade"
seoDescription: "An Apps Script audit that counts real rows per tab, dates the 2,500-row crossing, and picks the cheapest AppSheet plan that fits."
datePublished: 2026-08-31T17:22:54.981Z
cuid: cmthib4h800000ajchn3y91v9
slug: appsheet-s-2-500-row-ceiling-audit-your-sheet-before-you-pick-a-plan
canonical: https://magesheet.com/blog/appsheet-pricing-2026
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/c66ce2d5-05c6-4073-bfa9-4c143e126c1b.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/cd91b49f-2630-41f4-a854-6f0d22e7661e.jpg
tags: google-sheets, no-code, google-apps-script, appsheet, magesheet

---

A 25-person operations team was quoted a jump from Core to Enterprise Plus — $250 a month to $500 — and nobody in the room could say which limit they had hit. Their biggest table held about 1,900 rows, comfortably under the 2,500-row ceiling everyone had memorised. The trigger was the tab count: the single spreadsheet behind the app had grown to twelve tabs, and every tab the app reads is a table.

The seat price is the number people negotiate. The shape of your spreadsheet is the number that decides which row of the price list you are standing on, and it is the one nobody measures.

## Why the pricing page can't answer the question

Here are the published 2026 plans. Treat these as the vendor's list at the time of writing — limits and prices move, so check them against your own account before you commit a budget.

| Plan | Price | Tables | Rows per table |
| --- | --- | --- | --- |
| Free (build & test) | $0, up to 10 test users | — | — |
| Starter | $5/user/mo | 5 | 2,500 |
| Core | $10/user/mo | 10 | 2,500 |
| Enterprise Plus | $20/user/mo | 200 | 200,000 |
| Publisher Pro | $50/mo per app, flat | — | — |

Read that as a seat model and you multiply headcount by price. But which row you land on is set by two ceilings — how many tables the app reads, and how many rows sit in the largest one — and neither is visible from the Sheets UI. Notice that Starter and Core share the same 2,500-row ceiling. Crossing it doesn't move you one tier, it moves you two.

(Whether to use AppSheet at all is a different question, and I've written that one up separately: [AppSheet vs Apps Script vs PowerApps](https://magesheet.com/blog/appsheet-vs-apps-script-comparison). This post assumes you've picked AppSheet and want to know what it will cost as the data grows.)

## Put the plans in code, not in a slide

The plan list is data, so make it data. Sorted by price, the first plan that fits is the cheapest one that fits.

```javascript
// AppSheet plan ceilings as published for 2026. Vendor limits move -
// verify against your own account before committing a budget.
const PLANS = [
  { id: 'starter', usd: 5,  tables: 5,   rows: 2500 },
  { id: 'core',    usd: 10, tables: 10,  rows: 2500 },
  { id: 'plus',    usd: 20, tables: 200, rows: 200000 }
];

// Cheapest plan that fits a shape, or null when nothing does.
// PLANS is price-ascending, so the first match is the answer.
function planFor(shape) {
  for (let i = 0; i < PLANS.length; i++) {
    const p = PLANS[i];
    if (shape.tables <= p.tables && shape.maxRows <= p.rows) return p;
  }
  return null;
}

// Seats are the whole bill: no volume break in the published tiers.
function monthlyCost(plan, seats) {
  if (!plan) return null;
  return plan.usd * seats;
}
```

Run the twelve-tab team through it and the quote explains itself: `planFor({ tables: 12, maxRows: 1900 })` skips Starter and Core on table count alone and returns `plus`. At 25 seats that is $500 a month against Core's $250 — an extra $3,000 a year bought by two tabs.

## Count the rows AppSheet will actually carry

The obvious way to count a tab is `getLastRow()`. It is the wrong number. It returns the last row holding content in *any* column, so one note typed into column Z, one stale formula returning an empty string, one row of leftover formatting, and your 120-row table reports thousands.

Count instead from a key column — a field every real record has, like an id or a created-at stamp — and report both numbers, because the gap between them is the thing to go clean up.

```javascript
// Rows AppSheet will carry for one tab, next to what the used range
// claims. keyCol is the 1-based column every real record fills in.
function tableStats(sheet, keyCol) {
  const used = sheet.getLastRow();
  const name = sheet.getName();
  if (used < 2) return { name: name, rows: 0, usedRows: 0 };
  const col = sheet.getRange(2, keyCol, used - 1, 1).getValues();
  let rows = 0;
  for (let i = 0; i < col.length; i++) {
    const v = col[i][0];
    if (v !== '' && v !== null && v !== undefined) rows++;
  }
  return { name: name, rows: rows, usedRows: used - 1 };
}

// Every tab the app reads counts as its own table, however many
// spreadsheets they happen to live in.
function auditBook(ss, keyCol) {
  const out = [];
  const sheets = ss.getSheets();
  for (let i = 0; i < sheets.length; i++) {
    out.push(tableStats(sheets[i], keyCol || 1));
  }
  return out;
}

// The two numbers a plan is chosen on: table count, biggest table.
function shapeOf(stats) {
  let max = 0;
  for (let i = 0; i < stats.length; i++) {
    if (stats[i].rows > max) max = stats[i].rows;
  }
  return { tables: stats.length, maxRows: max };
}
```

Both subtractions matter. `used - 1` and starting the range at row 2 drop the header, which is not a record and should never be billed as one.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/1de533ad-570c-4511-9a57-700694bce7d7.png align="center")

## Work out the date you cross the ceiling

Knowing you fit today is half the answer. The other half is the date you stop fitting, because that date is when the bill changes.

```javascript
const DAY = 86400000;
const MIN_HISTORY_DAYS = 14;

// Rows added per CALENDAR day, measured across the span the data
// covers - not across the days that happen to carry rows. A weekday
// only table has about five dated days a week, so dividing by dated
// days overstates the rate by roughly a third.
function rowsPerDay(dates) {
  if (dates.length < 2) return null;
  let lo = Infinity, hi = -Infinity;
  for (let i = 0; i < dates.length; i++) {
    const t = dates[i].getTime();
    if (t < lo) lo = t;
    if (t > hi) hi = t;
  }
  const span = (hi - lo) / DAY;
  if (span < MIN_HISTORY_DAYS) return null;  // too short to trust
  return dates.length / span;
}

// First day the table passes a ceiling. Floor, never round: a budget
// deadline must not land after the wall does.
function daysToCeiling(rows, perDay, ceiling) {
  if (!perDay || perDay <= 0) return null;
  if (rows >= ceiling) return 0;
  return Math.floor((ceiling - rows) / perDay);
}
```

`rowsPerDay` returns `null` rather than a number it cannot stand behind. Two rows a day apart is not a growth rate, and a function that answers anyway will hand you a crossing date next week.

## Pitfalls

`getLastRow()` **is not your row count.** In my tests a tab with 120 real records but a stray value at row 4,000 reports `usedRows: 3999`. Feed the honest 120 to `planFor` and you get Starter; feed the used range and you get Enterprise Plus — a four-times-per-seat difference caused by one forgotten cell.

**The header row buys an upgrade you don't need.** A tab holding exactly 2,500 records returns `getLastRow() === 2501`. Compare that raw number to the ceiling and `planFor` jumps from Starter to Enterprise Plus — $15 more per user per month, for a row that is a column title.

**Dividing by dated days, not calendar days.** Twenty weekday rows spanning 25 calendar days is 0.8 rows/day. Divide by the 20 days that carry rows and you get 1.05 — 32% too fast. Starting from 2,000 rows, the honest rate puts the 2,500 ceiling 625 days out; the inflated one says 475. You would budget the upgrade five months before you needed it.

**Rounding the crossing date up.** At 2,499 rows and 2 rows a day, rounding says one day and flooring says zero. Zero is right: you cross today. A crossing date is a deadline, and a deadline that arrives after the wall is worse than no deadline.

## Wrap-up

Two numbers decide your AppSheet plan, and neither is your headcount: how many tabs the app reads, and how many real rows sit in the biggest one. Measure both today, project the second one forward on calendar days, and you will know the month your bill changes before the vendor tells you.

The full 2026 plan breakdown — free-tier limits, Publisher Pro's flat per-app option, and the per-seat math at scale — is on the [MageSheet blog](https://magesheet.com/blog/appsheet-pricing-2026).

Built by the MageSheet team.