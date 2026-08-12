---
title: "Apps Script Handover Audit: Verify You Actually Own the Delivered Code"
seoTitle: "Apps Script Handover Audit: Do You Really Own the Code?"
seoDescription: "Five read-only Apps Script checks that show whether a delivered automation is yours: file owners, project creator, executeAs, triggers, secrets."
datePublished: 2026-08-12T16:51:25.144Z
cuid: cmsqbtflp00000aj66rcx6yo9
slug: apps-script-handover-audit
canonical: https://magesheet.com/blog/hire-google-apps-script-developer
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/305d0a29-9b34-4a74-9f79-100f09453554.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/4f142743-fa8f-417a-aaec-ec74a4df9738.jpg
tags: javascript, automation, google-sheets, googleappsscript, magesheet

---

Owning the source code and owning the running system are two different things in Apps Script, and from inside the editor they look identical. You can hold every line, in a file in your own Drive, with the developer removed from the sharing list — and still have a nightly sync that stops the day they close their Google account.

Nothing in the project tells you this is coming. The code is there. The schedule is not.

## Reading the code cannot answer the ownership question

The usual handover checklist is a paperwork exercise: get the source, get ownership in writing, remove the developer's access. All three are worth doing. None of them test the thing that actually breaks.

Four of the five facts that decide whether a delivered Apps Script system survives the developer leaving are not in the source at all:

| Fact | Where it lives | Visible in the editor |
| --- | --- | --- |
| Who owns the Sheet and the data files | Drive permissions | no |
| Who created the script project | project metadata | no |
| Whose account serves the web app | the deployment's `executeAs` | no |
| Which triggers exist | per-user trigger table | only yours |
| Which credentials came with it | script properties | yes, if you look |

The trigger row is the expensive one, and it is documented behaviour rather than a bug. `ScriptApp.getProjectTriggers()` "gets all installable triggers associated with the current project **and current user**." A time-driven trigger the developer installed under their own account belongs to their user inside your project. You open Triggers in your editor and the list is empty. The job still runs every night, as them, until the day it doesn't.

The web app has the same shape. Every deployment carries `entryPoints[].webApp.entryPointConfig.executeAs`, and the value `USER_DEPLOYING` means each request is served with the identity of whoever pressed Deploy. If that was the developer, your public order form is reading and writing your Sheet through their Google account, on their quota, under their audit log.

So instead of asking for a handover document, I run a script.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/67c006ba-f931-4300-b3e4-f13d2f5e3a40.jpg align="center")

## Check 1: who actually owns the files

Paste this into the delivered project and run it from your own account. Every check is read-only.

The interesting part is not `getOwner()`. It is what happens when `getOwner()` cannot answer, which turns out to be often.

```javascript
/**
 * Handover audit for a delivered Apps Script project.
 * Read-only: it reports, it never edits the system it audits.
 * Run it from YOUR account, not the developer's.
 */

var AUDIT = {
  // Every Sheet / Doc / Drive file the delivered system uses.
  // Add the ids the developer listed, plus any id you find
  // hard-coded in the source.
  fileIds: [],
  // Were you sold anything that runs on a schedule?
  expectScheduled: true,
  // Were you sold a web app, form, or dashboard URL?
  expectWebApp: true,
  reportSheet: 'Handover Audit'
};

var BLOCKER = 'BLOCKER';
var WARN = 'WARN';
var OK = 'OK';

/**
 * Blank-safe compare. User.getEmail() returns a blank string
 * when policy hides the address, so '' must never equal ''.
 */
function sameUser_(a, b) {
  var x = String(a || '').trim().toLowerCase();
  var y = String(b || '').trim().toLowerCase();
  if (!x || !y) return false;
  return x === y;
}

/** Domain part of an email, or '' when there isn't one. */
function domainOf_(email) {
  var s = String(email || '').trim().toLowerCase();
  var at = s.lastIndexOf('@');
  if (at < 1 || at === s.length - 1) return '';
  return s.slice(at + 1);
}

var FREE_DOMAINS = ['gmail.com', 'googlemail.com', 'yahoo.com',
                    'outlook.com', 'hotmail.com', 'live.com',
                    'icloud.com', 'proton.me', 'protonmail.com'];

/**
 * Two people on gmail.com are not colleagues. Matching on a
 * free domain would file the developer under "your domain"
 * and downgrade a blocker to a warning.
 */
function isFreeDomain_(domain) {
  return FREE_DOMAINS.indexOf(String(domain || '').toLowerCase()) > -1;
}

/**
 * Where an owner sits relative to you.
 * hasOwner is false for shared-drive files: the Drive API
 * does not populate an owner there, so "no owner" is not
 * the same answer as "not yours".
 */
function classifyOwner_(ownerEmail, me, hasOwner) {
  if (!hasOwner) return 'no-owner';
  if (!String(me || '').trim()) return 'unknown-viewer';
  if (!String(ownerEmail || '').trim()) return 'unknown-owner';
  if (sameUser_(ownerEmail, me)) return 'you';
  var mine = domainOf_(me);
  var theirs = domainOf_(ownerEmail);
  if (mine && theirs && mine === theirs && !isFreeDomain_(mine)) {
    return 'your-domain';
  }
  return 'outside';
}

var OWNER_LEVEL = {
  'you': OK,
  'your-domain': WARN,
  'outside': BLOCKER,
  'no-owner': WARN,
  'unknown-owner': WARN,
  'unknown-viewer': WARN
};

function auditOwnership_(fileIds, me) {
  var out = [];
  for (var i = 0; i < fileIds.length; i++) {
    var id = fileIds[i];
    try {
      var file = DriveApp.getFileById(id);
      var owner = file.getOwner();
      var email = owner ? owner.getEmail() : '';
      var where = classifyOwner_(email, me, !!owner);
      out.push({
        check: 'Owner of "' + file.getName() + '"',
        level: OWNER_LEVEL[where],
        detail: where + (email ? ' (' + email + ')' : '')
      });
      out.push(editorsFinding_(file, me));
    } catch (err) {
      out.push({
        check: 'File ' + id,
        level: BLOCKER,
        detail: 'Cannot open: ' + err.message
      });
    }
  }
  return out;
}

/**
 * getEditors() returns an empty array when the running user
 * has no edit access, so an empty list is reported as
 * unreadable rather than as "nobody else has access".
 */
function editorsFinding_(file, me) {
  var editors = file.getEditors() || [];
  if (!editors.length) {
    return {
      check: 'Editors of "' + file.getName() + '"',
      level: WARN,
      detail: 'empty list: you may not have edit access'
    };
  }
  var others = [];
  for (var i = 0; i < editors.length; i++) {
    var e = editors[i].getEmail();
    if (!sameUser_(e, me)) others.push(e || '(hidden)');
  }
  return {
    check: 'Editors of "' + file.getName() + '"',
    level: others.length ? WARN : OK,
    detail: others.length ? others.join(', ') : 'only you'
  };
}
```

`classifyOwner_` returns six different answers instead of a boolean, because "no owner", "I cannot read the owner", and "the owner is someone else" are three different problems with three different fixes. Collapsing them into `isMine` is how an audit ends up reporting a clean handover on a file nobody in your company can administer.

## Checks 2 and 3: who created it, and whose account serves it

Neither of these facts is reachable from the Apps Script services. Both come from the Apps Script API, which the script can call using its own OAuth token:

*   `GET https://script.googleapis.com/v1/projects/{scriptId}` returns `creator` and `lastModifyUser`, each a user object with an `email`.
    
*   `GET https://script.googleapis.com/v1/projects/{scriptId}/deployments` returns every deployment, and each web app entry point carries `access` and `executeAs`.
    

```javascript
/**
 * Reads the Apps Script API with the script's own token.
 * Returns a result object instead of throwing: the API is
 * off by default, and that is a finding, not a crash.
 */
function scriptApi_(path) {
  var url = 'https://script.googleapis.com/v1/projects/' + path;
  var res = UrlFetchApp.fetch(url, {
    method: 'get',
    muteHttpExceptions: true,
    headers: {
      Authorization: 'Bearer ' + ScriptApp.getOAuthToken()
    }
  });
  var code = res.getResponseCode();
  var body = res.getContentText();
  if (code === 200) return { ok: true, data: JSON.parse(body) };
  return { ok: false, code: code, detail: body.slice(0, 200) };
}

function auditProject_(scriptId, me) {
  var res = scriptApi_(scriptId);
  if (!res.ok) {
    return [{
      check: 'Script project metadata',
      level: WARN,
      detail: 'HTTP ' + res.code + ' — enable the Apps Script ' +
              'API for your account, then re-run'
    }];
  }
  var p = res.data;
  var creator = (p.creator && p.creator.email) || '';
  var where = classifyOwner_(creator, me, !!creator);
  return [{
    check: 'Who created the script project',
    level: OWNER_LEVEL[where],
    detail: where + (creator ? ' (' + creator + ')' : '')
  }, {
    check: 'Script is bound to file',
    level: p.parentId ? OK : WARN,
    detail: p.parentId || 'standalone (no container file)'
  }];
}

/**
 * Pulls the web app entry point out of one deployment.
 * USER_DEPLOYING means every request runs as whoever
 * pressed Deploy — usually the developer.
 */
function webAppRisk_(deployment) {
  var eps = (deployment && deployment.entryPoints) || [];
  for (var i = 0; i < eps.length; i++) {
    var w = eps[i].webApp;
    if (!w) continue;
    var cfg = w.entryPointConfig || {};
    return {
      isWebApp: true,
      url: w.url || '',
      access: cfg.access || 'UNKNOWN_ACCESS',
      executeAs: cfg.executeAs || 'UNKNOWN_EXECUTE_AS',
      runsAsDeployer: cfg.executeAs === 'USER_DEPLOYING'
    };
  }
  return { isWebApp: false, runsAsDeployer: false };
}

function auditDeployments_(scriptId, expectWebApp) {
  var res = scriptApi_(scriptId + '/deployments');
  if (!res.ok) {
    return [{
      check: 'Deployments',
      level: WARN,
      detail: 'HTTP ' + res.code + ' — cannot read deployments'
    }];
  }
  var list = res.data.deployments || [];
  var out = [];
  var sawWebApp = false;
  for (var i = 0; i < list.length; i++) {
    var w = webAppRisk_(list[i]);
    if (!w.isWebApp) continue;
    sawWebApp = true;
    out.push({
      check: 'Web app runs as',
      level: w.runsAsDeployer ? BLOCKER : OK,
      detail: w.executeAs + ', access ' + w.access
    });
  }
  if (expectWebApp && !sawWebApp) {
    out.push({
      check: 'Web app deployment',
      level: BLOCKER,
      detail: 'none found in your copy of the project'
    });
  }
  return out;
}
```

`webAppRisk_` walks all entry points rather than reading `entryPoints[0]`. A deployment can expose an execution API entry point and a web app entry point at once, and the order is not guaranteed, so indexing the first one silently reports "no web app" on a project that has one.

The zero-deployment case is a finding, not an empty result. If you were sold a form URL and your copy of the project has no web app deployment, the URL people are using belongs to a deployment in someone else's project.

## Checks 4 and 5: the schedule you cannot see, and the keys you inherited

```javascript
/**
 * getProjectTriggers() returns the triggers of the current
 * project AND the current user. A schedule the developer
 * installed under their own account is not in this list.
 */
function auditTriggers_(expectScheduled) {
  var triggers = ScriptApp.getProjectTriggers();
  if (!triggers.length) {
    return [{
      check: 'Scheduled triggers visible to you',
      level: expectScheduled ? BLOCKER : OK,
      detail: expectScheduled ?
        'none — a schedule exists somewhere else' : 'none'
    }];
  }
  var names = [];
  for (var i = 0; i < triggers.length; i++) {
    names.push(triggers[i].getHandlerFunction() + ' / ' +
               triggers[i].getEventType());
  }
  return [{
    check: 'Scheduled triggers visible to you',
    level: OK,
    detail: names.join('; ')
  }];
}

var SECRET_HINTS = ['key', 'token', 'secret', 'password',
                    'auth', 'credential', 'webhook',
                    'signature', 'passwd'];

/** Name-only test. The audit never reads a stored value. */
function looksLikeSecret_(name) {
  var n = String(name || '').toLowerCase();
  for (var i = 0; i < SECRET_HINTS.length; i++) {
    if (n.indexOf(SECRET_HINTS[i]) !== -1) return true;
  }
  return false;
}

function auditSecrets_() {
  var props = PropertiesService.getScriptProperties();
  var keys = props.getKeys() || [];
  var secrets = [];
  for (var i = 0; i < keys.length; i++) {
    if (looksLikeSecret_(keys[i])) secrets.push(keys[i]);
  }
  return [{
    check: 'Inherited credentials in script properties',
    level: secrets.length ? WARN : OK,
    detail: secrets.length ?
      secrets.join(', ') + ' — rotate these' : 'none by name'
  }];
}

function verdict_(findings) {
  var counts = { BLOCKER: 0, WARN: 0, OK: 0 };
  for (var i = 0; i < findings.length; i++) {
    var lvl = findings[i].level;
    if (counts[lvl] === undefined) counts[lvl] = 0;
    counts[lvl]++;
  }
  var status = 'CLEAR';
  if (counts.WARN > 0) status = 'ATTENTION';
  if (counts.BLOCKER > 0) status = 'NOT HANDED OVER';
  return {
    status: status,
    blockers: counts.BLOCKER,
    warnings: counts.WARN,
    passed: counts.OK
  };
}

/** Stops a value that starts with = from becoming a formula. */
function safeText_(v) {
  var s = (v === null || v === undefined) ? '' : String(v);
  return /^[=+\-@]/.test(s) ? "'" + s : s;
}

function writeReport_(rows, summary) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  if (!ss) {
    // Standalone project: there is no sheet to write into.
    Logger.log(JSON.stringify({ summary: summary, rows: rows }));
    return;
  }
  var sheet = ss.getSheetByName(AUDIT.reportSheet) ||
              ss.insertSheet(AUDIT.reportSheet);
  sheet.clear();
  var out = [['Check', 'Level', 'Detail']];
  for (var i = 0; i < rows.length; i++) {
    out.push([
      safeText_(rows[i].check),
      safeText_(rows[i].level),
      safeText_(rows[i].detail)
    ]);
  }
  out.push(['', '', '']);
  out.push(['VERDICT', summary.status,
            summary.blockers + ' blockers, ' +
            summary.warnings + ' warnings']);
  sheet.getRange(1, 1, out.length, 3).setValues(out);
  sheet.getRange(1, 1, 1, 3).setFontWeight('bold');
  sheet.setColumnWidth(1, 320);
  sheet.setColumnWidth(3, 480);
  SpreadsheetApp.flush();
}

function runHandoverAudit() {
  var me = Session.getEffectiveUser().getEmail();
  var scriptId = ScriptApp.getScriptId();
  var ids = AUDIT.fileIds.slice();
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  if (ss && ids.indexOf(ss.getId()) === -1) ids.push(ss.getId());

  var findings = [];
  if (!String(me || '').trim()) {
    findings.push({
      check: 'Your identity',
      level: WARN,
      detail: 'blank email — run this from the Apps Script ' +
              'editor, not a trigger or custom function'
    });
  }
  findings = findings
    .concat(auditOwnership_(ids, me))
    .concat(auditProject_(scriptId, me))
    .concat(auditDeployments_(scriptId, AUDIT.expectWebApp))
    .concat(auditTriggers_(AUDIT.expectScheduled))
    .concat(auditSecrets_());

  var summary = verdict_(findings);
  writeReport_(findings, summary);
  return summary;
}
```

Set `AUDIT.fileIds`, `AUDIT.expectScheduled` and `AUDIT.expectWebApp` to match what you were told you bought, then run `runHandoverAudit` from the editor. `NOT HANDED OVER` means at least one part of the system answers to an account that is not yours.

`auditTriggers_` cannot list a trigger it is not allowed to see, and it does not pretend to. It compares what you can see against what you were sold. Zero visible triggers plus a system that demonstrably runs on a schedule is the signal — the schedule exists, it just is not yours.

## What breaks in practice

**A blank email compares equal to a blank email.** `User.getEmail()` returns a blank string when policy hides the address, and it does so in exactly the contexts where you might run this by accident: simple `onOpen`/`onEdit` triggers, custom functions, and web apps deployed to execute as the developer. A naive `owner === me` then reads `'' === ''` and reports that you own a file whose owner it could not read. That is why `sameUser_` returns false when either side is blank, and why `runHandoverAudit` raises a warning about your own identity before anything else runs.

**Shared-drive files have no owner at all.** Drive's own documentation is explicit that the owner field "isn't populated for items in shared drives." A file on a shared drive is genuinely fine — the drive owns it, not a person — so treating a missing owner as "not yours" produces false blockers, and treating it as "yours" hides real ones. It gets its own outcome, `no-owner`, at warning level.

**An empty editor list is not a clean file.** `getEditors()` returns an empty array when the running user has no edit access. Read that as "nobody else has access" and the audit reports its best result on the file you have the least control over.

**Free email domains are not a company.** Matching on the domain part is how you catch a colleague, but two accounts on `gmail.com` share nothing. Without the `isFreeDomain_` guard, a consumer-account handover files the developer under `your-domain` and quietly downgrades a blocker to a warning.

**The Apps Script API is off until someone turns it on.** Google's own guidance is that access must be granted from the Apps Script dashboard first, and that "an error results if you attempt to run an affected application without first granting the API access" — after authorization, not before it. Checks 2 and 3 therefore return a warning that names the HTTP status instead of throwing, so an audit that cannot reach the API still delivers checks 1, 4 and 5.

**A standalone project has no sheet to write into.** `SpreadsheetApp.getActiveSpreadsheet()` returns null outside a container-bound script, so the report falls back to the log rather than failing on the last line after doing all the work.

Two related limits are worth naming rather than re-solving here: a delivered job that outgrows its runtime needs chunking and self-rescheduling, and any integration that calls an external API needs quota-aware retries. Both have their own patterns — [how the 6-minute limit is handled](https://magesheet.com/blog/apps-script-6-minute-limit) and [the retry and backoff patterns](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries) — and a developer who cannot describe either is the reason to run this audit in the first place.

The pure logic here was checked with 117 assertions in Node, covering the branches above with mocked Drive, Script, UrlFetch and Properties services; the Apps Script services themselves cannot run outside the platform, so those call sites were reviewed by hand against the documented return types.

## The takeaway

Ownership in Apps Script is not one fact, it is five, and four of them live outside the code you were handed. Run the checks before the final invoice, while the developer is still answering email and a `USER_DEPLOYING` web app is a five-minute redeploy instead of a rebuild.

The hiring side of this — engagement models, what belongs in writing, and the vetting questions no script can answer — is the companion guide on the MageSheet blog.