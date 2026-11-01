---
title: "Apps Script Error Handling: The Failed Invoice Your Ledger Skips Forever"
seoTitle: "Apps Script Retry Ledger: Failed Sends That Never Retry"
seoDescription: "An Apps Script idempotency ledger that marks a thrown send as failed and retries it, plus a watchdog that mails only when problems change."
datePublished: 2026-11-01T13:21:02.643Z
cuid: cmvxuyw21000202oq4bmkeywh
slug: apps-script-failed-retry-ledger-watchdog
canonical: https://magesheet.com/blog/apps-script-error-handling-monitoring
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/e63eba2e-d41d-4d06-a2e6-0d4c878f91da.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/ac4f5cbe-d612-4482-bc9b-44d4f083f0a8.jpg
tags: automation, monitoring, google-apps-script, google-workspace, magesheet

---

Our [guide to Apps Script error handling and alerts](https://magesheet.com/blog/apps-script-error-handling-monitoring) is built around one idea: the expensive failure is the one that throws nothing. It gives you a wrapper that re-throws, a run log on a sheet, a deduplicated alert, a heartbeat with a watchdog, and a `processOnce` ledger so that a job which dies halfway does not send the same invoice twice.

I ran that code unchanged in Node against fakes of `SpreadsheetApp`, `LockService`, `CacheService`, `PropertiesService` and `MailApp`. Two scenarios, two findings:

- **One failed send becomes an invoice that is never sent.** A daily job sends the invoice for order 1001 and `MailApp` throws "Service invoked too many times for one day". The wrapper logs it and one alert goes out. The next three daily runs skip order 1001, because its key is already in the ledger as `in_progress`. Every run is `ok`, the heartbeat is `ok`, and the watchdog has nothing to report. Invoices sent for that order: 0.
- **A weekend outage sends 124 alert emails.** One hourly job is broken from Friday 18:00 to Monday 08:00, and the watchdog runs on an hourly trigger. `alertOnce` mails on every run, because its dedup window (3,600 seconds) is the same as the job's interval. The watchdog mails on every run, because nothing remembers what it already reported. 48 of those emails land on Saturday, from a Gmail allowance of 100 recipients a day on a consumer account.\
  \

  ![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/7b877239-1c26-4f52-a640-963caaba0c07.gif align="center")

The layer the guide describes is the right one. The first finding matters most, because it is the failure the guide was written to prevent: silent, and found weeks later by a customer or an accountant.

## Why the ledger skips the invoice

`processOnce` asks one question before it runs anything: is this key anywhere in column B? If it is, it returns `{ skipped: true }`. Otherwise it appends the key as `in_progress`, runs `fn()`, and sets the row to `done`.

When `fn()` throws, nothing touches the row again. It stays `in_progress`, and on the next run the key is found and skipped.

The guide describes three states for that column: never attempted, finished, and "attempted but unknown", and says a row stuck at `in_progress` for two hours "is a manual-review item". That is the right call for a run the platform killed mid-send, because nobody knows whether the email went out. A thrown exception is different. `MailApp.sendEmail` threw, so the invoice did not go. The code records both cases the same way, and no function reads `in_progress` rows back, so the manual review never happens.

People who build the same thing on other platforms make the same split. In a [Make community thread about sending vouchers from Google Sheets](https://community.make.com/t/best-idempotent-make-scenario-for-shopify-forms-vouchers-via-google-sheets/112819), the original poster wrote: "We would prefer manual review over a possible duplicate whenever the delivery result is ambiguous." The word that matters is ambiguous. A send that threw is not.

## Step 1: a ledger that tells a failure from a crash

```javascript
// Ledger tab: A at | B key | C state | D attempts | E error
const MAX_ATTEMPTS = 3;
const STUCK_MIN = 30;

function ledgerSheet_() {
  return SpreadsheetApp.openById(LOG_SHEET_ID)
    .getSheetByName('ledger');
}

function processOnce(key, fn) {
  const claim = claimKey_(key);
  if (!claim.run) return { skipped: true, state: claim.state };
  try {
    const result = fn();
    markKey_(claim.row, 'done', claim.attempts, '');
    return { skipped: false, result };
  } catch (err) {
    const state = claim.attempts >= MAX_ATTEMPTS
      ? 'gave_up' : 'failed';
    markKey_(claim.row, state, claim.attempts,
      String(err.message || err).slice(0, 200));
    throw err;
  }
}

// Holds the lock only while it reads and claims the key,
// not while fn() talks to Gmail or a payment API.
function claimKey_(key) {
  const lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try {
    const ledger = ledgerSheet_();
    const last = ledger.getLastRow();
    const rows = last
      ? ledger.getRange(1, 2, last, 3).getValues() : [];
    const i = rows.findIndex(r => r[0] === key);
    if (i === -1) {
      ledger.appendRow([new Date(), key, 'in_progress', 1, '']);
      return { run: true, row: ledger.getLastRow(), attempts: 1 };
    }
    const state = rows[i][1];
    if (state !== 'failed') return { run: false, state };
    const attempts = Number(rows[i][2]) + 1;
    markKey_(i + 1, 'in_progress', attempts, '');
    return { run: true, row: i + 1, attempts };
  } finally {
    lock.releaseLock();
  }
}

function markKey_(row, state, attempts, error) {
  const ledger = ledgerSheet_();
  ledger.getRange(row, 1).setValue(new Date());
  ledger.getRange(row, 3, 1, 3)
    .setValues([[state, attempts, error]]);
}
```

Three changes:

- `failed` **is its own state.** A send that throws writes `failed`, the error and the attempt number, then re-throws so `runGuarded` still logs and alerts. The next run's `claimKey_` sees `failed` and runs it again. After the third failure the row becomes `gave_up` and is skipped from then on.
- `in_progress` **now means one thing:** a run that started and never wrote an outcome. That is what the six-minute limit leaves behind, since, as the guide points out, a platform kill skips `catch` and `finally`. Those rows are not retried.
- **The lock covers the claim, not the send.** The original holds the script lock for the whole of `fn()`, so a slow send keeps every other job waiting on the same lock, including `flushLog`, which gives up after 10 seconds. Here the lock is held only while one key is read and written.

Retrying a `failed` row is only safe if a send that threw really did not happen. For an HTTP call to a payment or messaging API, a timeout can throw after the provider accepted the request. Pair the retry with the provider's idempotency key, as the guide already recommends.

## Step 2: a watchdog that reads the ledger and remembers what it said

```javascript
const REMIND_EVERY_H = 24;

// Rows a person has to look at. 'failed' is left out:
// the next run retries it.
function ledgerProblems_() {
  const ledger = ledgerSheet_();
  const last = ledger.getLastRow();
  if (!last) return [];
  const now = Date.now();
  return ledger.getRange(1, 1, last, 5).getValues()
    .filter(r => r[2] === 'gave_up' ||
      (r[2] === 'in_progress' &&
       now - new Date(r[0]).getTime() > STUCK_MIN * 60000))
    .map(r => ({
      key: r[1] + ':' + r[2],
      text: `${r[1]}: ${r[2]}, attempt ${r[3]} ${r[4]}`.trim()
    }));
}

// Same checks as the original watchdog, but each problem has
// a stable key. The minutes go in the text, never in the key.
function staleJobs_() {
  const props = PropertiesService.getScriptProperties();
  const out = [];
  Object.keys(EXPECTED_EVERY_MIN).forEach(job => {
    const raw = props.getProperty('heartbeat:' + job);
    if (!raw) {
      out.push({ key: job + ':never',
        text: `${job}: no successful run ever recorded` });
      return;
    }
    const beat = JSON.parse(raw);
    const ageMin = Math.round(
      (Date.now() - new Date(beat.at).getTime()) / 60000);
    const every = EXPECTED_EVERY_MIN[job];
    if (ageMin > every * 2) {
      out.push({ key: job + ':late',
        text: `${job}: last run ${ageMin} min ago, ` +
          `expected every ${every}` });
    } else if (beat.status !== 'ok') {
      out.push({ key: job + ':error',
        text: `${job}: ran ${ageMin} min ago but errored` });
    }
  });
  return out;
}

// Mails when the set of problems changes (including back
// to none), plus one reminder a day while it stays broken.
function watchdog() {
  const props = PropertiesService.getScriptProperties();
  const problems = staleJobs_().concat(ledgerProblems_());
  const keys = problems.map(p => p.key).sort().join('|');
  const lastKeys = props.getProperty('watchdog:keys') || '';
  const sentAt = Number(props.getProperty('watchdog:sentAt'));
  const due = keys !== '' &&
    Date.now() - sentAt > REMIND_EVERY_H * 3600000;
  if (keys === lastKeys && !due) return;

  if (MailApp.getRemainingDailyQuota() < 5) {
    console.error('watchdog mail suppressed: quota low');
    return;
  }
  MailApp.sendEmail(ALERT_TO,
    keys ? `Automation watchdog: ${problems.length} ` +
      'problem(s) need attention'
      : 'Automation watchdog: all clear',
    keys ? problems.map(p => p.text).join('\n')
      : 'Every watched job is back on schedule.');
  props.setProperty('watchdog:keys', keys);
  props.setProperty('watchdog:sentAt', String(Date.now()));
}
```

- `ledgerProblems_` turns `gave_up` rows and `in_progress` rows older than 30 minutes into problems. `failed` rows are left out, because the next run retries them.
- Each problem has a stable key, such as `invoice:1001:gave_up` or `magento_order_sync:late`. The minutes since the last run go into the email text, never into the key. Otherwise "last run 125 min ago" and "last run 185 min ago" would be two different problems, and the dedup would do nothing.
- `watchdog` mails when the set of keys changes, once a day while it stays the same, and once with "all clear" when it empties. That last email also covers the guide's own point that an alert channel you never hear from looks the same as a broken one.
- The quota check the guide put in `alertOnce` is in the watchdog too.

`alertOnce` gets one change: `cache.put(fingerprint, '1', 21600)` instead of `3600`. 21,600 seconds is the most `CacheService` accepts. With the 3,600-second window, an hourly job that keeps failing alerts every hour. With six hours it alerts four times a day, and the watchdog's daily reminder covers the rest.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/a4cfdd71-7f3b-4519-91a3-5dea998e8733.png align="center")

## The same scenarios, before and after

| Scenario | The guide's code | This version |
| --- | --- | --- |
| Order 1001's invoice throws once, the daily job runs 3 more days | 0 invoices sent; row stays in_progress; every run ok | 1 invoice sent on day 2; row done after attempt 2 |
| The invoice throws on every attempt | Skipped after the first failure; nothing reports it | Tried 3 times, then gave_up; the watchdog mails once, then daily |
| The run is killed mid-send | Row skipped; nothing reports it | Not retried; the watchdog reports it after 30 minutes |
| Hourly job broken Fri 18:00 to Mon 08:00, watchdog hourly | 124 alert emails, 48 on Saturday | 14 alert emails, 5 on Saturday |
| The problem is fixed | No email | One "all clear" email |

## Pitfalls

- **Cache entries can leave early.** Google's reference for `put` says cached data "may be removed before this time". An evicted fingerprint means a duplicate alert, not a lost one, which is the safe direction. The watchdog keeps its memory in Script Properties, which do not expire.
- **A dedup key with a number in it does not dedup.** `alertOnce` fingerprints the first 120 characters of the error message. If your message carries a row count, a cursor or a time ("fetched 37 orders but wrote none"), every run is a new fingerprint. Keep the variable parts out of the thrown message and put them in the log context.
- **Cache keys have a 250-character limit.** `'alert:' + job + ':'` plus the Base64 of a 120-character ASCII message is about 170 characters. Base64 of UTF-8 grows with non-ASCII text, so a long message in Turkish or Japanese can pass 250. I have not checked what Apps Script does then; if `cache.put` throws inside the wrapper's `catch`, the original error is replaced by that one.
- **Writing** `done` **can fail after the send.** If `markKey_` throws after a successful send, the row stays `in_progress` and goes to a person after 30 minutes. That is the safe direction too: a missed tick, not a second invoice.

## How it drifts once it is in use

- `gave_up` **rows pile up.** Each one is a customer without an invoice. The watchdog reminds you daily. Someone has to fix the cause and set the row back to `failed` to release it, or to `done` if the invoice went out by hand.
- **A renamed job looks dead.** Rename `invoice_mailer` in the code but not in `EXPECTED_EVERY_MIN`, and the watchdog reports the old name as late from the next day on. Rename both in the same change.
- **The person who installed the triggers leaves.** Installable triggers run as the account that created them. If the watchdog was installed by the same person as the jobs, it stops with them. Install it from a shared admin account. Our [Workspace offboarding write-up](https://dev.to/hayrullahkar/google-workspace-offboarding-a-dry-run-that-survives-a-failed-step-1k4j) is about the day that account gets suspended.
- **The ledger grows.** `claimKey_` still reads the whole key column on every call. The guide's advice to shard the ledger by month applies unchanged.

## What I tested, and what I did not

18 tests in Node against the same fakes, run under UTC, Europe/Istanbul, America/Los_Angeles and Asia/Tokyo. They cover a failed send retried once and marked `done`, the attempt count, `gave_up` after three failures, `done` and stuck `in_progress` rows never re-running, the lock being free while `fn()` runs, which rows the watchdog reports and after how long, no repeat email when only the minutes change, the daily reminder, the single "all clear", a new problem triggering a new email, the quota check, and the six-hour alert window. Then 15 deliberate breakages of the code, such as retrying `in_progress` rows, putting the minutes in the dedup key, or dropping the watchdog's memory. All 15 were caught.

The two findings at the top come from the guide's code, unchanged, in the same fakes. The only line I left out is its one-line usage example, which refers to an `order` variable that does not exist when the file loads.

What I have not measured:

- A real Apps Script project. Everything above ran against fakes.
- Whether `MailApp.sendEmail` can throw after Gmail has accepted a message.
- Google's quota day. My fake resets the quota at UTC midnight.
- How long other jobs wait on the script lock in a real project while the original `processOnce` holds it.
- What happens to installable triggers when the account that created them is suspended.
- How often the watchdog should run. The guide does not say; I assumed hourly.

## The takeaway

An idempotency ledger has two jobs: stop a retry from doing the work twice, and let it do the work once. Record a send that threw as `failed`, retry it a few times, and hand a person only the rows that nobody can know about.

The rest of the layer, from the wrapper to Cloud Logging costs, is in the original guide on the [MageSheet blog](https://magesheet.com/blog/apps-script-error-handling-monitoring).