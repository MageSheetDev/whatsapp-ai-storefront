---
title: "Build vs Buy in Google Sheets: The Break-Even Cell That Said -400 Months"
seoTitle: "Build vs Buy Break-Even in Google Sheets: Seats and Renewals"
seoDescription: "Three Apps Script custom functions for build vs buy: a guarded break-even, a projection with seats and renewals, and a verdict."
datePublished: 2026-11-05T17:59:47.319Z
cuid: cmw3uorl4000002mif5gfclyz
slug: build-vs-buy-break-even-google-sheets
canonical: https://magesheet.com/blog/build-vs-buy-apps-script-saas
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/5ad3911a-c82e-476e-a327-6e7193c8524f.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/563897e6-4c3c-4ea8-ad96-350dcb06ece3.jpg
tags: saas, automation, google-apps-script, magesheet

---

Our build-vs-buy guide gives one formula for deciding whether to replace a subscription with your own automation: break-even months = B / (S − R − M). B is the one-time build cost, S the subscription per month, R what the build costs to run, M monthly maintenance. In a sheet it is one cell: `=B2/(B1-B3-B4)`.

Run it with a small tool's numbers: 10 seats at $12, a $12,000 build quote, and maintenance at the guide's fallback of 15% of the build a year. The cell comes out at **\-400**. Put the obvious check next to it, `=IF(B5<=36,"BUILD","BUY")`, and it says **BUILD**. Upkeep alone was $150 a month against a $120 subscription, so the build never pays back, and the sheet recommended it anyway.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/b6167beb-f000-4a72-8adf-58530934c7df.gif align="center")

The guide also says to project the subscription forward with renewal price rises and headcount, because that "often halves the break-even". I built that projection too. Below are three custom functions that do the formula, the projection and the verdict, with the cases where the plain cell gives the wrong answer.

## Why one cell is not enough

The formula itself is right. Three things around it go wrong in a spreadsheet:

1. **Division does not know what a negative saving means.** When S − R − M is zero or below, the build never pays back. Sheets returns `#DIV/0!` at exactly zero, which at least looks broken, and a negative number below zero, which looks like an answer. Any comparison against a horizon reads -400 as "very fast".
2. **S is today's invoice.** A per-seat bill moves on two axes, the vendor's price at each renewal and your headcount. Holding S flat for three years answers a question nobody is asking.
3. **The verdict needs a rule for "close".** The guide gives three outcomes: break-even well inside how long the process will last means build, beyond it means buy, and close to it means buy, because cancelling a subscription is easy and unbuilding a build is not. A single IF has only two.

The classification comes first, and the guide is clear about it: payroll, email delivery and payment processing stay bought whatever the arithmetic says. The functions below are only for a process that is specific to how you work and has run the same way for a year or more.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/0b149abc-b6e9-4e3a-bfba-ce439e583a4d.png align="center")

## Block 1: the formula, guarded

`BREAKEVEN` takes the same four numbers. An empty maintenance cell uses the guide's fallback, 15% of B a year divided by 12. A saving of zero or less returns the word `never`, which no numeric comparison can mistake for a fast payback. Anything that is not a non-negative number returns `check inputs`.

```javascript
/**
 * Months until a build pays for itself: B / (S - R - M).
 * Leave maint empty to use 15% of the build cost a year.
 * @param {number} build one-time build cost, B
 * @param {number} sub subscription per month today, S
 * @param {number} run running cost of the build per month, R
 * @param {number} maint maintenance per month, M
 * @customfunction
 */
function BREAKEVEN(build, sub, run, maint) {
  const m = (maint === '' || maint === undefined)
    ? build * 0.15 / 12
    : maint;
  if (![build, sub, run, m].every(isAmount_)) {
    return 'check inputs';
  }
  const saving = sub - run - m;
  if (saving <= 0) return 'never';
  return Math.round(build / saving * 10) / 10;
}

function isAmount_(n) {
  return typeof n === 'number' && isFinite(n) && n >= 0;
}
```

With 18 seats at $12 ($216 a month), a $5,500 build and empty maintenance, `=BREAKEVEN(5500, 216, 0, "")` returns 37.4 months. The $5,500 is an invented figure; the guide does not give a build cost for its example, and neither will I pretend to.

## Block 2: the projection

`BREAKEVEN_PROJECTED` walks month by month. Each month it takes the seat count in force, multiplies it by the seat price after the renewals so far, subtracts running cost and maintenance, and adds that to a running total. The first month the total reaches the build cost is the break-even month.

Seats come from a two-column range: the month a count starts, and the count. Google's custom function guide says a range argument arrives as "a two-dimensional array of the cells' values", and that "Percentage values in Sheets become decimal numbers in Apps Script", so a cell formatted as 10% arrives as 0.1. A single cell works too, for a flat seat count.

```javascript
/**
 * Break-even month with seat changes and renewal uplift.
 * @param {number} build one-time build cost
 * @param {number} seatPrice price per seat per month today
 * @param {number[][]} seats rows of [from month, seat count]
 * @param {number} uplift price rise per renewal, as 10%
 * @param {number} run running cost of the build per month
 * @param {number} maint maintenance per month
 * @param {number} maxMonths how far to look, default 120
 * @customfunction
 */
function BREAKEVEN_PROJECTED(build, seatPrice, seats, uplift,
                             run, maint, maxMonths) {
  if (uplift >= 1) return 'enter uplift as 10%, not 10';
  const plan = seatPlan_(seats);
  const limit = maxMonths || 120;
  let saved = 0;
  for (let month = 1; month <= limit; month++) {
    const renewals = Math.floor((month - 1) / 12);
    const price = seatPrice * Math.pow(1 + uplift, renewals);
    saved += seatsIn_(plan, month) * price - run - maint;
    if (saved >= build - 1e-9) return month;
  }
  return 'not within ' + limit + ' months';
}

function seatPlan_(seats) {
  const rows = Array.isArray(seats) ? seats : [[1, seats]];
  return rows
    .filter(r => r[0] !== '' && r[1] !== '')
    .sort((a, b) => a[0] - b[0]);
}

function seatsIn_(plan, month) {
  let count = plan[0][1];
  for (const [from, n] of plan) {
    if (month >= from) count = n;
  }
  return count;
}
```

The price rises at month 13, 25 and so on, compounding: month 25 is 1.21 times today's price, not 1.20. A renewal is a step, not a slope, and the tests check both boundaries (the rise lands in month 13, not month 12).

Here is what it does to the guide's own scenario shape: 18 seats, a 10% rise at each renewal, and growth to 26 seats from month 13. Build costs are invented; maintenance is 15% of the build a year in every row.

| Build cost (invented) | Flat formula | 10% at each renewal | Plus 26 seats from month 13 |
| --- | --- | --- | --- |
| $4,000 | 24.1 months | 23 months | 19 months |
| $5,500 | 37.4 months | 33 months | 26 months |
| $8,000 | 69 months | 52 months | 38 months |
| $10,000 | 109.9 months | 68 months | 48 months |

The guide's "often halves" held once in these four cases, at the largest build. The longer the flat answer, the more the projection moves it, because more renewals and more months at 26 seats fall inside it. On a short payback the projection barely matters; on a long one it decides the case.

## Block 3: the verdict

`BUILD_OR_BUY` compares a break-even with the number of months you honestly expect the process to keep running the way it runs today. The guide asks for that judgement and does not give a threshold for "close"; I chose 75% of the horizon. Anything that is not a number, `never` or `not within 120 months`, is BUY.

```javascript
/**
 * Compares break-even with how many more months the process
 * will run the way it runs today. "Close" means past 75% of
 * that horizon.
 * @customfunction
 */
function BUILD_OR_BUY(breakEven, horizonMonths) {
  if (typeof breakEven !== 'number') return 'BUY';
  if (breakEven > horizonMonths) return 'BUY, revisit next year';
  if (breakEven > horizonMonths * 0.75) return 'BUY, too close';
  return 'BUILD';
}
```

With a 36-month horizon, the invented $5,500 case gets three different verdicts depending on which number you feed in: flat 37.4 months is `BUY, revisit next year`, the uplift alone (33 months) is `BUY, too close`, and uplift plus the seat growth (26 months) is `BUILD`. The decision lives in the inputs you leave out.

## Pitfalls

These are the input mistakes I tried against the functions, and what each one shows. The second row is not caught, and I could not find a way to catch it without asking for the billing period.

| Input | `=B2/(B1-B3-B4)` | The three functions |
| --- | --- | --- |
| Upkeep $150 a month against a $120 subscription | \-400 months; `IF(<=36)` says BUILD | `never`; `BUILD_OR_BUY` says BUY |
| $2,592 a year typed into the monthly cell | 2.2 months | 2.2 months (not caught) |
| Maintenance cell left empty | 25.5 months (empty counts as 0) | 37.4 months (15% fallback) |
| Uplift typed as 10 instead of 10% | not used | `enter uplift as 10%, not 10` |

- **Annual prices in a monthly cell.** Many vendors quote per year. Put $2,592 a year into S and the sheet says 2.2 months instead of 37.4. Neither the formula nor the function can tell; label the cell "per month" and divide annual quotes by 12 yourself.
- **The 15% fallback is large for small tools.** It is a rule of thumb for maintenance, and on a $12,000 build it is $150 a month, more than many single subscriptions. The guide notes that a small Apps Script automation touching only Google services usually costs far less to maintain; if you have a better estimate, put it in M rather than leaving it empty.
- **Unsorted or half-filled seat rows.** Hiring plans get edited. The function sorts the rows by start month and skips a row whose month or count is empty. Without the skip, a row with a month and no count reads as zero seats from that month on.

## How the answer goes stale

The break-even is a forecast, and forecasts drift. Recheck it when one of these changes:

- **The renewal quote arrives.** Replace the assumed uplift with the actual one. If the vendor moved your feature to a higher tier, that is a new S, not a 10% rise.
- **The hiring plan changes.** The seat range is the most hopeful input in the sheet. If the 26 seats in month 13 did not happen, the projection goes back toward the flat answer.
- **Maintenance turns out different.** Log what upkeep actually cost each quarter and use the average as M. A build that calls a third-party API spends more on maintenance after that API changes.
- **The horizon shrinks.** A reorganisation or a new supplier shortens how long the process will run as it does today. That moves the verdict without changing a single cost.

## What I tested, and what I did not

The three blocks above are extracted from this article and run in Node against 33 tests: the guide's formula, the negative and zero-saving cases, input checks, renewal boundaries at months 12, 13 and 25, unsorted and blank seat rows, the four table rows, the verdict thresholds, and 200 random cases where the projection with flat seats and no uplift must equal the formula rounded up to the next whole month. I then broke the code on purpose 18 ways. 16 were caught at first; of the two that were not, one was a line JavaScript made unnecessary (I deleted it) and one guarded a real case that had no test (I added the test). After that, 17 of 17 were caught.

I have not run these inside a live spreadsheet. The range and percentage behaviour comes from Google's custom function guide, quoted above. That guide does not say what an empty cell passes to a custom function, so the code treats both an empty string and a missing argument as empty, and I have not measured which one Sheets sends. Every dollar figure in the examples is invented to show the shape of the arithmetic; none of it is a vendor's price or a quote.

## The takeaway

Decide whether the process is yours before opening the sheet, then feed the break-even the bill you will actually pay: seats as you plan to hire, prices as they will be after each renewal, and `never` when upkeep eats the saving. The decision framework, the cost lists both sides hide, and the full three-year comparison are in the original [build vs buy guide on the MageSheet blog](https://magesheet.com/blog/build-vs-buy-apps-script-saas); what the build column costs to run on Apps Script is in [Apps Script pricing](https://magesheet.com/blog/google-apps-script-pricing).