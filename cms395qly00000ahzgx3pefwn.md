---
title: "Apps Script 6-Minute Limit: Run Long Jobs Without Cloud Functions"
seoTitle: "Apps Script 6-Minute Limit: 3 Patterns for Long Jobs"
seoDescription: "Three Apps Script patterns — self-rescheduling triggers, chunked state, and parallel fan-out — run hour-long jobs without Cloud Functions."
datePublished: 2026-07-27T13:18:18.390Z
cuid: cms395qly00000ahzgx3pefwn
slug: apps-script-6-minute-limit
canonical: https://magesheet.com/blog/apps-script-6-minute-limit
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/5826f7d3-cfcd-420f-b2ee-665b2508d1d2.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/46ea0016-b5cb-4ee4-820c-48449bdfa03c.jpg
tags: automation, google-sheets, google-apps-script, magesheet

---

There comes a point in every Apps Script project where the same error fires three days in a row and quietly ruins your weekend:

> Exception: Service Spreadsheets timed out while accessing document... Exception: Exceeded maximum execution time

You wrote a script that was supposed to import 50,000 Magento orders into a Sheet. It worked on the first 8,000 rows, then died at the six-minute mark with no rollback and no progress checkpoint. Half your data is in, half your sheet is locked, and your two options are (a) rewrite as a Cloud Function or (b) pray.

The truth is that you almost never need to leave Apps Script for long-running work. The 6-minute limit is a hard cap on **a single invocation**, not on a job. With three patterns — **self-rescheduling triggers**, **chunked state with PropertiesService**, and **parallel fan-out with LockService** — you can run jobs that take hours, survive crashes, and resume from where they left off. All without touching Google Cloud.

## Why the 6-Minute Limit Exists (And the Numbers You Need)

Google enforces this limit because Apps Script runs on shared multi-tenant infrastructure. A runaway loop with no upper bound would starve the platform; the timeout is a circuit breaker, not a punishment.

The exact numbers, verified against the [Google Workspace quotas page](https://developers.google.com/apps-script/guides/services/quotas) in 2026:

| Limit | Free Gmail | Workspace |
| --- | --- | --- |
| Per-execution timeout | 6 min | 6 min (paying does **not** raise this) |
| Daily total trigger runtime | 90 min | 6 hours |
| Simultaneous executions | 30 | 30 |
| `doGet` / `doPost` timeout | 30 sec | 30 sec |

The 6-minute limit is **per invocation**. The 90-minute or 6-hour cap is the **daily total** of all your trigger-driven runs combined. Web app entry points die at 30 seconds — webhook handlers must be fast or hand work off to a queue.

Stack the patterns below, and you can run 14 self-rescheduled 5-minute chunks on a free account, or 60+ on Workspace, all within a single day.

## The Three Patterns That Solve 95% of Long Jobs

1.  **Self-rescheduling trigger** — the workhorse. Process what you can, save a cursor, reschedule.
    
2.  **Chunked state with PropertiesService** — when you know the total size and want deterministic progress.
    
3.  **Parallel fan-out with LockService** — when chunks are independent and can run side by side.
    

Pick the simplest that fits.  
  

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/b6fafdec-f3d8-47b3-a82c-d01982ddda3b.jpg align="center")

## Pattern 1: The Self-Rescheduling Trigger

This is the 80% pattern. Process within 5 minutes (1-minute safety margin), save progress, and create a one-shot trigger to resume.

```javascript
const MAX_RUNTIME_MS = 5 * 60 * 1000; // 5 min, with 1 min safety
const PROP_KEY = 'IMPORT_CURSOR';

function processOrders() {
  const start = Date.now();
  const props = PropertiesService.getScriptProperties();
  let cursor = parseInt(
    props.getProperty(PROP_KEY) || '0', 10
  );

  const sheet =
    SpreadsheetApp.getActive().getSheetByName('Orders');
  const totalOrders = getTotalOrderCount();

  while (cursor < totalOrders) {
    if (Date.now() - start > MAX_RUNTIME_MS) {
      props.setProperty(PROP_KEY, String(cursor));
      scheduleSelf('processOrders');
      return;
    }

    const batch = fetchOrderBatch(cursor, 100);
    appendRowsToSheet(sheet, batch);
    cursor += batch.length;
  }

  // Done!
  props.deleteProperty(PROP_KEY);
  cleanupTriggers('processOrders');
  Logger.log('Import complete: ' + cursor + ' orders');
}

function scheduleSelf(handler) {
  cleanupTriggers(handler);
  ScriptApp.newTrigger(handler)
    .timeBased()
    .after(60 * 1000)
    .create();
}

function cleanupTriggers(handler) {
  ScriptApp.getProjectTriggers()
    .filter(t => t.getHandlerFunction() === handler)
    .forEach(t => ScriptApp.deleteTrigger(t));
}
```

Two non-obvious points: **leave a safety margin** (5 minutes, not 6), because the timer does not include the time between the last check and the actual exit. And **always clean up old triggers** before creating a new one — orphaned schedules hit the 20-trigger project cap.

This single pattern can move a million rows. Slower than Cloud Run, but $0 and zero ops surface.

## Pattern 2: Chunked State for Deterministic Progress

If your job has a known total — "rebuild dashboards for 200 stores" — you want progress visibility. Store a JSON state in PropertiesService and surface it in a status sheet.

```javascript
function rebuildAllDashboards() {
  const start = Date.now();
  const props = PropertiesService.getScriptProperties();
  const state = JSON.parse(
    props.getProperty('REBUILD_STATE') || '{}'
  );

  if (!state.queue) {
    state.queue = getStoreList().map(s => s.id);
    state.completed = [];
    state.failed = [];
  }

  while (state.queue.length > 0) {
    if (Date.now() - start > 5 * 60 * 1000) break;

    const storeId = state.queue.shift();
    try {
      rebuildOneDashboard(storeId);
      state.completed.push(storeId);
    } catch (err) {
      state.failed.push({
        storeId, error: err.message
      });
    }

    props.setProperty(
      'REBUILD_STATE', JSON.stringify(state)
    );
    writeStatusToSheet(state);
  }

  if (state.queue.length > 0) {
    scheduleSelf('rebuildAllDashboards');
  } else {
    props.deleteProperty('REBUILD_STATE');
  }
}
```

The queue is persisted **every iteration**, so a crash never loses more than one item. Failures go into `state.failed` and the script moves on.

## Pattern 3: Parallel Fan-Out with LockService

When chunks are independent — enriching 10,000 product descriptions with AI — running them serially is wasteful. Apps Script supports **30 simultaneous executions**. Split the work.

```javascript
function fanOutEnrichment() {
  const totalProducts = 10000;
  const numWorkers = 10;
  const chunkSize =
    Math.ceil(totalProducts / numWorkers);

  for (let i = 0; i < numWorkers; i++) {
    const start = i * chunkSize;
    const end =
      Math.min(start + chunkSize, totalProducts);
    PropertiesService.getScriptProperties()
      .setProperty(
        `WORKER_${i}_RANGE`,
        JSON.stringify({ start, end })
      );

    ScriptApp.newTrigger(`worker${i}`)
      .timeBased()
      .after(i * 1000)
      .create();
  }
}

function workerGeneric(workerIndex) {
  const lock = LockService.getScriptLock();
  const range = JSON.parse(
    PropertiesService.getScriptProperties()
      .getProperty(`WORKER_${workerIndex}_RANGE`)
  );

  for (let i = range.start; i < range.end; i++) {
    const enriched = enrichProduct(i);

    lock.waitLock(30000);
    try {
      writeEnrichedRow(i, enriched);
    } finally {
      lock.releaseLock();
    }
  }
}
```

LockService serializes writes while letting the slow AI calls run in parallel. This moves a 6-hour serial job down to about 40 minutes.

## The Foot-Guns Nobody Warns You About

**Trigger orphaning.** If your script crashes before `cleanupTriggers()`, you accumulate triggers forever. Wrap the whole job in try/catch and clean up in `finally`.

**PropertiesService write contention.** Two simultaneous writers to the same key silently overwrite each other. Use LockService for parallel writes, or shard keys per worker.

**Web app vs trigger timeout confusion.** `doPost` dies at 30 seconds. If your webhook needs 4 minutes of work, the handler must enqueue (write to a Sheet, return 200) and a separate trigger drains the queue.

**Quota stacking.** Long jobs also burn UrlFetchApp quota, Gmail send quota, and daily trigger runtime. Hitting any one silently kills the rest.

**Session state lost between invocations.** If your script holds a logged-in session, it does not survive across reschedules. Store the session token in PropertiesService.

## When to Stop Fighting and Move to Cloud Run

Move out of Apps Script when:

*   A single job genuinely needs more than 30 minutes of sustained CPU
    
*   You need more than 30 concurrent workers
    
*   You have a job DAG with dependencies requiring manual trigger coordination
    
*   Your team already has a Go or Node.js skillset and the maintenance cost outweighs the simplicity
    

Until you hit one of those, the patterns above scale further than people assume. We have seen Magento-to-Sheets pipelines move 200,000 rows nightly on a free Workspace account using pattern 1.

The production versions of these patterns — with retry logic, dead-letter queues, and alerting — are on the [MageSheet blog](https://magesheet.com/blog/apps-script-6-minute-limit).

* * *

*Built by the MageSheet team.*