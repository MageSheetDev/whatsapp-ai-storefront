---
title: "Google Forms Approvals in Apps Script: When the Requester Approves Their Own Request"
seoTitle: "Google Forms Approval Workflow: Fix Self-Approval Bugs"
seoDescription: "Apps Script fixes for a Google Forms approval chain: self-approval, duplicate approvers, unchecked decisions and reminders that never stop."
datePublished: 2026-11-12T18:17:40.191Z
cuid: cmwdveq2b000102p2ckf3fn7o
slug: google-forms-approval-self-approval-apps-script
canonical: https://magesheet.com/blog/google-forms-approval-workflow
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/59fb771e-00a8-47c0-994d-9f72f7022d35.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/4c13b964-68af-42c9-a7dd-6d6b184cb7bd.jpg
tags: automation, google-sheets, google-forms, google-apps-script, magesheet

---

The [Google Forms approval workflow on the MageSheet blog](https://magesheet.com/blog/google-forms-approval-workflow) routes each request through a chain of approvers: a form-submit trigger builds the chain, a web page records the decision, and a daily trigger chases anyone who has not answered. I ran its code in Node against a mock of the Apps Script services, with the kind of data a small team really has, and it did seven things its text does not promise.

The first one: a manager submits a 300 purchase request for herself. The chain is just her, she opens the email, clicks Approve, and the request is APPROVED. Nobody else was asked.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/55fec7cf-e836-4208-a26f-b9ffdf5d79e0.gif align="center")

Below are the seven results side by side, then three replacements for the functions involved. All names, addresses and amounts are invented, and the services are mocked, so this shows what the logic does, not what Google's servers do.

| Input (invented data) | Original code | With the fix |
| --- | --- | --- |
| Sales manager files her own 300 request | Chain is her alone, she approves it, APPROVED | Chain goes to the backup person, she is refused |
| Director is also the Sales manager, 9,000 request | Chain is director, finance, director: two approvals, two emails | Chain is director, finance: one approval, one email |
| Manager cell holds the address plus a trailing space | Nobody can approve, the reply names the address and a space | Approves normally |
| Decision sent as lowercase `reject` | Status APPROVED, reply says "Recorded: reject" | Status REJECTED |
| Decision `banana`, or REJECT with no reason | `banana` approves, the empty-reason REJECT is recorded | `Unknown decision.` and `Please add a reason.` |
| "Last updated" cell unreadable, 30 days | 0 mails, no report | Listed in the owner's digest |
| 5 requests waiting 10 days | 40 reminder mails (see below) | 10 reminders and 5 digests |

## Why the original does this

Three things in the original flow trust input that nothing has checked.

1. **The chain is a list of email addresses that nothing compares.** `onRequestSubmit` maps each role to an address: the manager from the Rules tab, the others from `ROLES`. It never asks whether an address is the requester, whether the same address appears twice, or whether a cell has a trailing space. In a small team the manager and the director can be the same person, and a manager is also someone who submits purchase requests.
2. **The decision arrives from the browser as typed.** `decide(id, token, decision, comment)` is called by `google.script.run`. The Approve and Reject buttons send `'APPROVE'` and `'REJECT'`, and the page refuses an empty reason for a rejection. Both are checks in the browser. On the server, `decision === 'REJECT'` is the only test, so every other string, including `'reject'` and `'banana'`, takes the approval branch, and the reply even echoes the word you sent.
3. **The reminder writes nothing down.** `chaseStale` mails the approver when a request is older than 48 hours and then forgets it did. The next run sees the same age and mails again. A request whose "last updated" cell cannot be read is skipped, because `NaN > 48` is false, so it is neither chased nor reported.

None of this needs an attacker. It needs a small org chart, a pasted cell, and one person on holiday.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/82544370-2b98-48ad-99dd-004149fa7f4a.png align="center")

## Block 1: build the chain, then check it

`buildChain_` takes the addresses in order, the requester and a backup. It lowercases and trims every address. An entry that does not look like an address fails the whole chain: a blank finance cell must not make finance silently disappear from the route. The requester is removed. A repeated address keeps its first position. If nothing is left, the backup is used, unless the backup is the requester too. In that case the result is `ok: false` and the request is stored as `NEEDS_OWNER`, with the script owner mailed instead of an approver.

Add one line to the `ROLES` object from the original post, `backup: 'ops@example.com'`, a person who never submits requests. The notes that `buildChain_` returns are written into the request's history, so the audit trail says why the chain differs from the rules.

```javascript
// Add one line to ROLES from the original post:
//   backup: 'ops@example.com'   (someone who never submits requests)
function normEmail_(s) {
  return String(s || '').trim().toLowerCase();
}

function nowIso_() {
  return new Date().toISOString();
}

function isEmail_(s) {
  return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(s);
}

// emails: one address per step, in order. Returns { ok, chain, notes }.
function buildChain_(emails, requester, backup) {
  const me = normEmail_(requester);
  const seen = {};
  const chain = [];
  const notes = [];
  for (const raw of emails) {
    const a = normEmail_(raw);
    if (!isEmail_(a)) {
      const why = 'bad approver address: "' + raw + '"';
      return { ok: false, chain: [], notes: [why] };
    }
    if (a === me) { notes.push('requester removed: ' + a); continue; }
    if (seen[a]) { notes.push('duplicate removed: ' + a); continue; }
    seen[a] = true;
    chain.push(a);
  }
  if (!chain.length) {
    const b = normEmail_(backup);
    if (!isEmail_(b) || b === me) {
      notes.push('no approver left');
      return { ok: false, chain: [], notes };
    }
    chain.push(b);
    notes.push('backup used: ' + b);
  }
  return { ok: true, chain, notes };
}

function onRequestSubmit(e) {
  const v = e.namedValues;
  const amount = Number(String(v['Amount'][0]).replace(/[^\d.]/g, ''));
  const requester = v['Email Address'][0];
  const emails = CHAINS.find(c => amount <= c.max).steps.map(role =>
    role === 'manager' ? managerFor_(v['Department'][0]) : ROLES[role]);
  const built = buildChain_(emails, requester, ROLES.backup);

  const rec = {
    id: Utilities.getUuid(),
    respRow: e.range.getRow(),
    requester, amount,
    chain: built.chain, stage: 0,
    token: built.ok ? newToken_() : '',
    status: built.ok ? 'PENDING' : 'NEEDS_OWNER',
    history: built.notes.map(n => ({ note: n, at: nowIso_() })),
  };
  appendRecord_(rec);
  if (built.ok) {
    notifyApprover_(rec);
  } else {
    const what = amount + ' from ' + requester;
    notifyOwner_('Request needs an approver: ' + what, built.notes);
  }
}
```

With the Sales manager also set as director, `buildChain_(['director@example.com', 'finance@example.com', 'director@example.com'], 'sam@example.com', 'ops@example.com')` returns the chain `['director@example.com', 'finance@example.com']` and the note `duplicate removed: director@example.com`.

## Block 2: the server decides what a decision is

`checkDecision_` is pure: it takes the record, the visitor's email, the token, the decision and the comment, and returns either `{ ok: true, decision, comment }` or `{ msg }` with the text to show. The order is closed, expired link, wrong person, own request, unknown decision, missing reason. The own-request check is a second line: the chain builder already removes the requester, but a row written before you deployed this still has the old chain.

`decide` keeps the lock and the token rotation from the original. The only change is that it acts on `chk.decision`, the normalised value, and never on the string it was given. The history gets the lowercase address of whoever decided.

```javascript
function checkDecision_(rec, who, token, decision, comment) {
  const no = msg => ({ msg });
  if (!rec || rec.status !== 'PENDING') {
    return no('This request is already closed.');
  }
  if (token !== rec.token) {
    return no('This link has expired: the request has moved on.');
  }
  const approver = normEmail_(rec.chain[rec.stage]);
  const me = normEmail_(who);
  if (!me || me !== approver) {
    return no('This step is assigned to ' + approver + '.');
  }
  if (me === normEmail_(rec.requester)) {
    return no('You cannot decide your own request.');
  }
  const d = String(decision).trim().toUpperCase();
  if (d !== 'APPROVE' && d !== 'REJECT') return no('Unknown decision.');
  const c = String(comment || '').trim().slice(0, 500);
  if (d === 'REJECT' && !c) return no('Please add a reason.');
  return { ok: true, decision: d, comment: c };
}

function decide(id, token, decision, comment) {
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);                      // one decision at a time
  try {
    const rec = findRecord_(id);
    const who = Session.getActiveUser().getEmail();
    const chk = checkDecision_(rec, who, token, decision, comment);
    if (!chk.ok) return chk.msg;

    rec.history.push({ stage: rec.stage, by: normEmail_(who),
      decision: chk.decision, comment: chk.comment, at: nowIso_() });

    if (chk.decision === 'REJECT') {
      rec.status = 'REJECTED';
    } else if (rec.stage + 1 < rec.chain.length) {
      rec.stage += 1;
      rec.token = newToken_();               // old links die here
    } else {
      rec.status = 'APPROVED';
    }
    if (rec.status !== 'PENDING') rec.token = '';

    saveRecord_(rec);
    if (rec.status === 'PENDING') notifyApprover_(rec);
    else notifyRequester_(rec);
    return 'Recorded: ' + chk.decision.toLowerCase() + '. Thank you.';
  } finally {
    lock.releaseLock();
  }
}
```

The Decide.html page stays as it is. Its empty-reason alert is still worth keeping, because it saves a round trip, but it is no longer the check.

## Block 3: remind twice, then tell a person

`planReminders_` takes records and a clock value and returns two lists: who to remind and what is stalled. It counts the reminders already sent at the current stage from the history, and it measures the wait from the later of "last updated" and the last reminder. A request gets at most two reminders, 48 hours apart. After the second and another 48 hours, it moves to the stalled list, and the owner gets one digest mail for all of them, whatever their number. `NEEDS_OWNER` rows and rows whose date cannot be read go straight to that list, so they show up instead of vanishing.

In my test version of `saveRecord_`, every save stamps "last updated" with the current time, so logging a reminder restarts the clock on its own. The planner does not rely on that. One test replaces the stamp with a copy that keeps the old date and gets the same mail counts.

`chaseStale` sends the reminder first and records it second. If the mail call throws, for example when the daily quota is gone, nothing is recorded and the next run tries again.

```javascript
const REMIND_AFTER_H = 48;
const MAX_REMINDERS = 2;

// recs: records as findRecord_ returns them. Pure: no sheet, no mail.
function planReminders_(recs, nowMs) {
  const plan = { remind: [], stalled: [] };
  recs.forEach(r => {
    if (r.status === 'NEEDS_OWNER') { plan.stalled.push(r.id); return; }
    if (r.status !== 'PENDING') return;
    const sent = r.history
      .filter(h => h.reminder && h.stage === r.stage);
    const last = Math.max(new Date(r.updated).getTime(),
      ...sent.map(h => new Date(h.at).getTime()));
    if (isNaN(last)) {                       // unreadable date: say so
      plan.stalled.push(r.id);
      return;
    }
    if ((nowMs - last) / 36e5 < REMIND_AFTER_H) return;
    const more = sent.length < MAX_REMINDERS;
    (more ? plan.remind : plan.stalled).push(r.id);
  });
  return plan;
}

function notifyOwner_(subject, lines) {
  MailApp.sendEmail({
    to: Session.getEffectiveUser().getEmail(),
    subject,
    htmlBody: '<ul><li>' + lines.join('</li><li>') + '</li></ul>',
  });
}

function chaseStale() {
  const rows = SpreadsheetApp.getActive().getSheetByName('Approvals')
    .getDataRange().getValues().slice(1);
  const recs = rows
    .filter(r => r[7] === 'PENDING' || r[7] === 'NEEDS_OWNER')
    .map(r => findRecord_(r[0]));
  const plan = planReminders_(recs, Date.now());
  const byId = id => recs.find(r => r.id === id);

  plan.remind.forEach(id => {
    const rec = byId(id);
    notifyApprover_(rec, { reminder: true });  // mail first, then log
    rec.history.push(
      { reminder: true, stage: rec.stage, at: nowIso_() });
    saveRecord_(rec);
  });
  if (plan.stalled.length) {
    const line = r => r.status + ': ' + r.amount + ' from ' +
      r.requester + ' (on ' + (r.chain[r.stage] || 'nobody') + ')';
    const n = plan.stalled.length;
    notifyOwner_('Approvals nobody is answering: ' + n,
      plan.stalled.map(byId).map(line));
  }
}
```

Nothing here reassigns a stalled request to someone else. That is a decision about who may approve what, and the digest hands it to a person. The original post makes the same call for escalation.

## What the reminders cost in mail

The reminder numbers are the ones that touch a limit. Google's quotas page (read 2026-11-12) lists "Email recipients per day (for example, with MailApp)" at 100 for consumer accounts and 1,500 for Google Workspace. In my mock, five requests waiting on one manager for ten days produced these mails:

| Over 10 days, 5 requests waiting on one approver | Original | With the fix |
| --- | --- | --- |
| Mails to the approver | 40 | 10 |
| Mails to the owner | 0 | 5, one digest a day from day 6 |
| First reminder | day 3 | day 2 |
| Stops after | never | two reminders per stage |

With the original, 20 forgotten requests use 20 of a free account's 100 recipients every day, for as long as they stay open. With the fix they cost at most two mails each, then one digest a day.

## Pitfalls

- **Removing the requester shortens the chain.** If the Sales manager submits a 3,000 request, the chain goes from manager, finance to finance alone. The manager's own sign-off is the one thing that had to go. If your policy wants the next level up in that case, put that person in `ROLES` and in the chain bands; the code does not guess.
- **A** `NEEDS_OWNER` **row is a dead end by design.** The owner gets the mail, fixes the Rules cell, and the requester submits the form again. I did not write a requeue function, and the original row stays as a record.
- **The comment is cut at 500 characters without a warning.** That keeps one cell from growing without limit. If you need longer reasons, change the number in `checkDecision_`.
- **Two reminders is my number.** The original post uses 48 hours and repeats; the cap of two is my choice. Change `REMIND_AFTER_H` and `MAX_REMINDERS` to match how your approvers work.
- `chaseStale` **still reads the status from column 8.** It uses `r[7]` as the original does, so inserting a column in front of status shifts it.

## How this goes stale

- **The stored chain does not follow the Rules tab.** A request keeps the addresses it was given at submission. If the manager leaves while a request waits on her, the reminders go to a mailbox nobody reads, and the digest line `(on ann@example.com)` is where you see it.
- **The backup person changes jobs.** Then every small request a manager submits for herself waits on someone who no longer approves. Keep the backup in the same place you keep the other roles, and check it when `ROLES` changes.
- **The history cell grows.** Each reminder and decision is one JSON entry in one cell. A request chased twice and approved by three people is still small, but do not archive rows by deleting them, because the history is the audit trail the original post relies on.
- **Departments get renamed.** `managerFor_` still throws for a department with no row, as before. The trigger then fails and the request is not recorded at all. This post does not change that.

## What I tested, and what I did not

The code in this post is extracted from the article text and run in Node against 80 tests: address normalisation, the chain builder with the requester, duplicates, bad addresses and the backup, every branch of the decision check, the reminder planner at its 48-hour and two-reminder boundaries, and full runs through the mocked services, including the ten-day reminder run. The same file passes under `TZ=UTC`, `Asia/Istanbul` and `America/Los_Angeles`. 7 of the 80 tests pin what the original code does with the same inputs, which is where the seven results above come from. I then broke the new code on purpose 51 ways, and the tests caught all 51. Three guards look redundant on the happy path: an empty visitor address against an empty chain, a decision entry in the history, and the lowercase address stored in the history. Each has its own test.

I have not run any of this in a live Google account. The mocks do not behave like `MailApp`, `LockService` or `Session`. In particular:

- `appendRecord_`, `findRecord_`, `saveRecord_` and `notifyRequester_` are not in the original post. The versions I tested are mine, and the row layout is the one in its data model.
- I did not test the race the lock protects against. Node runs one thing at a time, and my mock lock does nothing.
- I did not check what `MailApp` does with an address that has a trailing space. The original code stores the cell as it is, and the fix trims it before use.
- I have not called `decide` from a browser console. That an approver can pass any string to a `google.script.run` function is my reading of how the call works, not something I measured in a deployment. The normal Approve and Reject buttons never send a lowercase value.
- Whether `Session.getActiveUser().getEmail()` returns the approver's address depends on the deployment settings the original post describes. I did not touch or test that part.

## The takeaway

An approval chain is only as strict as the list of addresses behind it, and the server only as strict as its own checks, not the page's. Normalise and compare the addresses when you build the chain, accept exactly two decision words on the server, and make a reminder leave a record so that someone is told when it stops working. The full approval workflow, with the routing rules, the confirmation page and the limits, is in the original [Google Forms approval guide on the MageSheet blog](https://magesheet.com/blog/google-forms-approval-workflow). The question of who should approve when the usual person is away is one I raised in a [short #discuss post](https://dev.to/hayrullahkar/which-sheet-automations-get-a-human-approval-step-and-which-just-run-1dao).