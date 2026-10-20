---
title: "Apps Script Is Free. Your Triggers Share 90 Minutes a Day."
seoTitle: "Apps Script Trigger Runtime: Track the 90-Minute Quota"
seoDescription: "Apps Script is free, but every trigger you own shares 90 minutes a day. Log each run to one sheet and get a warning at 70%."
datePublished: 2026-10-20T08:20:36.764Z
cuid: cmvgeyb5l000008ig1ci8hzmj
slug: apps-script-trigger-runtime-budget
canonical: https://magesheet.com/blog/google-apps-script-pricing
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/aab76c0f-2af8-45e7-85df-99907c977215.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/bd3a5178-e1f3-4b21-a5f4-09295096a7d6.jpg
tags: javascript, automation, google-sheets, google-apps-script, magesheet

---

A free Gmail account can run Apps Script on a schedule without paying Google anything. There is no licence, no seat and no per-run charge, and my [guide to what Apps Script costs](https://magesheet.com/blog/google-apps-script-pricing) walks through why. The price is paid somewhere else, in a daily allowance of trigger time. On a consumer account, every trigger you own shares 90 minutes a day.

Ninety minutes sounds like plenty until you multiply. Six scripts that each run every 15 minutes make 576 runs a day. If each run takes 10 seconds, that is 96 minutes, and the last runs of the day fail. Nothing in the editor shows that total while you build, and the first sign is usually a failure email about a script that worked fine yesterday.

This post is the code I use to see that number before Google enforces it: a wrapper that logs every trigger run to one shared sheet, a function that adds up the last 24 hours, and a daily report that warns at 70% of the budget.

## The quota lines that matter here

From Google's Apps Script quota page (last updated 3 September 2026):

| Quota | Consumer (gmail.com) | Google Workspace |
| --- | --- | --- |
| Triggers total runtime | 90 min / day | 6 hr / day |
| Script runtime | 6 min / execution | 6 min / execution |
| Triggers | 20 / user / script | 20 / user / script |
| Email recipients per day | 100 | 1,500 |
| URL Fetch calls | 20,000 / day | 100,000 / day |

Two sentences on the same page decide how the budget has to be measured. The first is "Quotas are per user and reset 24 hours after the first request." Per user means the 90 minutes are shared by every project you own, not given to each one. A project that logs only itself can report a comfortable number while another project spends the rest. The second explains the error you see when the budget runs out, "Service using too much computer time for one day", which the page says most commonly occurs for scripts that run on a trigger.

So the log has to live outside any single project, and the window to watch is a rolling 24 hours, not a calendar day. The six-minute limit per execution is a different problem with its own fix, covered in [running long jobs past the Apps Script 6-minute limit](https://magesheet.com/blog/apps-script-6-minute-limit). This post is about the daily total.

## Log every trigger run to one shared sheet

Create one spreadsheet with two tabs, `Trigger log` and `Quota report`. Put `id, started, project, handler, status, seconds` in the first row of `Trigger log`. Then add this file to every project that has triggers, changing only `PROJECT`:

```javascript
// One log spreadsheet, shared by every project that uses this file.
const LOG_ID = 'PASTE_LOG_SPREADSHEET_ID';
const LOG_TAB = 'Trigger log';  // A:F = id, started, project,
                                //       handler, status, seconds
const PROJECT = 'order-sync';   // a short name for this project

function logged_(handler, fn) {
  const sh = SpreadsheetApp.openById(LOG_ID)
    .getSheetByName(LOG_TAB);
  const runId = Utilities.getUuid();
  sh.appendRow(
    [runId, new Date(), PROJECT, handler, 'RUNNING', '']
  );
  const t0 = Date.now();
  let status = 'OK';
  try {
    return fn();
  } catch (err) {
    status = 'ERROR';
    throw err;
  } finally {
    const secs = Math.round((Date.now() - t0) / 1000);
    const cell = sh.getRange('A:A')
      .createTextFinder(runId)
      .matchEntireCell(true)
      .findNext();
    if (cell) {
      sh.getRange(cell.getRow(), 5, 1, 2)
        .setValues([[status, secs]]);
    }
  }
}

// The function the time-driven trigger calls.
function syncOrders() {
  return logged_('syncOrders', syncOrdersWork_);
}
```

The trigger now points at `syncOrders`, and the real work moves to `syncOrdersWork_`. Each run writes a `RUNNING` row before the work starts, then fills in `OK` or `ERROR` and the elapsed seconds when it ends. A failed run is logged as `ERROR` and the error is thrown again, so Apps Script's own failure emails still arrive.

Two details in that block are there because the simpler version is wrong.

The row is found again by its id, not by `getLastRow()`. Reading the last row right after `appendRow` works while one project writes to the log. Once two projects share it, a second run can append its own row while the first is still working, and the first run then writes its status into the second run's row. `LockService.getScriptLock()` does not help, because a script lock only covers runs of the same project. A UUID in column A and a `TextFinder` with `matchEntireCell(true)` find the right row however many rows were added in between. Without `matchEntireCell`, a search for one id can stop at a longer cell that contains it.

The `RUNNING` row is written first, before the work, so that a run that never finishes still leaves a trace. When Apps Script stops an execution at the six-minute limit, I do not count on the `finally` block running, and the row stays `RUNNING`. The summary below treats those rows as a signal of their own.

## Add up the last 24 hours

The summary is a plain function with no Apps Script services in it, so it can be tested outside the editor:

```javascript
const DAY_MS = 24 * 60 * 60 * 1000;
const MAX_RUN_SECS = 6 * 60;

// rows: [id, started, project, handler, status, seconds]
function summarize_(rows, now, budgetMin) {
  const by = Object.create(null);
  let total = 0;
  let stuck = 0;
  let bad = 0;
  rows.forEach(r => {
    const started = new Date(r[1]).getTime();
    if (!(started > now - DAY_MS && started <= now)) return;
    let secs;
    if (r[4] === 'RUNNING') {
      const age = Math.round((now - started) / 1000);
      if (age > MAX_RUN_SECS) stuck++;
      secs = Math.min(age, MAX_RUN_SECS);
    } else {
      secs = r[5] === '' ? NaN : Number(r[5]);
    }
    if (!(secs >= 0)) {
      bad++;
      return;
    }
    const key = r[2] + ' / ' + r[3];
    if (!by[key]) by[key] = { key: key, runs: 0, secs: 0 };
    by[key].runs++;
    by[key].secs += secs;
    total += secs;
  });
  const budget = budgetMin * 60;
  return {
    total: total,
    budget: budget,
    pct: Math.round((total / budget) * 100),
    stuck: stuck,
    bad: bad,
    rows: Object.values(by).sort((a, b) => b.secs - a.secs),
  };
}
```

What it decides, row by row:

*   A row counts only if it started within the last 24 hours. A start date that does not parse, or one in the future, is skipped.
    
*   A finished row, `OK` or `ERROR`, counts its logged seconds. Failed runs still ran, so they count.
    
*   A `RUNNING` row older than six minutes cannot still be running. It counts as six minutes, the most a single execution can use, and is reported as stuck.
    
*   A `RUNNING` row younger than that counts the time elapsed so far.
    
*   A finished row whose seconds cell is empty or not a number is counted as unreadable, not as zero. `Number('')` is `0` in JavaScript, and a silent zero would make the budget look healthier than it is.
    
*   Totals are grouped by project and handler together, because two projects often both have a function called `sync`.
    

/Users/yusufpangal/Desktop/MageSheet dünyası/magesheet-publisher/diagrams/apps-script-trigger-runtime-budget.png

## A daily report that warns at 70%

The last piece runs once a day on its own time-driven trigger. It writes the totals to the `Quota report` tab and emails you when the budget is at least 70% used or when any run is stuck:

```javascript
const BUDGET_MIN = 90;  // 360 on a Google Workspace account
const ALERT_PCT = 70;

// Run once a day from its own time-driven trigger.
function quotaReport() {
  const ss = SpreadsheetApp.openById(LOG_ID);
  const rows = ss.getSheetByName(LOG_TAB)
    .getDataRange().getValues().slice(1);
  const s = summarize_(rows, Date.now(), BUDGET_MIN);

  const out = ss.getSheetByName('Quota report');
  out.clearContents();
  out.getRange(1, 1, 1, 3)
    .setValues([['Project / handler', 'Runs', 'Minutes']]);
  if (s.rows.length) {
    out.getRange(2, 1, s.rows.length, 3).setValues(
      s.rows.map(r => [r.key, r.runs, +(r.secs / 60).toFixed(1)])
    );
  }
  const note = s.pct + '% of ' + BUDGET_MIN +
    ' min in the last 24 h. Stuck in RUNNING: ' + s.stuck +
    '. Unreadable rows: ' + s.bad +
    '. Email recipients left today: ' +
    MailApp.getRemainingDailyQuota() + '.';
  out.getRange('E1').setValue(note);

  if (s.pct >= ALERT_PCT || s.stuck > 0) {
    MailApp.sendEmail(
      Session.getEffectiveUser().getEmail(),
      'Apps Script trigger budget at ' + s.pct + '%',
      note
    );
  }
}
```

The report puts one line in `E1`, for example: `11% of 90 min in the last 24 h. Stuck in RUNNING: 0. Unreadable rows: 0. Email recipients left today: 88.` The last figure comes from `MailApp.getRemainingDailyQuota()`, which reads the email allowance of the account running the report. It is the other daily limit that is easy to forget until it is gone.

Set `BUDGET_MIN` to 360 if the account is on Google Workspace. The script cannot tell the account type on its own, and a wrong guess in the generous direction defeats the purpose. Change `ALERT_PCT` if you want the warning earlier.

## What I tested, and what I did not

The three blocks above ran in Node against mocked `SpreadsheetApp`, `Utilities`, `MailApp` and `Session` objects: 23 checks, covering the 24-hour boundary, stuck and young `RUNNING` rows, empty seconds, two projects with the same handler name, a second project appending a row while the first is still working, and an id that is the start of another id. I then broke the code on purpose in twelve places, for example by putting `getLastRow()` back in place of the id lookup, removing the six-minute cap, or dropping the stuck-run alert. Ten of the twelve changes failed at least one check. The other two did not change what the code does.

What a mock cannot show, and I have not measured:

*   How closely the logged seconds match what Google charges. The timer starts inside your handler, so the time Apps Script spends starting the execution is not in the log. Read the total as a lower bound.
    
*   Whether simple triggers, such as a plain `onEdit(e)`, count toward the same 90 minutes. The wrapper works for them too, but I would not read their total as Google's own figure.
    
*   Whether two `appendRow` calls from different projects in the same instant can collide. Google's reference for `appendRow` says nothing about concurrent writers.
    

## Pitfalls

**The report spends budget too.** `quotaReport` is a trigger like any other, and reading a long log takes longer every week. Delete rows older than a week from `Trigger log` now and then.

**One unwrapped project is invisible time.** The total is only right when every project that owns a trigger writes to the same log. A project you forgot still spends from the same 90 minutes.

**Triggers belong to the person who created them.** An installable trigger runs under the account that created it, so its time comes out of that person's budget, not the spreadsheet owner's. If a colleague set up half the triggers, wrap their projects too and point them at a log both of you can edit.

**Frequency moves the total more than speed.** In the example at the top, moving the six schedules from every 15 minutes to every 30 halves the runs and the minutes. Making each run 20% faster saves 20%.

## The takeaway

Apps Script costs nothing to run. The limit you meet instead is a daily allowance shared by every project you own, and no single project can see it. One log sheet that all of them write to turns it into a number you can watch. The full cost picture, including the three cases where Apps Script does lead to a bill, is in the [Apps Script pricing guide on the MageSheet blog](https://magesheet.com/blog/google-apps-script-pricing).

Built by the MageSheet team.