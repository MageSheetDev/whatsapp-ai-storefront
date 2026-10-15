---
title: "Email From a Google Sheet: Claim the Row Before You Send It"
seoTitle: "Google Sheets SMTP Email: Send Once, Log Every Bounce"
seoDescription: "Apps Script can't speak SMTP. Send via MailApp or Mailjet, claim each row before sending, and join bounces back on a CustomID you control."
datePublished: 2026-10-15T13:38:08.827Z
cuid: cmv9l3ehw000002moevl83bqi
slug: send-email-google-sheets-smtp-apps-script
canonical: https://magesheet.com/blog/send-email-google-sheets-smtp
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/088c7625-7ada-4035-aeb4-20b59fa6b8e8.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/54eaf8a1-5fb0-4a96-a440-cb7f81185ad5.jpg
tags: email, automation, google-apps-script, googlesheets, magesheet

---

Every guide to sending email from a Google Sheet ends with the same safety rule: write SENT next to each row after you mail it, so a rerun skips what already went out. The rule is right, and on its own it does not stop a double send. It stops the next run from repeating the last one. It does nothing about two runs that are reading the sheet at the same time.

That happens more easily than it sounds. Run the function that installs the hourly trigger a second time and you now have two identical triggers, firing in the same minute, both reading an Outbox where nothing says SENT yet. Click Run in the editor while the trigger is halfway through and you get the same thing by hand. Every customer on the list hears from you twice, and the sheet ends up looking perfectly tidy.

There is a second gap on the way back in. The usual webhook that logs bounces into the sheet keys each event by the provider's MessageID, and the MessageID in Mailjet's own sample event is an integer larger than JavaScript can hold exactly. Parse it and its neighbours, and three different messages come back as the same number, so a bounce can land on the wrong row.

This post fixes both, with code you can paste into a sheet today.

## Why there is no SMTP setting to fill in

Apps Script has no raw socket access, so it cannot open a connection to smtp.yourprovider.com on port 587 the way a Node or Python script can. That is why "connect Google Sheets to SMTP" guides never show a box for your SMTP host and password: there is nowhere to put it. Two routes work instead.

*   **MailApp**, which sends through Google using the account that owns the script. No credentials, and a hard daily cap: Google documents 100 recipients a day on a consumer Gmail account and 1,500 on Workspace.
    
*   **Your provider's HTTPS send endpoint**, called with UrlFetchApp. Mailjet, Mailgun, SendGrid, Brevo and Postmark all offer one next to their SMTP port, and it does the same job over HTTPS with your own authenticated domain.
    

The code below starts on MailApp and moves to Mailjet by swapping one function. Everything else, including the safety logic, stays the same.

## A row has to claim itself before it sends

The fix for overlapping runs is to make each row move through states, and to commit the move before the email goes out.

| Status | What it means | What the next run does |
| --- | --- | --- |
| (blank) | not sent yet | sends it |
| SENDING | claimed, outcome not recorded | skips it, a person checks |
| SENT | the sender accepted it | skips it |
| FAILED: reason | the sender rejected it | skips it until someone clears the cell |

Two things make the claim stick. A script lock means a second run that starts while the first is busy gives up straight away instead of reading the sheet. And SpreadsheetApp.flush() right after writing SENDING pushes that value to the sheet before the send, rather than leaving it in the batch of pending writes Apps Script normally holds until the run ends.

The Outbox tab has seven columns: Email, Name, Subject, Body, Status, SentAt, CustomID. Subject and Body can contain placeholders such as {{Name}}, and any column header works as a placeholder.

```javascript
// Outbox: Email | Name | Subject | Body | Status | SentAt | CustomID
const OUTBOX = 'Outbox';

function sendFromSheet() {
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(1000)) return; // a previous run is still going
  try {
    const sheet = SpreadsheetApp.getActive().getSheetByName(OUTBOX);
    const rows = sheet.getDataRange().getValues();
    const col = indexer_(rows.shift());
    // throws here if a column the script writes is missing
    ['Email', 'Status', 'SentAt', 'CustomID'].forEach(col);
    for (let i = 0; i < rows.length; i++) {
      const row = rows[i];
      const r = i + 2; // sheet row number
      const to = String(row[col('Email')]).trim();
      if (!to || row[col('Status')] !== '') continue;
      const id = Utilities.getUuid();
      mark_(sheet, r, col, 'SENDING', '', id);
      SpreadsheetApp.flush(); // commit the claim before sending
      try {
        deliver_({
          to: to,
          subject: fill_(row[col('Subject')], row, col, false),
          htmlBody: fill_(row[col('Body')], row, col, true),
          name: 'Your Company',
          customId: id,
        });
        mark_(sheet, r, col, 'SENT', new Date(), id);
      } catch (err) {
        if (err.message === 'QUOTA') {
          mark_(sheet, r, col, '', '', ''); // nothing went out
          break;
        }
        mark_(sheet, r, col, 'FAILED: ' + err.message, '', id);
      }
    }
  } finally {
    lock.releaseLock();
  }
}

function deliver_(msg) {
  if (MailApp.getRemainingDailyQuota() < 1) {
    throw new Error('QUOTA');
  }
  MailApp.sendEmail({
    to: msg.to,
    subject: msg.subject,
    htmlBody: msg.htmlBody,
    name: msg.name,
  });
}

function indexer_(header) {
  return name => {
    const i = header.indexOf(name);
    if (i < 0) throw new Error('No column named ' + name);
    return i;
  };
}

function fill_(template, row, col, html) {
  return String(template).replace(/\{\{(\w+)\}\}/g, (m, key) => {
    const v = String(row[col(key)]);
    return html ? escape_(v) : v;
  });
}

function escape_(s) {
  return s.replace(/&/g, '&amp;').replace(/</g, '&lt;')
    .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

function mark_(sheet, r, col, status, sentAt, id) {
  sheet.getRange(r, col('Status') + 1).setValue(status);
  sheet.getRange(r, col('SentAt') + 1).setValue(sentAt);
  sheet.getRange(r, col('CustomID') + 1).setValue(id);
}
```

A few decisions in there are deliberate.

The loop skips anything that is not blank. A row stuck on SENDING means the run died between claiming it and recording the result, and from inside the sheet there is no way to know whether that one email went out. Sending it again is a guess. Leaving it for a person is not. I would rather explain one missing email than apologise for a duplicate.

Quota is checked inside deliver\_, per message, and an exhausted quota hands the row back. When MailApp reports zero recipients left, the row goes back to blank and the loop stops, so the next run picks it up where this one ended. A quota error is not a failure of that row and should not be recorded as one.

The column lookup throws instead of returning -1. With a plain indexOf, a renamed Status header gives -1, every row reads undefined, and nothing looks wrong. Here the four columns the script writes are checked before any row is touched, so a missing SentAt stops the run with nothing sent, and a typo such as {{Nmae}} in a template becomes a FAILED row with the reason in the cell.

Placeholders in the HTML body are escaped. A customer whose name field contains an ampersand or an angle bracket gets their name printed, not markup.

## Moving to a provider is one function

When you outgrow the daily cap, or start emailing people who did not ask to hear from you, you want a sending domain with SPF and DKIM. Store the keys in Project Settings, Script properties, never in the code: a key typed into a script inside a shared spreadsheet belongs to everyone with edit access.

This replaces deliver\_ from the MailApp version. Nothing else changes.

```javascript
// Replaces deliver_ from the MailApp version. Nothing else changes.
function deliver_(msg) {
  const props = PropertiesService.getScriptProperties();
  const key = props.getProperty('MJ_KEY');
  const secret = props.getProperty('MJ_SECRET');
  const auth = Utilities.base64Encode(key + ':' + secret);

  const res = UrlFetchApp.fetch('https://api.mailjet.com/v3.1/send', {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: 'Basic ' + auth },
    payload: JSON.stringify({
      Messages: [{
        From: {
          Email: 'billing@yourdomain.com',
          Name: msg.name,
        },
        To: [{ Email: msg.to }],
        Subject: msg.subject,
        HTMLPart: msg.htmlBody,
        CustomID: msg.customId, // comes back on every event
      }],
    }),
    muteHttpExceptions: true,
  });

  const code = res.getResponseCode();
  if (code === 429) throw new Error('QUOTA'); // stop, retry next run
  if (code >= 300) {
    const text = res.getContentText().slice(0, 200);
    throw new Error('Mailjet ' + code + ' ' + text);
  }
}
```

Mailjet does not answer with a quota number the way MailApp does. It answers 429 when you go over your plan's rate, so that becomes the same QUOTA signal: the row goes back to blank and waits for the next run. Any other code at 300 or above becomes FAILED with the first 200 characters of Mailjet's error, which is usually enough to see whether it was a bad address, an unverified sender or a wrong key. If you send in bulk, the [UrlFetchApp quotas and retries guide](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries) covers what to do about 429s beyond waiting an hour.

The line that matters for the next section is CustomID. That is a value I generate and write to the row before sending, and Mailjet repeats it on every event it reports for that message.

## Why the join key is CustomID and not MessageID

The common way to log bounces back is to store ev.MessageID from each webhook event. In Mailjet's own sample event the value is 19421777835146490, sent as a bare JSON number. JSON.parse turns every number into a JavaScript double, and doubles hold integers exactly only up to 2^53, which is 9007199254740992. Mailjet's sample is about twice that.

At that size the gap between neighbouring doubles is 4. I parsed the sample ID and the three after it in Node:

| Sent by Mailjet | After JSON.parse |
| --- | --- |
| 19421777835146490 | 19421777835146490 |
| 19421777835146491 | 19421777835146492 |
| 19421777835146492 | 19421777835146492 |
| 19421777835146493 | 19421777835146492 |

Three different messages, one key. Nothing throws. Messages sent close together get IDs close together, so a bounce for one customer gets attributed to whichever of them you match first. Google Sheets stores numbers as doubles too, so writing the raw value to a cell does not save you. Mailjet also sends the ID as a string in mj\_message\_id on some events, but its docs mark that field deprecated.

CustomID avoids the whole problem. It is a string I chose, it already sits on the row, and Mailjet's sample events carry it back as CustomID.

## The webhook, with a key the script can actually check

Apps Script can be the URL your provider posts events to. Deploy the project as a web app, executing as you, with access set to Anyone. That URL is now public, so it needs a check.

Most guides say to verify the provider's signature. In Apps Script that is often impossible: doPost(e) never sees the request headers, and SendGrid's signed event webhook and Postmark's basic auth both travel in headers. Mailjet's own guide recommends basic auth credentials in the webhook URL, and those arrive as an Authorization header too. What doPost does see is the query string. So I put a long random secret in Script properties as HOOK\_KEY and register the webhook as the /exec URL with ?key= and that secret appended.

```javascript
// Events: Time | Event | Email | CustomID
function doPost(e) {
  const want = PropertiesService.getScriptProperties()
    .getProperty('HOOK_KEY');
  if (!want || (e.parameter || {}).key !== want) {
    return reply_({ ok: false }); // still HTTP 200, see below
  }

  const data = JSON.parse(e.postData.contents);
  const events = Array.isArray(data) ? data : [data];
  const rows = events.map(ev => [
    ev.time ? new Date(ev.time * 1000) : new Date(),
    String(ev.event || ''),
    String(ev.email || ''),
    String(ev.CustomID || ''), // join key, never ev.MessageID
  ]);
  if (!rows.length) return reply_({ ok: true, n: 0 });

  const lock = LockService.getScriptLock();
  lock.waitLock(20000); // two batches must not get the same row
  try {
    const sheet = SpreadsheetApp.getActive().getSheetByName('Events');
    sheet.getRange(sheet.getLastRow() + 1, 1, rows.length, 4)
      .setValues(rows);
  } finally {
    lock.releaseLock();
  }
  return reply_({ ok: true, n: rows.length });
}

function reply_(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

The lock is there because two batches arriving at once would both read getLastRow() and write over each other. A single appendRow per event avoids that but costs one call per row. Writing the whole batch in one setValues inside a lock keeps it to one write.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/e3a4881e-a62a-44d1-a327-a1bd41e14ba1.png align="center")

Then the Events tab joins back to the Outbox on CustomID with an ordinary lookup, and a bounce lands next to the row that caused it.

## Pitfalls I would check before trusting this

**The wrong key still gets HTTP 200.** A web app cannot set a status code. The rejection is in the body, which the provider ignores. It does no harm, because nothing is written, but it also means a typo in the registered URL fails silently. Send one test event from the provider's dashboard and look for the row.

**The /exec URL answers a POST with a redirect.** doPost runs before the redirect, so the row is written, but Mailjet's webhook guide says anything other than 200 OK is retried every 30 seconds for up to 24 hours. Check the webhook log after the first real events. If it shows retries, the Events tab will hold duplicates, and the clean fix is to ignore an event whose CustomID, event and time are already present.

**Two installers, two triggers.** If the trigger is created in code, have the installer delete existing triggers for sendFromSheet first. The lock makes the duplicate harmless, but a duplicate you do not know about will hide the next problem.

**A row with several addresses spends several units of quota.** MailApp counts recipients, not messages. One cell with three comma-separated addresses costs three of your 100.

**SENDING rows need a person.** Put a filter view on Status = SENDING and look at it after any run that ended with an error. Most of the time the answer is in the provider's log: if the message is there, change the row to SENT; if not, clear it.

## The one thing to keep

A SENT column records what already happened. It cannot stop two runs from acting on the same row at once, and a number that came through JSON.parse cannot be trusted as an identifier past 2^53. Claim the row and flush before sending, take a lock around it, and join provider events on an ID you wrote yourself.

The original post, with the MailApp versus provider comparison and when each one is the right call, is on the [MageSheet blog](https://magesheet.com/blog/send-email-google-sheets-smtp).