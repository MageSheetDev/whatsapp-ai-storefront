---
title: "The Sheet Outlives AppSheet. Check That Its References Do."
seoTitle: "AppSheet vs Power Apps: Audit Your Data Before You Leave"
seoDescription: "AppSheet keeps your data in a Sheet you own. An Apps Script audit checks that refs, lists and photo paths still resolve without the app."
datePublished: 2026-10-06T15:57:27.241Z
cuid: cmuwv3vvw000006nqapdp10g5
slug: appsheet-vs-power-apps
canonical: https://magesheet.com/blog/appsheet-vs-power-apps
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/862fb5c8-4424-440a-80c4-f648b1a4d932.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/a2b5b6a0-e0a4-4a9f-9f40-a3e6bf493ec2.jpg
tags: google-sheets, low-code, google-apps-script, appsheet, magesheet

---

The usual argument for AppSheet over Power Apps is about where the data lives. AppSheet keeps its records in a Google Sheet you own, so if you cancel the subscription the app disappears and the spreadsheet stays. Power Apps keeps them in Dataverse, which is a proper relational database but not a file you can open.

That argument is correct as far as it goes. The rows survive. What I wanted to know before relying on it was whether the rows still mean anything once the app that wrote them is gone, because AppSheet writes several kinds of value into the Sheet that only AppSheet knows how to read.

So I wrote an audit that answers that question with counts, and runs in Apps Script on the same spreadsheet.

## What AppSheet actually writes into the cells

Open the Sheet behind a typical field-service app and three column types look different from what you see in the app.

| Column type in the app | What the cell holds | What breaks without the app |
| --- | --- | --- |
| Ref | The key of the row it points to, often a random string from `UNIQUEID()` such as `3f9a1c7e` | Nothing checks that the key still exists |
| EnumList of Ref | Several keys joined by space, comma, space: `P1 , P2 , P7` | A plain `split(',')` reads it wrong |
| Image or File | A path relative to the app's folder in Drive, such as `Jobs_Images/3f9a1c7e.Photo.jpg` | The path means nothing unless you know which folder it is relative to |

None of this is a defect. A Ref column storing a key is how every database stores a relationship. The difference is enforcement. Dataverse refuses to delete a customer that jobs still point to. A Google Sheet has no such rule, and AppSheet only removes child rows automatically when the relationship is marked "Is a part of". Delete a customer through the app without that setting, or delete a row directly in the Sheet, and the jobs that referenced it keep a key that resolves to nothing.

Inside the app this is easy to miss, because nobody browses jobs looking for a customer that is no longer there. Outside the app, in an export, a pivot table or a replacement build, it becomes a row whose customer cannot be found. That is the thing to measure before you cancel anything.

## Index the keys, then look for references that miss

The first block builds a key index per table and reports two problems on the key column itself: blank keys and duplicate keys. Then it checks every reference against that index, splitting list cells on the separator AppSheet actually uses.

```javascript
var LIST_SEP = ' , ';

function cell(value) {
  return String(value == null ? '' : value).trim();
}

function isBlankRow(row) {
  return row.every(function (v) { return cell(v) === ''; });
}

function splitList(value) {
  var s = cell(value);
  if (!s) return [];
  return s.split(LIST_SEP)
    .map(function (v) { return v.trim(); })
    .filter(function (v) { return v !== ''; });
}

function indexKeys(rows, keyCol) {
  var keys = Object.create(null);
  var dupes = [];
  var blanks = [];
  rows.forEach(function (row, i) {
    if (isBlankRow(row)) return;
    var k = cell(row[keyCol]);
    if (!k) blanks.push('row ' + (i + 2));
    else if (keys[k]) dupes.push(k);
    else keys[k] = true;
  });
  return { keys: keys, dupes: dupes, blanks: blanks };
}

function findOrphans(rows, refCol, index, isList) {
  var orphans = [];
  rows.forEach(function (row, i) {
    if (isBlankRow(row)) return;
    var values = isList
      ? splitList(row[refCol])
      : [cell(row[refCol])];
    values.forEach(function (v) {
      if (v && !index.keys[v]) {
        orphans.push('row ' + (i + 2) + ': ' + v);
      }
    });
  });
  return orphans;
}
```

Three details in there carry more weight than they look.

`Object.create(null)` instead of `{}`. A plain object inherits `toString`, `constructor` and friends, so a reference cell containing `toString` would be found in the index and never reported. It is an unlikely key, but an audit is exactly the place where "unlikely" should not be a pass.

`cell()` stringifies and trims both sides. A numeric key like `1001` comes back from `getValues()` as a number, and the reference column may hold the same value as a number or as text. Comparing strings makes both match.

`' , '` rather than `','`. AppSheet joins list items with a comma surrounded by spaces precisely so that a value containing a comma survives. `'Smith, John , Doe, Jane'.split(',')` gives four pieces; splitting on `' , '` gives the two names that were stored. A list someone typed by hand as `P1,P2` stays a single value under this rule and gets reported, which is the right outcome: a human needs to look at it.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/30bed9c4-41fc-4868-992a-fe3d77a6568b.png align="center")

## Resolve the file paths against Drive

Photos are where an exit hurts most, because a job record without its signature or site photo is often worthless. An Image column holds either a full URL, which is self-contained, or a relative path, which only works if you know the folder it starts from. In AppSheet that is the app's default folder in Drive, by default somewhere under `appsheet/data/`. Copy its ID from the folder's URL.

```javascript
function classifyFileCell(value) {
  var s = cell(value);
  if (!s) return { kind: 'empty' };
  if (/^https?:\/\//i.test(s)) return { kind: 'url' };
  var parts = s.split('/').filter(function (p) {
    return p !== '';
  });
  return { kind: 'relative', parts: parts };
}

function findByPath(root, parts) {
  var folder = root;
  for (var i = 0; i < parts.length - 1; i++) {
    var sub = folder.getFoldersByName(parts[i]);
    if (!sub.hasNext()) return null;
    folder = sub.next();
  }
  var files = folder.getFilesByName(parts[parts.length - 1]);
  return files.hasNext() ? files.next() : null;
}

function findMissingFiles(rows, fileCol, root) {
  var missing = [];
  rows.forEach(function (row, i) {
    var c = classifyFileCell(row[fileCol]);
    if (c.kind !== 'relative') return;
    if (!findByPath(root, c.parts)) {
      missing.push('row ' + (i + 2) + ': ' + c.parts.join('/'));
    }
  });
  return missing;
}
```

`findByPath` walks the path one folder at a time with `getFoldersByName` and finishes with `getFilesByName`. A path that points at a folder nobody created, or at a file that was moved out of the app folder to tidy up Drive, comes back as `null` and lands in the report with its row number.

## The runner and the report

The last block describes the app's tables once, reads each tab by header name, runs the checks and writes one line per check into an `Exit audit` tab.

```javascript
var APP_FOLDER_ID = 'PASTE_THE_APP_FOLDER_ID';
var TABLES = {
  Customers: { key: 'CustomerID' },
  Parts: { key: 'PartID' },
  Jobs: {
    key: 'JobID',
    refs: [{ col: 'Customer', table: 'Customers' }],
    lists: [{ col: 'Parts used', table: 'Parts' }],
    files: ['Photo']
  }
};

function readTable(ss, name) {
  var sheet = ss.getSheetByName(name);
  if (!sheet) throw new Error('Missing tab: ' + name);
  var values = sheet.getDataRange().getValues();
  var head = values.shift().map(cell);
  return { name: name, head: head, rows: values };
}

function col(table, name) {
  var i = table.head.indexOf(name);
  if (i === -1) {
    throw new Error(table.name + ' has no column "' + name + '"');
  }
  return i;
}

function line(table, check, column, hits) {
  return [
    table, check, column, hits.length,
    hits.length ? 'FAIL' : 'PASS',
    hits.slice(0, 5).join(' | ')
  ];
}

function auditExit() {
  var ss = SpreadsheetApp.getActive();
  var root = DriveApp.getFolderById(APP_FOLDER_ID);
  var data = {};
  var index = {};
  var out = [];

  Object.keys(TABLES).forEach(function (name) {
    var t = readTable(ss, name);
    var keyCol = col(t, TABLES[name].key);
    var k = indexKeys(t.rows, keyCol);
    data[name] = t;
    index[name] = k;
    out.push(line(name, 'blank key', TABLES[name].key, k.blanks));
    out.push(line(name, 'duplicate key', TABLES[name].key, k.dupes));
  });

  Object.keys(TABLES).forEach(function (name) {
    var t = data[name];
    var spec = TABLES[name];
    (spec.refs || []).forEach(function (r) {
      var hits = findOrphans(t.rows, col(t, r.col), index[r.table]);
      out.push(line(name, 'orphan ref', r.col, hits));
    });
    (spec.lists || []).forEach(function (r) {
      var hits = findOrphans(
        t.rows, col(t, r.col), index[r.table], true);
      out.push(line(name, 'orphan list item', r.col, hits));
    });
    (spec.files || []).forEach(function (f) {
      var hits = findMissingFiles(t.rows, col(t, f), root);
      out.push(line(name, 'missing file', f, hits));
    });
  });

  var report = ss.getSheetByName('Exit audit')
    || ss.insertSheet('Exit audit');
  report.clearContents();
  report.getRange(1, 1, 1, 6).setValues([[
    'Table', 'Check', 'Column', 'Count', 'Result', 'First hits'
  ]]);
  report.getRange(2, 1, out.length, 6).setValues(out);
  return out;
}
```

Keys are indexed for every table before any reference is checked, so the order of `TABLES` does not matter. Each line in the report carries the count and the first five hits with row numbers, which is enough to open the tab and see whether you are looking at three stray test rows or a pattern.

I tested the pure functions in Node with 40 assertions, including a mocked `SpreadsheetApp` and a mocked Drive folder tree for the full `auditExit()` run. The real Drive and Sheets calls have to be tried on your own copy.

Reading the result is simple. All PASS means the claim holds for your app: the Sheet carries its own meaning and an export, a pivot table or a rebuild will see what the app saw. Orphan references mean decisions are owed before you leave, because the app has been hiding rows whose parent is gone. Missing files mean photos you believed you had are not where the records say they are. If your data model needs those relationships enforced rather than audited, that is the honest case for a relational store like Dataverse, and the audit tells you so in numbers rather than opinion.

## Pitfalls

**A renamed column passes silently.** Without the guard in `col()`, `indexOf` returns `-1`, `row[-1]` is `undefined`, every cell reads as empty, and the check reports zero orphans. The column that no longer exists gets a green PASS. Throwing on a missing header turns that into an error you cannot miss.

**Duplicate folder names in Drive.** Drive allows two folders with the same name in the same parent. `getFoldersByName` returns the first one it finds, so if someone made a second `Jobs_Images`, the audit can report files as missing that sit in the other copy. If the missing count looks wrong, check the app folder for duplicates before trusting it.

**Blank rows are not blank keys.** A fully empty row left in the middle of a tab would otherwise show up as a blank key on every run. `isBlankRow` skips it. A row with data but no key is still reported, because that row cannot be referenced by anything.

**Large photo tables.** Each relative path costs at least two Drive calls, one per folder level and one for the file. A small photo table finishes quickly; a large one can run into the execution time limit. Run the file check one table at a time, or split it with the continuation pattern in the [6-minute limit guide](https://magesheet.com/blog/apps-script-6-minute-limit).

**Size is a separate question.** This audit tells you whether the data is portable, not whether it fits a plan. Row and table counts per tab are covered in the [AppSheet plan limit audit](https://magesheet.com/blog/appsheet-pricing-2026).

The Sheet outliving the app is the strongest argument AppSheet has, and it is worth checking rather than assuming. One run, one tab of counts, and you know whether cancelling the subscription leaves you with records or with keys pointing at nothing.

The full cost and lock-in comparison of AppSheet and Power Apps, including the seat-count table and when an owned Apps Script build beats both, is on the [MageSheet blog](https://magesheet.com/blog/appsheet-vs-power-apps).