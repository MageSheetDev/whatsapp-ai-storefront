---
title: "Google Sheets to Custom App: Log Two Weeks of Edits Before You Ask for a Quote"
seoTitle: "Google Sheets Edit Log: Count Collisions Before a Quote"
seoDescription: "Log two weeks of edits in Apps Script, count overlapping writes by different people, then decide if Google Sheets needs a custom app."
datePublished: 2026-11-15T09:57:11.283Z
cuid: cmwhnunln000402p5bjl3f48c
slug: google-sheets-to-custom-app-log-two-weeks-of-edits-before-you-ask-for-a-quote
canonical: https://magesheet.com/blog/google-sheets-to-custom-app-cost
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/31d77336-c3bd-45ec-b1d8-9b0545870a1a.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/1c2a04dd-7e83-441a-a43e-9d6b196b0ef7.jpg
tags: javascript, automation, google-sheets, google-apps-script, magesheet

---

The MageSheet guide to [moving from Google Sheets to a custom app](https://magesheet.com/blog/google-sheets-to-custom-app-cost?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost) names four structural problems that justify the cost: collisions, visibility, accountability and reach. Two of them can be counted from inside the sheet. Collisions are "several people edit the same tab and overwrite each other". Accountability is "a balance changed, and nobody can say who changed it or why". The guide ends with six questions to answer before you ask anyone for a quote. None of them asks how often your own people actually collide.

So I wrote the counter. One installable `onEdit` handler appends a row per edit to a hidden tab, a pure function finds the edits where a different person wrote over an overlapping range within an hour, and a report function turns that into a short summary with a rule of thumb. I ran all three against mocks of the Apps Script services with 14 days of invented edits, and this is what the report tab looked like:

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/8bfa7aa9-279f-4a41-8881-3778a6733ffb.gif align="center")

The data is invented and the services are mocked, so the numbers show what the code does with a quiet two weeks, not what your sheet will show. Two collisions in 93 edits is a mild case on purpose: the point of the log is that the answer can be small.

## What the guide claims, and what a log can measure

The guide says problems 1 and 2 (collisions and visibility) can often be solved without leaving Google, with a [custom web interface on top of the sheet](https://magesheet.com/blog/custom-frontends-apps-script?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost) and [role checks in Apps Script](https://magesheet.com/blog/state-auth-apps-script?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost), and that a proper audit log and many concurrent users are where a database and a custom app start to earn their cost. That is a claim about frequency: how often, how many people. A log answers it.

| Problem in the guide | Can this log measure it? | What it gives you |
| --- | --- | --- |
| Collisions | Yes | Count of overlapping writes by different people, per week, and which ranges |
| Accountability | Partly | Share of edits with no known user, and who edited what, when |
| Visibility | No | Needs a look at the sharing settings, not at edits |
| Reach (phone, offline) | No | A question for the people who work in the field |

Two things before the code. First, a collision here is a proxy. Two people writing to overlapping cells within an hour might be a hand-over that both of them meant. The log counts the overlap, and you read the top ranges to decide which ones were accidents. Second, I did not repeat two measurements that already exist. How to count slow formulas and sharing settings is in [the formula audit post](https://dev.to/hayrullahkar/outgrown-google-sheets-count-the-slow-formulas-before-you-decide-4ceh), and the break-even arithmetic is in [the build vs buy post](https://dev.to/hayrullahkar/build-vs-buy-in-google-sheets-the-break-even-cell-that-said-400-months-4d3p).

## Block 1: one row per edit

The handler has to be an installable trigger, not the simple `onEdit`. Google's triggers guide says simple triggers "can't access other files because that would require authorization" (read 2026-11-15), and the `Session` reference says a user's email address is not available in any context that allows a script to run without that user's authorization, with the simple `onEdit(e)` as an example. An installable edit trigger, in the words of the installable triggers page, "runs when a user modifies a value in a spreadsheet", and "the installable version runs with the authorization of the user who created the trigger".

What the handler writes is decided by what the event object offers. The events page describes `e.user` as the active user "if available", and says `e.oldValue` is "Only available if the edited range is a single cell". So the log stores the range as A1 text plus its row and column counts, and the previous value only for a one-cell edit. A blank address is stored as `(unknown)`, not as an empty cell, so that the report can count it.

`installEditLog` is safe to run twice. It looks for triggers that call `logEdit` on this spreadsheet, keeps the first, deletes any extras, and creates one only if none exist. The `getTriggerSourceId()` comparison is there so that a trigger on another file in the same project is not mistaken for this one.

Two details in the handler come from tests, not from the docs. The previous value is cut at 200 characters, because a pasted paragraph in one cell should not make the log heavy. And a previous value that starts with `=`, `+`, `-` or `@` gets a leading apostrophe, because a log of old values that turns an old formula into a live formula is worse than no log.

```javascript
const LOG_SHEET = '_EditLog';
const LOG_HEADER = [
  'at', 'user', 'sheet', 'range', 'rows', 'cols', 'old',
];
const UNKNOWN = '(unknown)';

function getLog_(ss) {
  let sh = ss.getSheetByName(LOG_SHEET);
  if (!sh) {
    sh = ss.insertSheet(LOG_SHEET);
    sh.appendRow(LOG_HEADER);
    sh.hideSheet();
  }
  return sh;
}

function logEdit(e) {
  if (!e || !e.range) return;
  const sheet = e.range.getSheet();
  const name = sheet.getName();
  if (name === LOG_SHEET) return;
  const rows = e.range.getNumRows();
  const cols = e.range.getNumColumns();
  const mail = e.user && e.user.getEmail ? e.user.getEmail() : '';
  const who = String(mail || '').trim().toLowerCase();
  let old = '';
  if (rows * cols === 1 && e.oldValue !== undefined) {
    old = String(e.oldValue).slice(0, 200);
    if (/^[=+\-@]/.test(old)) old = "'" + old;   // keep it text
  }
  getLog_(sheet.getParent()).appendRow([
    new Date(), who || UNKNOWN, name,
    e.range.getA1Notation(), rows, cols, old,
  ]);
}

function installEditLog() {
  const ss = SpreadsheetApp.getActive();
  const mine = ScriptApp.getProjectTriggers().filter(t =>
    t.getHandlerFunction() === 'logEdit' &&
    t.getTriggerSourceId() === ss.getId());
  mine.slice(1).forEach(t => ScriptApp.deleteTrigger(t));
  if (!mine.length) {
    ScriptApp.newTrigger('logEdit')
      .forSpreadsheet(ss).onEdit().create();
  }
  getLog_(ss);
  return mine.length ? 'already installed' : 'installed';
}
```

Run `installEditLog` once from the editor, as the person who should own the trigger. From then on every manual edit adds a row. The handler ignores edits to `_EditLog` itself.

## Block 2: what counts as a collision

`findCollisions_` is a pure function: rows in, a result out, no Apps Script service inside. Each row is `{ at, user, sheet, a1 }`. An edit is a collision when an earlier edit on the same sheet meets four conditions: a different known user, an overlapping range, at most `windowMinutes` earlier, and both parsed cleanly.

The part that takes care is the range. Comparing `a1` strings for equality would miss almost everything: one person pastes a block, `B2:D9`, and the other fixes `C5`. The function parses each A1 string into column and row bounds, including whole columns such as `A:A` and whole rows such as `3:3`, and tests whether the boxes intersect. A neighbouring cell, `E9` against `B2:D9`, does not.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/b9a8500f-e609-444c-beac-480eb38e36a5.png align="center")

The rest are decisions I made, each with a test behind it:

- **Each later edit counts once.** It is matched to the most recent earlier edit that qualifies. Without that rule, A, B, A, B inside an hour would be counted as four pairs instead of three collisions.
- **Unknown users are not collisions.** If either side is `(unknown)`, the edit is counted in a separate `unclear` number. Two blank accounts are not evidence of two different people, and they are not evidence of one.
- **The window is inclusive.** An edit exactly 60 minutes later is inside a 60 minute window, and one millisecond later is outside. The input does not need to be sorted, and the function does not change it.
- **Rows it cannot read are skipped and counted.** A bad timestamp or a range string that is not A1 goes into `skipped`, so the report does not quietly drop them.

```javascript
function colNum_(s) {
  let n = 0;
  for (const ch of s) n = n * 26 + ch.charCodeAt(0) - 64;
  return n;
}

function cell_(s) {
  const m = /^\$?([A-Z]*)\$?(\d*)$/.exec(s);
  if (!m || !(m[1] || m[2])) return null;
  return { c: m[1] ? colNum_(m[1]) : 0, r: m[2] ? Number(m[2]) : 0 };
}

function span_(x, y) {
  if (!x || !y) return [x || y || 1, Infinity];
  return [Math.min(x, y), Math.max(x, y)];
}

function parseA1_(a1) {
  const p = String(a1).trim().toUpperCase().split(':');
  if (p.length > 2) return null;
  const a = cell_(p[0]);
  const b = cell_(p[p.length - 1]);
  if (!a || !b) return null;
  if (p.length === 1 && !(a.c && a.r)) return null;
  const [c1, c2] = span_(a.c, b.c);
  const [r1, r2] = span_(a.r, b.r);
  return { c1, c2, r1, r2 };
}

function overlap_(a, b) {
  return a.c1 <= b.c2 && b.c1 <= a.c2 && a.r1 <= b.r2 && b.r1 <= a.r2;
}

function findCollisions_(rows, windowMinutes) {
  if (typeof windowMinutes !== 'number' || !(windowMinutes >= 0)) {
    throw new Error('windowMinutes must be a number >= 0');
  }
  const win = windowMinutes * 60000;
  const ev = [];
  let skipped = 0;
  rows.forEach((r, i) => {
    const at = r.at == null ? NaN : new Date(r.at).getTime();
    const box = parseA1_(r.a1);
    if (isNaN(at) || !box) { skipped++; return; }
    const u = String(r.user == null ? '' : r.user).trim().toLowerCase();
    const user = u === UNKNOWN ? '' : u;
    ev.push({ at, i, box, user, sheet: r.sheet, a1: r.a1 });
  });
  ev.sort((x, y) => x.at - y.at || x.i - y.i);
  const collisions = [];
  let unclear = 0;
  for (let j = 0; j < ev.length; j++) {
    const b = ev[j];
    let hit = null;
    let vague = false;
    for (let k = j - 1; k >= 0; k--) {
      const a = ev[k];
      if (b.at - a.at > win) break;
      if (a.sheet !== b.sheet || !overlap_(a.box, b.box)) continue;
      if (!a.user || !b.user) { vague = true; continue; }
      if (a.user !== b.user) { hit = a; break; }
    }
    if (hit) {
      collisions.push({
        sheet: b.sheet, a1: b.a1, over: hit.a1, by: b.user,
        from: hit.user, at: b.at,
        minutes: Math.round((b.at - hit.at) / 6000) / 10,
      });
    } else if (vague) {
      unclear++;
    }
  }
  return { collisions, unclear, skipped };
}
```

## Block 3: the report, and a rule of thumb that is mine

`summarize_` is also pure. It calls `findCollisions_` and returns the numbers: edits, distinct known editors, the share of edits with an unknown user, how many edits touched 100 cells or more, collisions per week, the top collided ranges and a verdict. `writeEditReport` reads the log, calls it and writes the result to a `_EditReport` tab, clearing the old one first.

Weeks start on Monday and are counted in UTC. That is a simplification: an edit at 23:30 on Sunday in Los Angeles lands in the next week in the report. I chose it because the same log then gives the same report on any machine, and I tested it under three time zones.

The verdict is the part you should change. It is my rule of thumb, not a fact about software projects, and the thresholds are constants you can edit:

| Collisions per week | My reading | Why |
| --- | --- | --- |
| Under 1 | Stay on Sheets | The log shows no sign of the collision problem, so the pain is somewhere else |
| 1 to under 5 | Try a custom UI with role checks first | One write path and per-person ranges can be built on the sheet |
| 5 or more | Ask for a custom app quote | Overlaps are frequent enough to bring a number to the quote conversation |
| 20% or more of edits with an unknown user | Fix identity first | The log cannot say who, so the collision count is not reliable either |

The order matters. The identity check comes first, because if one edit in five has no known user, the log cannot say who did what, and the collision count is not reliable either. A custom interface or app with sign-in fixes that. The sheet's own log, on that evidence, does not.

If the verdict says custom UI, the guide's two options are the way to try it: the [custom front end](https://magesheet.com/blog/custom-frontends-apps-script?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost) so nobody writes into the grid, and [role checks](https://magesheet.com/blog/state-auth-apps-script?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost) so each person writes only to their own range. If it says quote, you have a number to put on the table with the six answers the guide asks for.

```javascript
const WINDOW_MIN = 60;      // our choice: same cells within one hour
const BIG_EDIT_CELLS = 100; // our choice: a paste this big is "big"
const REPORT_SHEET = '_EditReport';

function weekOf_(ms) {
  const d = new Date(ms);
  const back = (d.getUTCDay() + 6) % 7;      // Monday start, in UTC
  return new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth(),
    d.getUTCDate() - back)).toISOString().slice(0, 10);
}

function verdict_(perWeek, unknownPct) {
  if (unknownPct >= 20) {
    return 'Identity gap: the log cannot say who. Fix that first.';
  }
  if (perWeek >= 5) return 'Ask for a custom app quote.';
  if (perWeek >= 1) return 'Try a custom UI with role checks first.';
  return 'Stay on Sheets: no evidence of collisions.';
}

function summarize_(rows, windowMinutes) {
  const found = findCollisions_(rows, windowMinutes);
  const known = {};
  let unknown = 0, big = 0, lo = Infinity, hi = -Infinity;
  rows.forEach(r => {
    const u = String(r.user == null ? '' : r.user).trim().toLowerCase();
    if (!u || u === UNKNOWN) unknown++; else known[u] = true;
    if (r.cells >= BIG_EDIT_CELLS) big++;
    const t = r.at == null ? NaN : new Date(r.at).getTime();
    if (!isNaN(t)) { lo = Math.min(lo, t); hi = Math.max(hi, t); }
  });
  const weeks = Math.max(1, (hi - lo) / 6048e5 || 0);
  const perWeek = Math.round(10 * found.collisions.length / weeks) / 10;
  const unknownPct = rows.length
    ? Math.round(1000 * unknown / rows.length) / 10 : 0;
  const byWeek = {};
  const byRange = {};
  found.collisions.forEach(c => {
    const w = weekOf_(c.at);
    byWeek[w] = (byWeek[w] || 0) + 1;
    const key = c.sheet + '!' + c.a1;
    byRange[key] = (byRange[key] || 0) + 1;
  });
  const top = Object.keys(byRange).sort((x, y) =>
    byRange[y] - byRange[x] || (x < y ? -1 : 1)).slice(0, 5)
    .map(k => [k, byRange[k]]);
  return {
    edits: rows.length, editors: Object.keys(known).sort(), unknown,
    unknownPct, big, collisions: found.collisions.length,
    unclear: found.unclear, skipped: found.skipped, perWeek,
    byWeek, top, verdict: verdict_(perWeek, unknownPct),
  };
}

function writeEditReport() {
  const ss = SpreadsheetApp.getActive();
  const log = ss.getSheetByName(LOG_SHEET);
  if (!log || log.getLastRow() < 2) throw new Error('no edits logged');
  const rows = log.getDataRange().getValues().slice(1).map(v => ({
    at: v[0], user: v[1], sheet: v[2], a1: v[3], cells: v[4] * v[5],
  }));
  const s = summarize_(rows, WINDOW_MIN);
  const out = [
    ['edits logged', s.edits],
    ['distinct known editors', s.editors.length],
    ['edits with unknown user (%)', s.unknownPct],
    ['edits of ' + BIG_EDIT_CELLS + '+ cells', s.big],
    ['collisions (' + WINDOW_MIN + ' min window)', s.collisions],
    ['unclear (an unknown user was involved)', s.unclear],
    ['collisions per week', s.perWeek],
  ];
  Object.keys(s.byWeek).sort().forEach(w =>
    out.push(['week of ' + w + ' (UTC)', s.byWeek[w]]));
  s.top.forEach(t => out.push(['top: ' + t[0], t[1]]));
  out.push(['my rule of thumb says', s.verdict]);
  const sh = ss.getSheetByName(REPORT_SHEET) ||
    ss.insertSheet(REPORT_SHEET);
  sh.clear();
  sh.getRange(1, 1, out.length, 2).setValues(out);
}
```

## Pitfalls

- **An overlap is not always a loss.** The log counts writes that overlapped, not values that were lost. For a single-cell edit you can read the old value and judge. For a pasted block there is no old value, and you have to ask the two people.
- **A big paste is one row.** A paste of 10,000 cells is a single log row with its row and column counts. The report shows how many edits were 100 cells or more, so one bulk import does not look like a hundred small edits, and so you can see who does the pasting.
- **The log tab is part of the file.** Anyone with edit access can open a hidden tab, and a log you can edit is a weak audit trail. For a sheet where accountability is the real problem, this is a measurement tool for two weeks, not the audit log the guide says a real system needs.
- **Two weeks can be a quiet two weeks.** Month-end close, a stocktake or a holiday can change the picture. If you can, include one of those periods.
- **Renaming a tab splits its history.** The log stores the sheet name as text, so `Orders` and `Orders 2026` are two sheets as far as the report is concerned.

## How this drifts after you set it up

- **The log grows by seven cells per edit.** Google's Drive limits page puts a spreadsheet at "Up to 20 million cells or 100MB for spreadsheets that are created in or converted to Google Sheets" (read 2026-11-15). Seven columns is at most 2,857,142 rows, if nothing else is in the file. At 1,000 edits a day that is about 7.8 years, by my arithmetic. Real files have other tabs, and the report reads the whole log each run. Delete or export the log once the two weeks are done.
- **The trigger belongs to one person.** The installable page says it runs under the account of the person who created it. When that person leaves the company and the account is removed, I would expect the logging to stop. I did not test what Google does in that case, so check the trigger list in the editor after any staff change.
- **Script and API edits are invisible.** The triggers guide says "Script executions and API requests don't cause triggers to run." A bulk import, another automation or a form-fed script can change thousands of cells and never reach the log.
- **Structure changes are a different trigger.** Per the installable page, an installable change trigger covers a user modifying "the structure of a spreadsheet itself", for example adding a sheet or removing a column. The edit trigger used here does not, so an added sheet or a removed column is not in the log. I would expect the same for inserted rows, but I did not check that.
- **Fast edits may be dropped.** Under the `onEdit(e)` heading, the triggers guide says "The `onEdit()` trigger only queues up to 2 trigger events." I did not test whether that applies to the installable version, or what happens when several people type at once. Treat the count as a floor.

## What I tested, and what I did not

The three blocks above are extracted from this article's text and run in Node against 113 tests: the handler and the installer against mocked `SpreadsheetApp`, `ScriptApp` and a fake clock, the A1 parser and range overlap on corners, neighbours, whole columns and rows, `findCollisions_` on window boundaries, unsorted input, unknown users and unreadable rows, and the summary, the thresholds and the report tab. The same file passes under `TZ=UTC`, `TZ=Asia/Istanbul` and `TZ=America/Los_Angeles`. I then broke the code on purpose 119 ways, and the tests caught all 119. The invented two-week run in the GIF comes from that same code. Two further mutations (a sort tie-break and an early `break`) change nothing a caller can see, so I did not count them.

I have not run any of this in a live Google account. The mocks do not behave like Google's services. In particular:

- I have not seen what `e.user.getEmail()` returns for a real editor with an installable trigger. Google's reference says the restrictions generally do not apply when the developer runs the script, or when the user is in the same Workspace domain, and I read that, I did not test it. The `(unknown)` branch exists because the docs allow a blank address.
- I used `getTriggerSourceId()` as the spreadsheet's id. The reference says only "Returns the id specific to the source", so I have not confirmed that it equals `getId()` for a bound trigger. If it does not, the installer would create a second trigger on every run, and you would see it in the trigger list.
- I have not run `appendRow` with a string that starts with `=` in Sheets. The apostrophe prefix is what I understand keeps it as text. The test only checks that the prefix is added.
- I did not check what `getA1Notation()` returns for an edit that covers a whole column. The parser handles `A:A` because the function is cheap to make safe, not because I saw Google return it.
- I have not timed the report on a log with hundreds of thousands of rows, and I did not check how close that gets to the 6 minute run limit.
- Quotes from Google's pages were read on 2026-11-15 through a page fetcher that summarises, not from a saved copy, so check the wording on the page before you rely on it.

## The takeaway

Before you ask for a custom app quote, spend two weeks measuring the two problems that can be measured. Log each edit with its author and range, count the overlapping writes by different people, and read the share of edits with no known author. Under one collision a week, the evidence for a build is thin. If it is five or more, or one edit in five has no author, you have a number to bring to the conversation. The full cost breakdown, the three options side by side and the six questions to answer before a quote are in the [MageSheet guide to moving from Google Sheets to a custom app](https://magesheet.com/blog/google-sheets-to-custom-app-cost?utm_source=hashnode&utm_medium=social&utm_campaign=google-sheets-to-custom-app-cost).