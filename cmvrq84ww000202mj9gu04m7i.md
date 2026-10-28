---
title: "Google Workspace Offboarding: A Dry Run That Survives a Failed Step"
seoTitle: "Google Workspace Offboarding Script: Dry Run That Resumes"
seoDescription: "Apps Script offboarding with a reviewed plan in a sheet: SKIP rows, typed-email confirmation, and a rerun that resumes instead of repeating."
datePublished: 2026-10-28T06:21:38.893Z
cuid: cmvrq84ww000202mj9gu04m7i
slug: workspace-offboarding-dry-run-resume
canonical: https://magesheet.com/blog/automate-google-workspace-onboarding-offboarding
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/89135947-73e8-4e05-b1e7-46633665a80f.jpg
tags: security, automation, google-apps-script, google-workspace, magesheet

---

Our guide to [automating Google Workspace onboarding and offboarding](https://magesheet.com/blog/automate-google-workspace-onboarding-offboarding) builds the leaver process around a dry run. `planOffboarding` reads the account and writes a plan to a sheet, a person reviews it, and `applyOffboarding(token)` executes "exactly that plan and nothing else". It stops on the first failure so you are left with "a knowable state instead of a mystery".

I ran those two functions, unchanged, against a fake Admin SDK with one invented leaver: two groups, one phone, one license. The helpers the guide leaves to the reader (`writePlanToSheet`, `execute`, `audit`) were written the plainest way it describes them. Three things happened that the guide's promises do not cover.

- **A failed plan cannot be finished.** The tokens step threw a transient `Backend Error` on the first run. The second run started again at step 1, signed out and suspended a second time, then stopped on the group the first run had already removed. The third run stopped in the same place. In my fake, removing someone who is no longer a member throws, which is how I expect the Directory API to answer. The license was never dropped.
- **A failed last step wipes the phone twice.** With group removal made harmless and only the license step failing once, the second run repeated every step and sent the device wipe a second time.
- **The reviewed sheet is not what runs.** The plan executed comes from `CacheService`. I deleted the `ACCOUNT_WIPE` row from the sheet, as a reviewer would for a personal phone, and the wipe went out anyway.

\[ GÖRSELİ BURAYA EKLE → diagrams/workspace-offboarding-dry-run-resume-demo.gif · alt: "Terminal: the same invented leaver run twice. With the original plan and apply functions, the license step fails once, the second apply repeats sign out, suspend, both group removals and the wipe, and two wipe commands reach the phone. With the resumable version, the second apply sends only the license step, and one wipe command reaches the phone." \]

The pattern in the guide is the right one. The problem is where the plan lives and how it remembers progress. This post moves both into the sheet.

## Why the cache is the wrong home for a plan

`CacheService` is a cache. Google's reference for `put` says it plainly: "cached data may be removed before this time if a lot of data is cached", and six hours is the maximum it accepts anyway. A plan that a manager reviews after lunch can be gone before they finish. The guide treats an expired plan as a safe failure, and it is, but it also means the plan has no durable record of which steps already ran.

That missing record is the root of the first two findings. `applyOffboarding` deletes the cached plan only when every step succeeds. After a failure the plan is still there, intact, with nothing to say that steps 1 to 4 are done. Running it again is the only option the guide offers, and running it again means running all of it.

The third finding has the same cause. The sheet is written once, for people to read, and never read back. The guide also suggests making the reviewer type the target email into a confirmation cell before anything destructive runs. Its code does not do that yet.

## Step 1: the plan is sheet rows with fingerprints

`planOffboarding` reads the same things as before and builds the same actions. The differences are where it puts them. Each action becomes one row on a `Plans` tab with a sequence number, a blank decision column (G) and a blank status column (H). The script also keeps a SHA-256 fingerprint of each row's op, target and detail in Script Properties, together with the leaver's email and the time the plan was made.

```javascript
const PLAN_TAB = 'Plans';  // A token B seq C op D target E detail
                           // F reversible G decision H status I at
const MAX_PLAN_AGE_H = 24;

// PHASE 1: decide. Reads only, then writes one sheet row per
// action and keeps a fingerprint of each row it wrote.
function planOffboarding(email) {
  const user = AdminDirectory.Users.get(email);
  const groups = AdminDirectory.Groups
    .list({ userKey: email }).groups || [];
  const devices = AdminDirectory.Mobiledevices
    .list('my_customer', { query: 'email:' + email })
    .mobiledevices || [];

  const acts = [['SIGN_OUT', email, '', 'n/a'],
                ['SUSPEND', email, '', 'yes']];
  groups.forEach(g =>
    acts.push(['REMOVE_GROUP', email, g.email, 'yes']));
  acts.push(['REVOKE_TOKENS', email, '', 'no']);
  devices.forEach(d => acts.push(
    ['ACCOUNT_WIPE', d.resourceId, d.model, 'NO']));
  acts.push(['DROP_LICENSE', email, user.orgUnitPath, 'yes']);

  const token = Utilities.getUuid().slice(0, 6).toUpperCase();
  const prints = {};
  const rows = acts.map((a, i) => {
    prints[i + 1] = fingerprint_(a[0], a[1], a[2]);
    return [token, i + 1, a[0], a[1], a[2], a[3], '', '', ''];
  });
  const sh = SpreadsheetApp.getActive().getSheetByName(PLAN_TAB);
  sh.getRange(sh.getLastRow() + 1, 1, rows.length, 9)
    .setValues(rows);
  PropertiesService.getScriptProperties().setProperty(
    'plan_' + token,
    JSON.stringify({ email: email, created: Date.now(),
                     prints: prints }));
  return token;
}

function fingerprint_(op, target, detail) {
  const bytes = Utilities.computeDigest(
    Utilities.DigestAlgorithm.SHA_256,
    [op, target, detail].join('|'));
  return Utilities.base64Encode(bytes).slice(0, 16);
}
```

The reviewer's only tool is column G. Typing `SKIP` there leaves a row out. Everything else on the row is checked against its fingerprint before anything runs, so the plan a person looked at is the plan that executes.

## Step 2: apply runs the reviewed rows and remembers progress

`applyOffboarding` now takes two arguments: the token and the leaver's email, typed by the person running it. That is the confirmation gate from the guide, in code. Then it refuses a plan older than 24 hours, takes a script lock so a trigger and a person cannot run the same plan at once, and checks every row.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/ad071a85-df45-49b1-a329-a257c0f2dcec.png align="center")

```javascript
// PHASE 2: do. Runs the rows a person reviewed, in order. Type
// SKIP in column G to leave a row out; do not delete rows.
// Safe to run again after a failure: DONE rows are not repeated.
function applyOffboarding(token, typedEmail) {
  const props = PropertiesService.getScriptProperties();
  const raw = props.getProperty('plan_' + token);
  if (!raw) throw new Error('unknown or finished plan ' + token);
  const plan = JSON.parse(raw);
  if (typedEmail !== plan.email) {
    throw new Error('typed email does not match ' + token);
  }
  if (Date.now() - plan.created > MAX_PLAN_AGE_H * 3600000) {
    throw new Error('plan older than ' + MAX_PLAN_AGE_H +
                    ' h: run planOffboarding again');
  }
  const lock = LockService.getScriptLock();
  lock.waitLock(30000);  // a trigger and a person, same moment
  try {
    const sh = SpreadsheetApp.getActive().getSheetByName(PLAN_TAB);
    const all = sh.getDataRange().getValues();
    const mine = [];
    all.forEach((r, i) => {
      if (r[0] === token) mine.push({ row: i + 1, v: r });
    });
    checkRows_(mine, plan.prints);
    mine.forEach(m => {
      const [, seq, op, target, detail, , decision, status] = m.v;
      if (status === 'DONE' || status === 'SKIPPED') return;
      const at = new Date();
      if (decision === 'SKIP') {
        sh.getRange(m.row, 8, 1, 2).setValues([['SKIPPED', at]]);
        return;
      }
      try {
        execute_({ op: op, target: target, detail: detail });
      } catch (err) {
        sh.getRange(m.row, 8, 1, 2)
          .setValues([['FAILED: ' + err.message, at]]);
        throw err;  // stop here; the next run resumes at this row
      }
      sh.getRange(m.row, 8, 1, 2).setValues([['DONE', at]]);
    });
    props.deleteProperty('plan_' + token);
  } finally {
    lock.releaseLock();
  }
}

// Every planned row must still be there, unchanged, and no row
// may have been added. Edits go in column G, nowhere else.
function checkRows_(mine, prints) {
  const seen = {};
  mine.forEach(m => {
    const seq = m.v[1];
    const fp = fingerprint_(m.v[2], m.v[3], m.v[4]);
    if (prints[seq] !== fp) {
      throw new Error('row ' + seq + ' changed or added after ' +
                      'planning: plan again');
    }
    seen[seq] = true;
  });
  Object.keys(prints).forEach(seq => {
    if (!seen[seq]) {
      throw new Error('row ' + seq + ' was deleted: restore it ' +
                      'and type SKIP in column G instead');
    }
  });
  mine.sort((a, b) => a.v[1] - b.v[1]);
}
```

Each step writes `DONE` and a timestamp to its own row the moment it succeeds. A failure writes `FAILED:` and the error message to that row and stops. Running the same token again skips every `DONE` and `SKIPPED` row and starts at the failed one. The plan's Script Properties entry is deleted only after the last row is handled, so a finished plan cannot be run a second time.

`checkRows_` turns the sheet into a contract. A row whose op, target or detail changed after planning stops the run. So does a row someone added, and so does a planned row that disappeared. The error for a deleted row says what to do instead: restore it and type `SKIP`. Deleting a row is how the wipe slipped through in the original, so it has to be loud.

## Step 3: steps that check before they act

Progress lives in the sheet now, but there is still a gap between an API call succeeding and the `DONE` being written. If the execution dies in that gap, the next run repeats one step. So each step looks before it acts wherever the API makes that cheap.

```javascript
const PRODUCT_ID = 'Google-Apps';
const SKU_BY_OU = { '/Sales': '1010020028' };  // your OU -> SKU map

// One case per op. Each one checks before it acts, so a run
// that died between the call and the DONE write is harmless.
function execute_(a) {
  switch (a.op) {
    case 'SIGN_OUT':
      return AdminDirectory.Users.signOut(a.target);
    case 'SUSPEND':
      return AdminDirectory.Users
        .update({ suspended: true }, a.target);
    case 'REMOVE_GROUP': {
      const m = AdminDirectory.Members.hasMember(a.detail, a.target);
      if (!m.isMember) return;  // already out
      return AdminDirectory.Members.remove(a.detail, a.target);
    }
    case 'REVOKE_TOKENS': {
      const list = AdminDirectory.Tokens.list(a.target).items || [];
      list.forEach(t =>
        AdminDirectory.Tokens.remove(a.target, t.clientId));
      (AdminDirectory.Asps.list(a.target).items || [])
        .forEach(p => AdminDirectory.Asps.remove(a.target, p.codeId));
      return;
    }
    case 'ACCOUNT_WIPE':
      return AdminDirectory.Mobiledevices.action(
        { action: 'admin_account_wipe' }, 'my_customer', a.target);
    case 'DROP_LICENSE': {
      const sku = SKU_BY_OU[a.detail];
      if (!sku) throw new Error('no SKU for org unit ' + a.detail);
      return AdminLicenseManager.LicenseAssignments
        .remove(PRODUCT_ID, sku, a.target);
    }
    default:
      throw new Error('unknown op ' + a.op);
  }
}
```

`REMOVE_GROUP` asks `hasMember` first, so a group that was already left is not an error. `REVOKE_TOKENS` works from the current token and app-password lists, which are empty the second time. `DROP_LICENSE` looks up the SKU from the org unit stored in the plan and refuses an org unit it does not know, instead of guessing. An unknown op throws, so a typo in the sheet cannot turn into a silent no-op.

Two steps cannot check first. A device has no "already wiped" flag to ask about, and signing out twice is harmless. That is why the wipe is the step that most needs the per-row `DONE`.

Here is the same scenario, run through both versions:

| Same fake leaver | The guide's code | This version |
| --- | --- | --- |
| Tokens step fails once, then apply again | Runs 2 and 3 stop at the group run 1 already left; the license is never dropped | Run 2 starts at the tokens row; sign-out is sent once |
| License step fails once, then apply again | The phone wipe is sent twice | The phone wipe is sent once |
| Reviewer deletes the wipe row from the sheet | The wipe is sent | The run stops before any call: restore the row and type SKIP |
| Reviewer types SKIP on the wipe row | No such column | Row marked SKIPPED, no wipe |
| Person applying types the wrong email | No check | Refused before any call |

## Pitfalls

- **Column G is the only editable column.** People sort, filter and "tidy" sheets. Sorting is fine because `checkRows_` re-sorts by sequence. Editing a target or op is not, and the run will stop and say so.
- **Script Properties have a size limit.** Each value is capped at 9 KB. A fingerprint is 16 characters, so a plan's entry stays small until a leaver is in several hundred groups. If yours are, store the fingerprints on a hidden tab instead.
- **The advanced service argument order is not obvious.** The Admin SDK advanced services follow the REST method, path parameters after the body. `Members.hasMember(groupKey, memberKey)`, `Mobiledevices.action(body, customerId, resourceId)` and `LicenseAssignments.remove(productId, skuId, userId)` follow that order. Let the editor's autocomplete confirm each one before you trust it.
- `admin_account_wipe`**, not** `admin_remote_wipe`**.** The first removes work data from the device. The second factory-resets it, which on a personal phone is somebody's photos.

## How it drifts once it is in use

- **The account changes after the plan.** Someone adds the leaver to a new group an hour after planning. The plan does not know about it. The 24-hour limit keeps that window short. For a leaver handled across several days, plan again on the day you apply.
- **Editions change.** `SKU_BY_OU` is a copy of a billing decision. When finance moves a team to another edition, the license step for that team starts failing with "no SKU". That is better than dropping the wrong license, but it is a row to update.
- **The** `Plans` **tab only grows.** Every leaver adds rows. Archive finished plans once a quarter. The audit value is in the `DONE` timestamps, so do not delete them.
- **Someone runs apply from a trigger.** The lock stops two runs from overlapping. It does not stop a trigger from applying a plan nobody reviewed. Keep apply a manual action with the typed email.
- **A** `FAILED` **row waits for someone to notice it.** Nothing retries on its own, which is deliberate for a leaver. Send the failure to a person the same hour; our [guide to Apps Script error handling and alerts](https://magesheet.com/blog/apps-script-error-handling-monitoring) covers that layer.

## What I tested, and what I did not

28 tests in Node against a fake Admin SDK, Licensing service, Sheets, Script Properties and lock, run under UTC and Europe/Istanbul. They cover each refusal (wrong or missing email, unknown token, a 24-hour-old plan, deleted, edited and added rows), `SKIP`, resuming after a failure, the phone being wiped once when the last step fails, rows of another plan left alone, and each step's own check. Then 20 deliberate breakages of the code, such as repeating `DONE` rows, ignoring `SKIP`, skipping the fingerprint check or switching to `admin_remote_wipe`. All 20 were caught.

The three findings at the top come from running the guide's own `planOffboarding` and `applyOffboarding` unchanged in the same kind of fake.

What I have not measured:

- A real Workspace tenant. Nothing here has run against Google.
- What the Directory API returns when you remove someone who is no longer a member. The first finding depends on it throwing. The second and third findings do not.
- The exact advanced service signatures listed above. They follow the REST reference, not a run.
- A failure in the moment between a successful wipe or license call and its `DONE` write. One step would repeat. The tests do not reproduce it.
- How often Google returns transient errors on these calls. I made one happen on purpose.

## The takeaway

A dry run protects you only if the reviewed plan is the one that executes, and a failed run can continue where it stopped. Put the plan in the sheet, check every row against what was planned, and write `DONE` after each step.

The full lifecycle, from the native options to the license audit, is in the original guide on the [MageSheet blog](https://magesheet.com/blog/automate-google-workspace-onboarding-offboarding).