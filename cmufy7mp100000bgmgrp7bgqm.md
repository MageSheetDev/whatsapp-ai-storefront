---
title: "Two Kits, One Shared Part: The Stock Your Sheet Counts Twice"
seoTitle: "Google Sheets Inventory: Kit Stock From Components"
seoDescription: "A kit row with its own quantity column oversells. Compute every kit from free component stock and hold order claims on the parts."
datePublished: 2026-09-24T19:52:15.811Z
cuid: cmufy7mp100000bgmgrp7bgqm
slug: two-kits-one-shared-part-the-stock-your-sheet-counts-twice
canonical: https://magesheet.com/blog/google-sheets-inventory-template
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/3b11ad02-3ceb-469c-aa20-4260b34cd0ce.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/165b5ec2-4971-4f1e-8e26-e42d15630669.jpg
tags: ecommerce, google-sheets, inventory-management, google-apps-script, magesheet

---

A Google Sheets inventory template gives every SKU one row and one number: quantity on hand. That holds for as long as every SKU is a thing sitting on a shelf.

Then you list two bundles. Desk Set A is a body and a black cap. Desk Set B is the same body and a grey cap. You have five bodies, five black caps, five grey caps, so the sheet gets two more rows, and both of them say five.

Both numbers are correct. Their sum is not. The storefront offers ten desk sets and there are five bodies in the building.

Nothing about that failure looks like a bug while it is happening. The rows are consistent, the formulas add up, and the oversell arrives a week later as a cancelled order and an apology.

## A kit does not have stock, it has a minimum

The mistake is giving a kit a quantity column at all. A kit is not something you own, it is something you could build, and what you could build is a question about the parts.

```javascript
// Components are [{ sku, per }]. The answer is the smallest number
// of whole kits every component can cover, never an average, and
// never whichever component happens to be first in the list.
function kitAvailability(components, onHand) {
  if (!components || !components.length) return 0;
  var best = Infinity;

  for (var i = 0; i < components.length; i++) {
    var c = components[i];
    var per = Number(c.per);
    if (!(per > 0) || Math.floor(per) !== per) {
      throw new Error('Components per kit must be whole: ' + c.sku);
    }

    var have = Number(onHand[c.sku]);
    if (!isFinite(have)) {
      throw new Error('Component not in the inventory sheet: ' + c.sku);
    }

    var sets = Math.floor(have / per);
    if (sets < best) best = sets;
  }

  return best < 0 ? 0 : best;
}
```

Three of those lines are there because a test said so.

`Math.floor(have / per)` because a kit that needs two caps out of five caps is two kits and one orphan, not two and a half. The remainder is real stock and it is not sellable as that kit.

`best < 0 ? 0 : best` because on-hand goes negative in every real sheet — a sale recorded before the receipt, a count corrected downward — and a negative divided by a positive is a negative number of kits, which will happily render on a product page.

The throw on a missing component is the one that matters most. A component that is not in the inventory sheet is a typo in the recipe, and the tempting alternative is to skip that line. Skipping it means the kit is priced on the parts you did list, which is how a bundle goes on sale with a part nobody stocks.

## The number has to come from free stock

On-hand is not what you can sell. Five bodies with three already claimed by open orders is two bodies, and the July build of this dashboard has the [reorder point and allocation side of that](https://magesheet.com/blog/ecommerce-realtime-inventory-dashboard) in full, so this post stays on the kit.

What changes with kits is where the claim lives:

```javascript
function freeStock(onHand, committed) {
  var free = {};
  Object.keys(onHand).forEach(function (sku) {
    var c = Number((committed || {})[sku]) || 0;
    free[sku] = Number(onHand[sku]) - c;
  });
  return free;
}

function catalogAvailability(kits, onHand, committed) {
  var free = freeStock(onHand, committed);
  var out = {};
  Object.keys(kits).forEach(function (kitSku) {
    out[kitSku] = kitAvailability(kits[kitSku], free);
  });
  return out;
}
```

`freeStock` walks the stock rows, not the commitments, which sounds like a detail and is a decision. A commitment against a SKU that has no stock row is a recipe pointing at a part you no longer carry, and adding it to the map would invent a component with negative stock and quietly drag a kit to zero. Walking the stock rows leaves that fault where it can be seen.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/1094decd-6dc4-41c2-bc62-0ebab5e97286.png align="center")

## Committing a kit claims parts, not kits

This is the piece that makes the two numbers stop lying to each other. When an order takes a kit, nothing is claimed against the kit, because there is nothing there to claim. The claim lands on every component:

```javascript
function commitKit(kitSku, count, kits, onHand, committed) {
  var components = kits[kitSku];
  if (!components) throw new Error('Unknown kit: ' + kitSku);

  var free = freeStock(onHand, committed);
  var possible = kitAvailability(components, free);
  if (count > possible) {
    return { ok: false,
             reason: 'only ' + possible + ' of ' + kitSku +
                     ' can be built from free stock' };
  }

  var next = {};
  Object.keys(committed || {}).forEach(function (k) {
    next[k] = committed[k];
  });
  components.forEach(function (c) {
    next[c.sku] = (Number(next[c.sku]) || 0) + c.per * count;
  });

  return { ok: true, committed: next };
}
```

Now the five bodies behave like five bodies. Sell three Desk Set A and Desk Set B drops from five to two without anybody buying one, which is the whole point: the two kits were never independent, and the sheet was the only thing that thought they were.

A short test tells that story better than a paragraph:

```javascript
var r1 = commitKit('DESK-A', 5, KITS, STOCK, {});
var r2 = commitKit('DESK-B', 1, KITS, STOCK, r1.committed);

// r1.ok === true
// r2.ok === false, 'only 0 of DESK-B can be built from free stock'
```

The refusal returns a reason and changes nothing. An order that cannot be built has to stop at the point where it is still a decision, not halfway through a commitment map.

## The refresh, in one write

```javascript
var SHEETS = { inv: 'Inventory', kits: 'Kits', open: 'Commitments' };

function refreshKitAvailability() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();

  var onHand = mapFromRows(ss.getSheetByName(SHEETS.inv), 0, 2);
  var committed = mapFromRows(ss.getSheetByName(SHEETS.open), 0, 1);

  var kits = {};
  ss.getSheetByName(SHEETS.kits).getDataRange().getValues()
    .slice(1).forEach(function (r) {
      if (!r[0]) return;
      kits[r[0]] = kits[r[0]] || [];
      kits[r[0]].push({ sku: r[1], per: Number(r[2]) });
    });

  var avail = catalogAvailability(kits, onHand, committed);

  var sheet = ss.getSheetByName(SHEETS.kits);
  var rows = sheet.getDataRange().getValues().slice(1);
  var out = rows.map(function (r) { return [avail[r[0]]]; });
  sheet.getRange(2, 4, out.length, 1).setValues(out);
}

function mapFromRows(sheet, keyCol, valCol) {
  var out = {};
  sheet.getDataRange().getValues().slice(1).forEach(function (r) {
    if (!r[keyCol]) return;
    out[r[keyCol]] = (Number(out[r[keyCol]]) || 0) + Number(r[valCol]);
  });
  return out;
}
```

`mapFromRows` sums rather than assigns, because a commitments tab has one row per order line and the same component appears on many of them. An assignment would keep the last row and silently forget every other open order for that part.

The single `setValues` at the end is the same batching rule as the rest of this dashboard, and the reason it matters at a few thousand rows is in the [earlier build](https://magesheet.com/blog/ecommerce-realtime-inventory-dashboard).

## Pitfalls

*   **Cases and singles in the same column.** You buy in cases of twelve and sell singles, so a movement of `3` means nothing without its pack size. Convert at the edge and refuse what does not convert: `toEaches(2.5, 12)` is thirty units, `toEaches(0.5, 5)` is two and a half units and has to throw rather than round. A rounded movement is a permanent, invisible discrepancy.
    
*   **Pre-assembled kits are components too.** If somebody builds forty sets on Friday and puts them on a shelf, those forty are now stock in their own right and their parts are gone. Availability becomes assembled-on-hand plus what is still buildable, and the assembly step has to decrement the components the moment it happens, not when the set is sold.
    
*   **Nested kits.** A bundle containing a bundle needs the recipe flattened to real parts before any of this runs, and a recipe that contains itself needs to fail loudly instead of recursing until the script dies.
    
*   **Reorder points belong to parts.** A kit has no lead time and nothing to reorder; its body does. Setting a reorder point on the kit row produces an alert nobody can act on.
    
*   **Negative free stock is a signal, not a rounding problem.** `freeStock` returns the negative rather than clamping it, because minus two bodies means two orders are already promising stock that is not there, and the sooner that is on a screen the cheaper it is.
    
*   **The stock take has to count parts.** Counting assembled sets and loose parts into the same column is how the discrepancy appears the following week with no way to tell which of the two was miscounted.
    
*   **A few thousand rows and a per-row write.** Recomputing every kit on every edit puts a read and a write inside a loop, which is the shape that meets the [six-minute limit](https://magesheet.com/blog/apps-script-6-minute-limit) first. Read once, compute in memory, write once.
    

## The one thing worth taking away

A quantity column can only describe something that occupies space. A kit occupies none: its number is a claim about parts, derived from the scarcest one, and two kits that share a part were never two independent numbers. Publish per-kit availability by all means, but compute it from free component stock every time, and hold the claim at component level, or the sum of your honest numbers will keep offering stock that does not exist.

The full template this builds on, with the column set, the formula cheat-sheet and the point where a static sheet stops being cheaper than a build, is on the [MageSheet blog](https://magesheet.com/blog/google-sheets-inventory-template).