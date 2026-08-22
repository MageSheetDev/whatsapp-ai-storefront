---
title: "When a Commission Rate Changes, Last Quarter's Payouts Shouldn't"
seoTitle: "Commission Rule Versioning in Google Sheets (Apps Script)"
seoDescription: "A rate change should not reprice a paid quarter. Date-versioned commission rules, derived entry ids and an append-only ledger in Apps Script."
datePublished: 2026-08-22T16:31:18.502Z
cuid: cmt4li37x00000ai58fjtfguu
slug: when-a-commission-rate-changes-last-quarter-s-payouts-shouldn-t
canonical: https://magesheet.com/blog/commission-tracking-google-sheets-guide
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/d0cc621e-3029-4918-9bce-7de26d016dc0.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/ce523a90-ea0d-4423-a8cb-31bbe46edc6f.jpg
tags: javascript, automation, google-sheets, google-apps-script, magesheet

---

A rep opened her March statement in August and the total had moved by $312. Nobody had edited the ledger. Finance had raised her rate in the `Rules` tab in July, the nightly recompute walked the full transaction history, and every March sale was suddenly paid at July's rate.

Both numbers were arithmetically correct. That is exactly what made it impossible to argue with, and impossible to trust. Once a rep learns that last quarter's number can move, every future number is provisional.

I have written up the two layers around this one already: the [attribution engine](https://magesheet.com/blog/b2b-sales-commission-tracking) that decides whose sale it was, and the [tiered calculator](https://magesheet.com/blog/whatsapp-automation-small-businesses-2026) that turns a sale into an amount. This post is the layer that keeps both honest over time: rules that carry dates, ledger entries whose id comes from the facts, and a log that only ever grows.

## Why one rate per rep breaks

The usual `Rules` tab has one row per rep: `rep_id`, `pct`, maybe a quota. That table answers "what is Ayse's rate", but a payout question is never that. It is "what was Ayse's rate on 10 March, when this charge settled".

A single-row rules tab cannot answer it, so every read silently substitutes today's answer. Three failures follow from that one gap:

*   A rate change repriced quarters that were already paid. The recompute is doing what you told it to.
    
*   Someone copied a plan row instead of closing the old one, both rows match, and a first-match lookup picks whichever sorted first. No error, no warning.
    
*   A sale dated on the switchover day lands on the wrong side of it, because the sheet hands you a local-midnight `Date` and your comparison string parses as UTC midnight.
    

The fix is not more careful editing. It is making the rate a fact with a date range around it, and making the write path unable to overwrite what it already wrote.

## 1\. Rules carry dates, and the lookup carries the sale date

The `Rules` tab gets two more columns, and rows are never edited in place. Changing a rate means closing the current version and adding a new one:

| rule\_id | rep\_id | effective\_from | effective\_to | pct |
| --- | --- | --- | --- | --- |
| R-2025 | ayse | 2025-01-01 | 2026-03-01 | 0.08 |
| R-26Q1 | ayse | 2026-03-01 | 2026-07-01 | 0.10 |
| R-26Q3 | ayse | 2026-07-01 |  | 0.12 |
| R-mert | mert | 2026-01-01 |  | 0.05 |

The ranges are half-open: `effective_from` is included, `effective_to` is not. That is what lets one version end and the next begin on the same date without a sale on that day matching both or neither. An empty `effective_to` means the version is still current.

```javascript
/**
 * Midnight for a sheet Date or a yyyy-mm-dd string, in one
 * calendar space. Sheets hands you a local-midnight Date;
 * new Date('2026-03-01') is UTC midnight. Mixing them shifts
 * a sale across a rule boundary.
 */
function toDay(value) {
  if (value instanceof Date) {
    return Date.UTC(
      value.getFullYear(), value.getMonth(), value.getDate()
    );
  }
  const p = String(value).slice(0, 10).split('-');
  return Date.UTC(Number(p[0]), Number(p[1]) - 1, Number(p[2]));
}

/**
 * The rule version in force on the sale date. Versions are
 * half-open: [effectiveFrom, effectiveTo). An empty
 * effectiveTo means the version is still current.
 */
function ruleForDate(repId, saleDate, rules) {
  const at = toDay(saleDate);
  const hits = rules.filter(function (r) {
    if (r.repId !== repId) return false;
    const from = toDay(r.effectiveFrom);
    const to = r.effectiveTo ? toDay(r.effectiveTo) : null;
    return at >= from && (to === null || at < to);
  });
  if (hits.length === 0) return null;
  if (hits.length > 1) {
    throw new Error(
      'Overlapping rules for ' + repId + ' on ' +
      new Date(at).toISOString().slice(0, 10) + ': ' +
      hits.map(function (r) { return r.ruleId; }).join(', ')
    );
  }
  return hits[0];
}
```

Two decisions in there are worth stating out loud.

`ruleForDate` throws on overlap instead of returning the first hit. A duplicated plan row is a data error, and the moment it resolves quietly you get a payout that nobody can reproduce. Failing the run costs you one morning; a silent first-match costs you a dispute you cannot answer.

`toDay` exists because a date in Apps Script arrives in two shapes. Read the sale date from the sheet and you get a `Date` at local midnight. Read the rule boundary from a `yyyy-mm-dd` string and `new Date()` gives you UTC midnight. In a UTC+3 sheet those are three hours apart, which is enough to push a 1 March sale back into February's version. Both go through `toDay` before anything is compared.

With that table, `ruleForDate('ayse', '2026-02-28', RULES)` returns `R-2025` at 8 percent and `ruleForDate('ayse', '2026-03-01', RULES)` returns `R-26Q1` at 10 percent. A $10,000 March sale is worth $1,000 in March and it is still worth $1,000 when the job re-runs in August, when Ayse's current rate is 12 percent.

## 2\. The entry id comes from the facts, not from a counter

Knowing the right rate is half of it. The other half is a writer that can run twice without paying twice, because it will: a trigger overlaps its own previous run, a webhook retries, someone clicks the menu item again after a timeout.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/b3a513f2-d233-4f57-b6d1-3460455e2b97.jpg align="center")

The trick is to stop treating the entry id as a sequence number and start deriving it from the three facts that define the entry: which transaction, which rep, which rule version.

```javascript
function round2(n) {
  return Math.round(n * 100) / 100;
}

/**
 * One ledger entry for one credit row. entryId is derived from
 * the facts, so a re-run rebuilds the same id and the writer
 * can skip it instead of paying twice.
 */
function entryFor(txn, credit, rule, commission, computedAt) {
  return {
    entryId: [txn.txnId, credit.repId, rule.ruleId].join('|'),
    txnId: txn.txnId,
    repId: credit.repId,
    saleDate: txn.date,
    amount: round2(txn.amount * credit.creditPct),
    commission: round2(commission),
    ruleId: rule.ruleId,
    reversesEntryId: '',
    computedAt: computedAt
  };
}

/**
 * Append-only writer. Returns the rows to append and the ids
 * it skipped; it never edits or deletes an existing row.
 */
function postEntries(candidates, existingIds) {
  const seen = {};
  existingIds.forEach(function (id) { seen[id] = true; });
  const toAppend = [], skipped = [];
  candidates.forEach(function (e) {
    if (seen[e.entryId]) { skipped.push(e.entryId); return; }
    seen[e.entryId] = true;
    toAppend.push(e);
  });
  return { toAppend: toAppend, skipped: skipped };
}
```

`postEntries` takes the ids already in the `Ledger` tab and returns two lists, so the caller does one read and one append. In Apps Script that matters: pull the id column once with `getRange(...).getValues()`, hand it in, and write the result in a single `setValues` call rather than one `appendRow` per entry. Wrap the read-decide-write sequence in `LockService.getScriptLock()` so two overlapping triggers cannot both decide the id is new.

The derived id also gives you the behaviour you actually want when a rule genuinely changes. A correction re-run with a different rule version produces `ch_1|ayse|R-26Q3`, which is not `ch_1|ayse|R-26Q1`, so it appends as a new row and the original stays exactly where it was. The ledger shows both, with `rule_id` and `computed_at` on each, and the difference between them is a visible correction instead of a number that moved.

The split case falls out of the same shape. A deal credited 70 percent to the AE and 30 percent to the BDR is two credit rows, so it is two candidate entries with two different ids, each priced against that rep's own rule version. On the March sale above: Ayse gets $7,000 of credited revenue at 10 percent, which is $700, and Mert gets $3,000 at his 5 percent, which is $150.

## 3\. A clawback is a row, not an edit

When the refund webhook arrives, the temptation is to find the original ledger row and zero it out. Do that and the March statement you already sent no longer matches the March statement in the sheet, which is the same trust problem as the rate change wearing different clothes.

```javascript
/**
 * A clawback is a new negative row dated to the refund, not an
 * edit of the original. Returns null outside the window.
 */
function reversalFor(entry, refundDate, windowDays) {
  const days =
    (toDay(refundDate) - toDay(entry.saleDate)) / 86400000;
  if (days > windowDays) return null;
  return {
    entryId: entry.entryId + '|rev',
    txnId: entry.txnId,
    repId: entry.repId,
    saleDate: entry.saleDate,
    amount: -entry.amount,
    commission: -entry.commission,
    ruleId: entry.ruleId,
    reversesEntryId: entry.entryId,
    computedAt: refundDate
  };
}

/**
 * A rep's statement for one payout period: every row whose
 * computedAt falls inside it, positives and reversals netted.
 */
function statementFor(repId, from, to, ledger) {
  const start = toDay(from), end = toDay(to);
  const lines = ledger.filter(function (e) {
    const at = toDay(e.computedAt);
    return e.repId === repId && at >= start && at < end;
  });
  const total = lines.reduce(function (sum, e) {
    return sum + e.commission;
  }, 0);
  return { repId: repId, lines: lines, total: round2(total) };
}
```

The window is measured from the sale date, not from the day the entry was computed, because the policy a rep agreed to is about their sale. The reversal is dated to the refund, so it lands in the period where the money actually moves. And it keeps the original `ruleId`: you are undoing a specific priced entry, not repricing it at today's rate.

Carry the March example through. Ayse's ledger holds `ch_1|ayse|R-26Q1` at $700 computed on 11 March. The customer refunds on 5 April, inside a 90-day window, so a second row appends: `ch_1|ayse|R-26Q1|rev` at -$700, dated 5 April. March still pays $700. April carries -$700. Across the year the two rows net to zero, and every statement you have already sent still reproduces from the ledger.

Because the reversal id is derived the same way, running the refund handler twice appends one row, not two.

## What the tests caught

I ran the pure logic under Node with a fake calendar before any of it touched a sheet: 50 assertions across `toDay`, `ruleForDate`, `entryFor`, `postEntries`, `reversalFor` and `statementFor`. Four things only showed up there.

The date-shape bug is real and quiet. Comparing a sheet `Date` directly against `new Date('2026-03-01')` puts a 1 March sale in February's version in any sheet east of UTC. It never throws; it just pays 8 percent where the plan says 10.

Overlapping rule versions resolved to a plausible answer. Before the throw, a duplicated row returned `R-26Q1` while `R-dup` matched the same day just as well. The test that pins this asserts an exception, not a value.

The clawback boundary is off by one in the direction nobody checks. With a 90-day window on a 10 March sale, 8 June is inside and 9 June is outside. Write it as `days > windowDays` returning null, and test both days, or you will discover which side you chose during a dispute.

Rounding belongs at entry time. `1234.56 * 0.07` is `86.4192`, and summing raw floats across a quarter drifts from what each line shows. Every entry rounds to cents when it is created, so a statement total is the sum of the numbers the rep can see.

## Pitfalls

*   **Never key an entry on its row number.** Rows shift when someone sorts the tab, and a re-run then reconciles against the wrong row. The id has to survive a sort.
    
*   **Never backdate a rule version to fix a payout.** Add a correcting entry instead. A backdated version rewrites every statement that quoted it, which is the failure this whole design exists to prevent.
    
*   **Close the old version on the day the new one starts.** Leaving `effective_to` empty on both is how you get the overlap exception, usually on the first run after a plan change.
    
*   **A rule version change is not a bug fix.** If the old rate was genuinely wrong, the visible correction row is the honest artifact. Two rows and a reason beat one row that quietly moved.
    
*   **Batch the writes.** Apps Script gives you six minutes per execution, and one `appendRow` per entry burns it on a backfill. Read once, append once, and have the trigger resume from the last processed transaction.
    
*   **Give each rep a view of their own rows only.** A ledger everyone can read stops being a payout tool and becomes salary politics.
    
*   **Know the ceiling.** This holds comfortably to roughly 15 reps and a few thousand transactions a quarter. Past that the ledger wants a real database, and the honest answer is to say so before you build it.
    

## Wrap-up

A commission system earns trust by being reproducible, not by being clever. Three properties get you there: rate rows carry the dates they were in force, the lookup asks with the sale date, and the writer derives its id from the facts so the log can only grow. Once those hold, a re-run is a no-op, a refund is a row, and a statement you sent in March still reproduces in August.

The production version, with Stripe and Gumroad ingestion, the per-rep tokenized view, and the payroll export, is written up on the [MageSheet blog](https://magesheet.com/blog/commission-tracking-google-sheets-guide).

Built by the MageSheet team.