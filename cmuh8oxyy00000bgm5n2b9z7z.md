---
title: "The 24-Hour Window Decides Whether Your WhatsApp Reply Sends or Gets Rejected"
seoTitle: "WhatsApp Automation Tool: 3 Ways and What Each Costs"
seoDescription: "The three ways to automate WhatsApp compared: SaaS, Cloud API direct, or a Google Sheets build you own. Costs, ownership and the ToS rule."
datePublished: 2026-09-25T17:33:25.912Z
cuid: cmuh8oxyy00000bgm5n2b9z7z
slug: whatsapp-automation-tool
canonical: https://magesheet.com/blog/whatsapp-automation-too
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/2c3ef58b-be0f-4b35-ad98-209378649dd2.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/68f6bf85-b413-43ca-af63-e8962a8a9dea.jpg
tags: automation, google-sheets, whatsapp, google-apps-script, magesheet

---

There are exactly three ways to automate WhatsApp: rent a SaaS platform, wire up Meta's Cloud API yourself, or build on Google Sheets and Apps Script. I have built the third one, and what surprised me is how little that choice changes about the part that actually breaks.

All three send through the same official Cloud API, and that API applies the same rule to all three. You may send a free-form message only within 24 hours of the customer's most recent message to you. Outside that window only an approved template goes through, and anything else comes back as a re-engagement rejection.

So whichever tool you picked, some code is answering one question before every outbound message: is the window still open? When that code is wrong the symptom is not a crash. It is a reply the customer never receives, or a template charge on a message that could have been free.

Here is the function, and the three ways I have watched it get the answer wrong.

## The obvious version is wrong in two directions

```javascript
// Wrong. Do not ship this.
var last = new Date(message.timestamp);
if (Date.now() - last < 24 * 60 * 60 * 1000) {
  sendFreeForm(text);
} else {
  sendTemplate(name);
}
```

Meta's webhook hands you `timestamp` as a string of epoch **seconds**: `"1753276800"`. `new Date("1753276800")` does not read that as seconds, or as anything else. It is an Invalid Date, every arithmetic result off it is `NaN`, and `NaN < 86400000` evaluates to `false`. The branch falls to the template side and stays there. The automation looks healthy, customers get answered, and every single reply is billed as a template.

That is the forgiving version of the bug, because it fails in the direction that still delivers. The unforgiving version is the same mistake reversed: storing `Date.now()` in milliseconds on one path and epoch seconds on another, so elapsed time comes out a thousand times too small and the window reads as open long after it closed. Then the free-form replies start bouncing and nothing in your logs says why.

Both bugs come from the same hole. The timestamp arrives in more than one shape and nothing normalises it.

## One normaliser, one decision

```javascript
/** Normalise any timestamp shape to epoch SECONDS. NaN if unusable. */
function toEpochSeconds(value) {
  if (value instanceof Date) {
    var ms = value.getTime();
    return isNaN(ms) ? NaN : Math.floor(ms / 1000);
  }
  if (typeof value === 'number') {
    if (!isFinite(value) || value <= 0) return NaN;
    // Sheets and Date.now() hand back milliseconds; the webhook hands
    // back seconds. Past year 2286 in seconds is really milliseconds.
    return value > 1e11 ? Math.floor(value / 1000) : Math.floor(value);
  }
  if (typeof value === 'string') {
    var trimmed = value.trim();
    if (trimmed === '') return NaN;
    if (/^\d+$/.test(trimmed)) return toEpochSeconds(Number(trimmed));
    var parsed = Date.parse(trimmed);
    return isNaN(parsed) ? NaN : Math.floor(parsed / 1000);
  }
  return NaN;
}

var SERVICE_WINDOW_SECONDS = 24 * 60 * 60;

/** How may the next outbound message be sent? */
function replyMode(lastInbound, now) {
  var last = toEpochSeconds(lastInbound);
  var current = toEpochSeconds(now);
  if (isNaN(last) || isNaN(current)) {
    return { mode: 'template', reason: 'unreadable timestamp',
             secondsLeft: 0 };
  }
  var elapsed = current - last;
  if (elapsed < 0) {
    return { mode: 'template', reason: 'timestamp in the future',
             secondsLeft: 0 };
  }
  if (elapsed >= SERVICE_WINDOW_SECONDS) {
    return { mode: 'template', reason: 'service window closed',
             secondsLeft: 0 };
  }
  return { mode: 'free_form', reason: 'service window open',
           secondsLeft: SERVICE_WINDOW_SECONDS - elapsed };
}
```

Two properties are worth naming because they are doing real work.

It fails closed. Every path that cannot produce a number returns `template`. That is not a happy accident of `NaN` comparisons, it is an explicit branch, because a template sent inside an open window costs money and a free-form sent outside one does not arrive at all.

It has one unit. Seconds, everywhere, decided at the boundary. The millisecond confusion cannot happen downstream because nothing downstream sees a millisecond.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/1d305017-9c07-4cce-851e-9302f9e2233c.png align="center")

## Record the event that actually resets the window

The window resets on the customer's message, not on yours. This is the detail that decides whether the rest of the code can ever be right, and it is easy to write the wrong one because the outbound path is the one you are debugging.

```javascript
var SHEET_NAME = 'Contacts';

/** Meta posts every inbound message here. */
function doPost(e) {
  var payload = JSON.parse(e.postData.contents);
  var change = payload.entry[0].changes[0].value;
  var message = (change.messages || [])[0];
  if (!message) return ContentService.createTextOutput('ignored');
  recordInbound(message.from, message.timestamp);
  return ContentService.createTextOutput('ok');
}

function findRow(sheet, waId) {
  var ids = sheet.getRange('A2:A').getValues();
  for (var i = 0; i < ids.length; i++) {
    if (String(ids[i][0]) === String(waId)) return i + 2;
  }
  return 0;
}

function recordInbound(waId, timestamp) {
  var sheet = SpreadsheetApp.getActive().getSheetByName(SHEET_NAME);
  var row = findRow(sheet, waId);
  // Store raw epoch seconds. A formatted date cell comes back as a Date
  // in the spreadsheet's timezone, which is a different number once the
  // file is opened from somewhere else.
  if (!row) sheet.appendRow([waId, Number(timestamp)]);
  else sheet.getRange(row, 2).setValue(Number(timestamp));
}
```

Now the send path has one job, and it reports which mode it used rather than guessing later.

```javascript
function sendReply(waId, text, templateName) {
  var sheet = SpreadsheetApp.getActive().getSheetByName(SHEET_NAME);
  var row = findRow(sheet, waId);
  var lastInbound = row ? sheet.getRange(row, 2).getValue() : null;
  var decision = replyMode(lastInbound, new Date());
  var props = PropertiesService.getScriptProperties();

  var body = decision.mode === 'free_form'
    ? { messaging_product: 'whatsapp', to: waId, type: 'text',
        text: { body: text } }
    : { messaging_product: 'whatsapp', to: waId, type: 'template',
        template: { name: templateName, language: { code: 'en' } } };

  var response = UrlFetchApp.fetch(
    'https://graph.facebook.com/v21.0/' +
      props.getProperty('PHONE_ID') + '/messages',
    {
      method: 'post',
      contentType: 'application/json',
      headers: { Authorization: 'Bearer ' + props.getProperty('WA_TOKEN') },
      payload: JSON.stringify(body),
      muteHttpExceptions: true
    });

  if (response.getResponseCode() >= 300) {
    throw new Error(decision.mode + ' rejected: ' +
      response.getContentText());
  }
  return decision;
}
```

## Pitfalls

**Writing the timestamp on send instead of on receive.** If `recordInbound` runs from the outbound path, every message you send refreshes the window in your own sheet. The sheet then says open forever, and Meta rejects the free-form messages you keep sending after the real window closed. The rejection text names the recipient, not your bug, so this one hides well.

**Formatting the timestamp column as a date.** The moment that column is a date rather than a plain number, `getValue()` hands back a `Date` built in the spreadsheet's timezone. The normaliser above absorbs it, which is the point of having one. Without a normaliser the same sheet gives different answers depending on which timezone the file was last opened in.

**Testing only the middle of the window.** One hour in, everything passes. The boundary is where the decision changes: at exactly 24 hours the window is closed, not open. `elapsed >= SERVICE_WINDOW_SECONDS` rather than `>`, and a test that pins both sides of it.

**Trusting a timestamp that is ahead of you.** Clock skew and a customer message stamped in the future produce a negative elapsed time, which is smaller than the window and therefore reads as open. That is why the negative branch is separate rather than folded into the comparison.

Those four are the ones I have actually hit. The unit tests that pin them are thirteen assertions over two pure functions, and they run in Node without any Apps Script service, because `toEpochSeconds` and `replyMode` take values rather than reading the sheet themselves.

## Where this lives in each of the three options

| Option | Who owns this function | What you change when it is wrong |
| --- | --- | --- |
| SaaS platform | The vendor | A support ticket, and you wait |
| Cloud API direct | You | Your own code, deployed by you |
| Google Sheets and Apps Script | You | The script bound to your sheet |

This is the smallest honest version of the build-versus-buy question. Renting means somebody else already wrote this function and you cannot read it. Owning means it is yours to get wrong and yours to fix in an afternoon.

The full comparison of the three routes, with costs, ownership, and the Terms of Service line that rules out the unofficial senders, is on the [MageSheet blog](https://magesheet.com/blog/whatsapp-automation-tool).