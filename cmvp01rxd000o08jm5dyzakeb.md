---
title: "WhatsApp Response Time in Google Sheets: The Webhook Can't Tell Your Bot From Your Agent"
seoTitle: "WhatsApp First Response Time in Google Sheets, Bots Excluded"
seoDescription: "The Cloud API status webhook does not say who replied. Tag each message when you send it and measure first response in business minutes."
datePublished: 2026-10-26T08:33:19.786Z
cuid: cmvp01rxd000o08jm5dyzakeb
slug: whatsapp-response-time-in-google-sheets-the-webhook-can-t-tell-your-bot-from-your-agent
canonical: https://magesheet.com/blog/whatsapp-conversation-data-google-sheets
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/550f529c-fd82-47d0-add6-5f0f34ab6e99.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/b3abaa49-c7d0-42dc-bc06-d9ecd3fe118e.jpg
tags: automation, whatsapp, google-apps-script, googlesheets, magesheet

---

Our guide to [syncing WhatsApp conversation data into Google Sheets](https://magesheet.com/blog/whatsapp-conversation-data-google-sheets) says the first response time must not count automated replies. An auto-acknowledgement is not a response, and if you count it your median drops to a few seconds.

I ran the webhook code from that guide, unchanged, against fake Meta events: a customer writes, a bot acknowledges after 2 seconds, an agent replies after 10 minutes. The sheet got 8 rows. Six of them are `status:sent`, `status:delivered` and `status:read`, and nothing in them tells the bot's message from the agent's. The first outbound row is the bot, at +2 s. A first-response formula over that tab reports 2 seconds.

The rule in the guide is right. The data it collects cannot apply it. This post fixes the data: every outgoing message carries a tag saying who wrote it, Meta hands the tag back in the status webhook, and the response time is measured from tagged rows in business minutes.

## Why the status webhook cannot tell a bot from a person

With the WhatsApp Cloud API, Meta POSTs to your webhook when a customer writes (a `messages` entry) and when something you sent changes state (a `statuses` entry). Meta's status reference lists the fields of a status: `id`, `status`, `timestamp`, `recipient_id`, and a few optional ones for groups, pricing and errors. There is no message text and no sender. A status says "message wamid.X was delivered to this number at this time". It does not say who wrote message wamid.X.

One optional field matters here. The reference describes `biz_opaque_callback_data` as included "only if the business set it" when sending. Whatever string you attach to an outgoing message comes back on every status of that message. That is the hook: tag the message when you send it, read the tag when the status arrives.

The three routes from the original guide still decide whether you can do this at all:

|  | Business app (phone) | Inbox platform | Cloud API direct |
| --- | --- | --- | --- |
| Automatic sync | No | Usually | Yes |
| Who wrote each reply | Not available | Whatever the vendor exports | Yours to tag when you send |
| Who holds the data | The phone | The vendor | You |

Only the third route lets you set that field on every message. The rest of this post assumes it.

## Step 1: a webhook that keeps the tag

The receiver is the guide's `doGet`/`doPost` with four changes. Column H stores `biz_opaque_callback_data`. Message bodies are not stored unless you flip `STORE_TEXT`, because the report needs timestamps, not transcripts. `doGet` also checks `hub.mode`. And the write happens under `LockService`, because two deliveries that arrive together would both read the same `getLastRow()` and write to the same row.

```javascript
const VERIFY_TOKEN = 'pick-a-long-random-string';
const STORE_TEXT = false;  // message bodies are personal data

function doGet(e) {
  // Meta's one-time subscription handshake
  const p = e.parameter;
  if (p['hub.mode'] === 'subscribe' &&
      p['hub.verify_token'] === VERIFY_TOKEN) {
    return ContentService.createTextOutput(p['hub.challenge']);
  }
  return ContentService.createTextOutput('forbidden');
}

function doPost(e) {
  const body = JSON.parse(e.postData.contents);
  const rows = [];
  (body.entry || []).forEach(entry => {
    (entry.changes || []).forEach(change => {
      const v = change.value || {};
      const ours = (v.metadata || {}).display_phone_number || '';
      (v.messages || []).forEach(m => {
        const text = STORE_TEXT && m.text ? m.text.body : '';
        rows.push([toDate_(m.timestamp), 'inbound', m.from,
          ours, m.type, text, m.id, '']);
      });
      (v.statuses || []).forEach(s => {
        // A status has no text and no sender. Column H is the
        // tag we attached when we sent the message (block 2).
        rows.push([toDate_(s.timestamp), 'status:' + s.status,
          s.recipient_id, ours, '', '', s.id,
          s.biz_opaque_callback_data || '']);
      });
    });
  });
  if (rows.length) {
    const lock = LockService.getScriptLock();
    lock.waitLock(20000);  // two deliveries must not share a row
    try {
      const sh = SpreadsheetApp.getActive()
        .getSheetByName('Messages');
      sh.getRange(sh.getLastRow() + 1, 1, rows.length, 8)
        .setValues(rows);
    } finally {
      lock.releaseLock();
    }
  }
  return ContentService.createTextOutput('EVENT_RECEIVED');
}

function toDate_(unixSeconds) {
  return new Date(Number(unixSeconds) * 1000);  // seconds, not ms
}
```

WhatsApp timestamps are Unix seconds. Without the `* 1000` every row lands in January 1970.

## Step 2: send every message through one function

The tag only exists if you set it. So every outgoing message, from a bot, a template sender or an agent's reply tool, goes through `sendText`, and the caller has to say who it is. A typo such as `'Human'` throws before any request is made, so a mislabelled sender cannot quietly become an untagged reply.

```javascript
const GRAPH = 'https://graph.facebook.com/v21.0/';
const PHONE_NUMBER_ID = '123456789012345';  // Meta app dashboard

// Every outgoing message goes through here, tagged with who
// wrote it: 'human' for an agent, 'auto' for bots and templates.
function sendText(to, text, sender) {
  if (sender !== 'human' && sender !== 'auto') {
    throw new Error('sender must be human or auto: ' + sender);
  }
  const token = PropertiesService.getScriptProperties()
    .getProperty('WA_TOKEN');
  const url = GRAPH + PHONE_NUMBER_ID + '/messages';
  const res = UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: 'Bearer ' + token },
    payload: JSON.stringify({
      messaging_product: 'whatsapp',
      to: to,
      type: 'text',
      text: { body: text },
      biz_opaque_callback_data: sender,
    }),
    muteHttpExceptions: true,
  });
  if (res.getResponseCode() !== 200) {
    throw new Error('send failed: ' + res.getContentText());
  }
  return JSON.parse(res.getContentText()).messages[0].id;
}
```

This is the real cost of the approach. A reply that does not go through `sendText` has no tag, and the report treats it as no reply at all. If an agent answers from the WhatsApp Business app on their phone or from another inbox, the sheet will say the customer is still waiting. I chose that failure on purpose. A wrong "still waiting" shows up in the hourly email and someone corrects it. A wrong "answered in 2 seconds" hides in a median and nobody ever sees it.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/015aced8-b599-40a4-901d-e3199d9a52b2.png align="center")

## Step 3: messages into conversations, minutes into business minutes

Raw rows are not a report. An hourly trigger rolls the Messages tab into a Conversations tab: one row per customer per conversation, with the time it opened, the first human reply and the minutes in between. The rules are in the code:

- A conversation opens with an inbound message. 24 hours of silence from the customer starts a new one.
- Meta can deliver the same event twice. A row is skipped if its message id and kind were already seen.
- A message is dated by its first status of any kind. If the `sent` webhook is lost, `delivered` still dates the reply. A late `read` on a message from yesterday is not a new reply.
- A message with a `failed` status is never a reply, even if a `sent` arrived first.
- Only messages tagged `human` count.

```javascript
const GAP_HOURS = 24;  // this much silence starts a new conversation
const HOURS = { start: 9, end: 18, days: [1, 2, 3, 4, 5] };
const TEAM_EMAIL = 'sales@example.com';

// Minutes of [from, to] inside business hours, script time zone.
function businessMinutes(from, to, h) {
  let total = 0;
  const day = new Date(from);
  day.setHours(0, 0, 0, 0);
  while (day < to) {
    if (h.days.indexOf(day.getDay()) !== -1) {
      const open = new Date(day);
      open.setHours(h.start);
      const close = new Date(day);
      close.setHours(h.end);
      const a = Math.max(open, from);
      const b = Math.min(close, to);
      if (b > a) total += b - a;
    }
    day.setDate(day.getDate() + 1);
  }
  return Math.round(total / 60000);
}

// Messages rows -> one row per customer per conversation:
// A number, B opened, C first human reply, D inbound count,
// E business minutes to reply, F outcome, G key
function buildConversations(rows) {
  const seen = {};
  const failed = {};
  const events = [];
  rows.forEach(r => {
    const k = r[6] + '|' + r[1];
    if (seen[k]) return;  // Meta delivered this event twice
    seen[k] = true;
    if (r[1] === 'status:failed') failed[r[6]] = true;
    events.push(r);
  });
  events.sort((a, b) => a[0] - b[0]);
  const open = {};
  const list = [];
  const dated = {};
  events.forEach(r => {
    const t = r[0];
    const who = r[2];
    if (r[1] === 'inbound') {
      let c = open[who];
      if (!c || t - c.last > GAP_HOURS * 3600000) {
        c = { who: who, opened: t, last: t, n: 0, reply: null };
        open[who] = c;
        list.push(c);
      }
      c.n++;
      c.last = t;
      return;
    }
    // A message is dated by its first status of any kind, so a
    // lost 'sent' webhook does not lose the reply, and a late
    // 'read' on an old message is not a new reply.
    if (dated[r[6]]) return;
    dated[r[6]] = true;
    const human = r[7] === 'human' && !failed[r[6]];
    const c = open[who];
    if (human && c && !c.reply) c.reply = t;
  });
  return list.map(c => [
    c.who, c.opened, c.reply || '', c.n,
    c.reply ? businessMinutes(c.opened, c.reply, HOURS) : '',
    '', c.who + '|' + c.opened.getTime(),
  ]);
}

// The agent's Outcome (F) survives a rebuild, matched by key.
// Rows whose messages were deleted (retention) stay as they were.
function mergeConversations(fresh, old) {
  const outcome = {};
  const rebuilt = {};
  old.forEach(r => { if (r[5]) outcome[r[6]] = r[5]; });
  fresh.forEach(r => {
    r[5] = outcome[r[6]] || '';
    rebuilt[r[6]] = true;
  });
  const kept = old.filter(r => r[6] && !rebuilt[r[6]]);
  return kept.concat(fresh);
}

function rebuildConversations() {
  const ss = SpreadsheetApp.getActive();
  const msgs = ss.getSheetByName('Messages').getDataRange()
    .getValues().slice(1);
  const sh = ss.getSheetByName('Conversations');
  const old = sh.getDataRange().getValues().slice(1);
  const rows = mergeConversations(buildConversations(msgs), old);
  if (sh.getLastRow() > 1) {
    sh.getRange(2, 1, sh.getLastRow() - 1, 7).clearContent();
  }
  if (rows.length) sh.getRange(2, 1, rows.length, 7).setValues(rows);
  return rows;
}

// Hourly trigger: rebuild, then email chats that have waited
// more than an hour of business time with no human reply.
function alertUnanswered() {
  const late = overdue(rebuildConversations(), new Date(), 60);
  if (!late.length) return;
  MailApp.sendEmail(TEAM_EMAIL,
    late.length + ' WhatsApp chats waiting over an hour',
    late.map(r => r[0] + ' since ' + r[1]).join('\n'));
}

function overdue(rows, now, mins) {
  return rows.filter(r => r[1] !== '' && r[2] === '' &&
    businessMinutes(r[1], now, HOURS) > mins);
}
```

`businessMinutes` is why the midnight enquiry stops wrecking the week. A message at 23:00 on Tuesday answered at 08:55 on Wednesday is 0 business minutes. A message at 17:30 on Friday answered at 09:15 on Monday is 45. It works day by day with `setHours`, so a weekend that crosses a daylight saving change still gives the right number.

`mergeConversations` handles two things a rebuild would otherwise break. The agent's Outcome dropdown in column F is matched back by key (number plus opening time) instead of being wiped by `clearContent`. And once you delete old message rows for retention, the conversation rows built from them stay as they were, so last quarter's report does not shrink when the transcripts go.

## The dashboard formulas, checked one by one

With column E holding business minutes and C holding the first human reply, the formulas change too. Reading the original four against the tab layout above, each has a problem:

| Metric | Original formula | Problem | Formula for this tab |
| --- | --- | --- | --- |
| Leads this month | `=COUNTIFS(Conversations!B:B, ">="&EOMONTH(TODAY(),-1)+1)` | Counts conversations. A customer who comes back after 24 hours is counted twice, though the guide defines leads as distinct numbers. | `=IFERROR(COUNTUNIQUE(FILTER(Conversations!A2:A, Conversations!B2:B>=EOMONTH(TODAY(),-1)+1)), 0)` |
| Median first response | `=MEDIAN(FILTER(Conversations!E:E, Conversations!E:E>0))` | `>0` drops replies of 0 business minutes, and business hours make those common: every night message answered before opening. | `=MEDIAN(FILTER(Conversations!E2:E, Conversations!E2:E<>""))` |
| Win rate | `=COUNTIF(Conversations!F:F,"Won") / COUNTA(Conversations!F:F)` | `COUNTA(F:F)` also counts the header cell. | `=COUNTIF(Conversations!F2:F,"Won") / COUNTA(Conversations!F2:F)` |
| Unanswered over 1 hour | `=COUNTIFS(Conversations!E:E, ">60")` | Counts chats answered late. An unanswered chat has an empty E and never matches. | `=COUNTIFS(Conversations!C2:C, "", Conversations!B2:B, "<"&(NOW()-1/24))` |

The last formula uses wall-clock time, so it counts a Saturday message as waiting by Saturday afternoon. `alertUnanswered` uses business minutes for the email, which is the number people act on.

## Pitfalls

- **The phone app is invisible to this report.** Messages sent from the WhatsApp Business app on a phone do not go through your `sendText`, so they carry no tag. Decide which channel agents reply from before you publish response times.
- **Templates are business-initiated and billed.** Outside the 24-hour customer service window you can only send approved templates. Tag them `auto`. They are follow-ups, not responses, and our [24-hour window guide](https://magesheet.com/blog/whatsapp-automation-tool) covers what gets rejected outside it.
- **The webhook URL is public.** Meta signs each POST in an `X-Hub-Signature-256` header, but `doPost(e)` in Apps Script cannot read request headers, so this receiver cannot verify the signature. Anyone who learns the `/exec` URL can post fake rows. Keep the URL out of shared documents and treat a sudden spike of unknown numbers as a sign it leaked.
- **The sender tag is a short string.** It goes into a field meant for your own callback data. Keep it to `human` or `auto`; per-agent names belong in your own sheet, keyed by message id.

## How this drifts once it runs

- **The access token expires.** The temporary token on the Meta app dashboard is meant for testing and does not last. When it lapses, `sendText` throws on every call, so the failure is loud, but nothing goes out until `WA_TOKEN` holds a long-lived system user token.
- **The Graph version ages out.** `v21.0` in the URL is pinned. Meta retires old versions on a schedule. Put a calendar reminder next to the constant.
- **The Messages tab grows forever.** Eight columns per event and three or four events per outgoing message add up. `rebuildConversations` reads the whole tab every hour. Archive or delete message rows on the same schedule as your retention rule. The merge keeps the conversation rows.
- **New agents bring new habits.** The day someone answers from their phone, their chats show as unanswered. The hourly email is where you will notice it first.

## What I tested, and what I did not

42 tests in Node against fakes of `SpreadsheetApp`, `LockService`, `UrlFetchApp` and `MailApp`, run under four time zones (UTC, Europe/Istanbul, Europe/London, America/New_York): business minutes across nights, weekends and the 25 October daylight saving change, duplicate deliveries, failed sends, lost `sent` webhooks, late read receipts, 23 and 25 hour gaps, outcome carry-over, and the webhook rows feeding the rollup end to end. Then 24 deliberate breakages of the code. 23 were caught. The 24th removed a check that a reply must come after the conversation opened, and no test failed because rows are sorted by time first, so that check could never fire. I deleted it.

I also ran the original guide's `doPost` unchanged against the same fake events, which is where the 8 rows and the +2 s come from.

What I have not measured:

- A live WhatsApp Cloud API number. I have not seen a real status webhook carry `biz_opaque_callback_data` back. The behaviour comes from Meta's reference, not from a run.
- Whether `recipient_id` on a status always matches `from` on the inbound message for the same customer. The rollup assumes it does. Check a few rows from your own number before you trust a report.
- How often Meta delivers the same event twice. The dedupe handles any number of repeats, but I do not know the real rate.
- That `Date.getHours()` in Apps Script uses the script's time zone. The tests set the time zone of the Node process instead.
- The four formulas in a real sheet. I checked them by reading, not by running them in Google Sheets.
- `LockService` under real concurrent deliveries. The test only shows that the write sits inside the lock.

## The takeaway

The status webhook tells you that a message was sent, not who sent it. If you want a response time that means something, tag every outgoing message at the moment you send it and count only the tagged ones.

The original guide, with the three routes to a sheet compared in full, is on the [MageSheet blog](https://magesheet.com/blog/whatsapp-conversation-data-google-sheets).