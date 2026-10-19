---
title: "MailApp's Daily Limit Counts Recipients: Routing Sheet Email Between MailApp and a Provider"
seoTitle: "MailApp Quota Counts Recipients: Route Sheet Email Right"
seoDescription: "MailApp counts recipients, not emails, and resets 24 hours after the first send. Route each email from a Sheet to MailApp or a provider by rule."
datePublished: 2026-10-19T04:28:23.681Z
cuid: cmver7tnx000008l888xhedvy
slug: google-sheets-email-route-mailapp-provider
canonical: https://magesheet.com/blog/google-sheets-email-integrations
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/fa22d28e-c815-4854-b19f-396a9c163986.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/5155522d-ca5d-48df-a7bb-b2bc684f1408.jpg
tags: automation, google-sheets, email-marketing, google-apps-script, magesheet

---

A Google Sheet that sends email usually ends up with two senders. `MailApp` for internal alerts, because it needs no account and costs nothing. A provider such as Mailjet for anything a customer sees, because that mail should come from a verified domain with bounce data. My [overview of the email options for Google Sheets](https://magesheet.com/blog/google-sheets-email-integrations) ends with exactly that recommendation.

Running both is easy. Deciding which one each message takes is where the mistakes are. Google's quota table for `MailApp` is short, and two lines in it are easy to misread: the daily limit counts recipients, not emails, and the day does not start at midnight.

This post is the routing function I use: one `sendRouted_(msg)` that looks at every recipient and the remaining quota, and sends through `MailApp` or the provider. Provider setup, webhooks and logging are in the Mailjet and SMTP guides linked at the end, so they are not repeated here.

## What the quota actually counts

These are the lines from Google's Apps Script quota page that matter for sending:

| Quota | Consumer (gmail.com) | Google Workspace |
| --- | --- | --- |
| Email recipients per day | 100 | 1,500 |
| Email recipients per day within domain | 100 | 2,000 |
| Email recipients per message | 50 | 50 |

Three things in Google's wording shape the code.

**It counts recipients.** The `MailApp` reference says "Quotas are based on the number of email recipients." A digest sent to five managers uses five, not one. A loop that counts sends against 100 runs out long before it expects to.

**The day is rolling.** The quota page says quotas "reset 24 hours after the first request", not at midnight. A script that sends at 9:00 and assumes a fresh quota at 00:01 is wrong for nine hours.

**The remaining number is a snapshot.** `MailApp.getRemainingDailyQuota()` "returns the number of recipients you can send emails to for the rest of the day", and the reference adds that the value "is valid for the current execution and might vary between executions." So read it before every send, not once at the top of the run.

## Who counts as internal

Routing starts with the recipients. `To`, `Cc` and `Bcc` all count, because a customer in `Bcc` is still a customer receiving mail from your personal address.

```javascript
const INTERNAL_DOMAINS = ['yourcompany.com'];
const MAX_PER_MESSAGE = 50; // Apps Script: 50 recipients per message

// "Ana <ana@x.com>, b@y.com" -> ['ana@x.com', 'b@y.com']
function addresses_(field) {
  return String(field || '')
    .split(/[,;]/)
    .map(part => {
      const m = part.match(/<([^>]+)>/);
      return (m ? m[1] : part).trim().toLowerCase();
    })
    .filter(a => /^[^@\s<>"]+@[^@\s<>"]+\.[a-z]{2,}$/.test(a));
}

// Exact domain or a subdomain of it. A bare endsWith would
// also match notyourcompany.com.
function isInternal_(address) {
  const domain = address.slice(address.lastIndexOf('@') + 1);
  return INTERNAL_DOMAINS.some(d =>
    domain === d || domain.endsWith('.' + d));
}

// Every unique recipient of one message, To + Cc + Bcc.
function recipients_(msg) {
  const all = addresses_(msg.to)
    .concat(addresses_(msg.cc), addresses_(msg.bcc));
  return all.filter((a, i) => all.indexOf(a) === i);
}
```

Two details in `isInternal_` matter. It takes the domain after the last `@`, so `yourcompany.com@gmail.com` is a Gmail address, not yours. And it matches the exact domain or a real subdomain with a leading dot. A plain `domain.endsWith('yourcompany.com')` also matches `notyourcompany.com`, which would send customer mail from your own account.

## The routing rule

The rule itself is a pure function, so it can be tested without Apps Script. It gets the message and the remaining quota and returns a route with a reason.

```javascript
const RESERVE = 5; // keep a few for the script's own error alerts

// Pure: which route one message takes, and why.
// remaining = MailApp.getRemainingDailyQuota(), a recipient count.
function planRoute_(msg, remaining) {
  const recips = recipients_(msg);
  if (recips.length === 0) {
    return { route: 'skip', why: 'no valid address' };
  }
  if (recips.length > MAX_PER_MESSAGE) {
    return { route: 'skip',
      why: recips.length + ' recipients, max 50' };
  }
  const outside = recips.filter(a => !isInternal_(a));
  if (outside.length > 0) {
    return { route: 'provider',
      why: 'customer mail: ' + outside.length + ' outside' };
  }
  if (recips.length > remaining - RESERVE) {
    return { route: 'provider',
      why: 'quota: ' + recips.length + ' needed, ' +
        remaining + ' left' };
  }
  return { route: 'mailapp', why: recips.length + ' internal' };
}
```

The order of the checks is the policy.

*   **Anyone outside the company means the provider.** One customer in `Cc` turns an internal alert into customer mail. The provider route keeps it off your personal address and gives you delivery data.
    
*   **Internal mail goes through** `MailApp` **only if every recipient fits**, with a small reserve left over. The reserve keeps room for the script's own failure alerts, which are usually sent with `MailApp` too.
    
*   **More than 50 recipients is skipped, not split.** `MailApp` allows 50 recipients per message, and Mailjet's Send API v3.1 rejects more than 50 addresses in `To`, `Cc` or `Bcc`. Splitting a list silently is a decision someone should make on purpose.
    

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/26cea944-1646-4293-8c14-bad88fada50f.png align="center")

## Sending, with a fallback

The last block does the sending. The provider half is a Mailjet v3.1 call, using the `From`, `To`, `Cc`, `Bcc` and `HTMLPart` fields from Mailjet's v3.1 documentation. Any provider with an HTTP send endpoint fits in the same place.

```javascript
// Mailjet Send API v3.1. Any provider's HTTP send fits here.
function providerSend_(msg) {
  const p = PropertiesService.getScriptProperties();
  const list = f => addresses_(f).map(a => ({ Email: a }));
  const m = { From: { Email: p.getProperty('FROM_EMAIL') },
    To: list(msg.to), Subject: msg.subject, HTMLPart: msg.html };
  if (msg.cc) m.Cc = list(msg.cc);
  if (msg.bcc) m.Bcc = list(msg.bcc);
  const auth = Utilities.base64Encode(
    p.getProperty('MJ_KEY') + ':' + p.getProperty('MJ_SECRET'));
  const res = UrlFetchApp.fetch('https://api.mailjet.com/v3.1/send', {
    method: 'post', contentType: 'application/json',
    headers: { Authorization: 'Basic ' + auth },
    payload: JSON.stringify({ Messages: [m] }),
    muteHttpExceptions: true,
  });
  if (res.getResponseCode() >= 300) {
    throw new Error('Mailjet ' + res.getResponseCode());
  }
}

// Ask for the quota before every send: Google says the value is
// only valid for the current execution, and it counts recipients.
function sendRouted_(msg) {
  const plan = planRoute_(msg, MailApp.getRemainingDailyQuota());
  if (plan.route === 'mailapp') {
    try {
      MailApp.sendEmail({ to: msg.to, cc: msg.cc, bcc: msg.bcc,
        subject: msg.subject, htmlBody: msg.html });
      return plan;
    } catch (err) {
      if (!/too many times|quota/i.test(err.message)) throw err;
      plan.route = 'provider';
      plan.why = 'MailApp refused: ' + err.message;
    }
  }
  if (plan.route === 'provider') providerSend_(msg);
  return plan;
}
```

`sendRouted_` reads the quota on every call. If `MailApp` still refuses because the quota changed between the read and the send, the error is caught only when its message mentions the quota, and the message goes through the provider instead. Any other `MailApp` error, such as an invalid address, is thrown. Hiding those would turn a data problem into a silent reroute.

Write `plan.route` and `plan.why` into the row next to the send status. When someone asks why a message came from the provider and not from your address, the answer is in the sheet: `customer mail: 1 outside`, or `quota: 3 needed, 6 left`.

## How I tested it

`MailApp` and `UrlFetchApp` do not run outside Google, so I tested the three blocks in Node against fakes. The fake `MailApp` subtracts one from the quota per recipient, the way Google describes it. The fake `UrlFetchApp` records the Mailjet request.

26 checks pass. They cover display names and `;` separators, a quoted name with a comma in it, look-alike and subdomain addresses, a customer hidden in `Bcc`, duplicates across `To` and `Cc`, a three-recipient message against a quota of seven, the reserve boundary, 51 recipients, the Mailjet payload and auth header, a quota error from `MailApp`, a non-quota error from `MailApp`, a 400 from Mailjet, and three sends in a row where the second one runs out of quota.

To check that the tests can fail, I broke the code fourteen ways. Matching with `endsWith` and no dot failed 1 check. Reading only the `To` field failed 3. Counting one per message instead of one per recipient failed 2. Dropping the reserve failed 3. Removing the outside-recipient rule failed 7. Reading the quota once per run failed 1. Catching every `MailApp` error failed 1. The other seven, from lower-casing to the 50 limit to the HTTP status check, failed at least 1 each.

Two things I did not test, because they need a live Google account. `getRemainingDailyQuota()` returns one number, while the quota table lists separate limits for all recipients and for recipients within your domain, and I have not checked which one the number reflects on Workspace. I also have not confirmed that a `MailApp` call rejected for quota sends nothing. Google says the script throws and stops, which is what the fallback relies on.

## Pitfalls I would check first

**Counting emails instead of recipients.** Any loop with `if (sent < 100)` is counting the wrong thing. Ask `getRemainingDailyQuota()` and compare it with the recipient count.

**Waiting for midnight.** The quota comes back 24 hours after the first request. A run that failed for quota should try again later, not at a fixed time.

**Checking only** `To`**.** Customers in `Cc` or `Bcc` receive the same mail from the same address.

**A suffix match on the domain.** `endsWith('yourcompany.com')` is true for `notyourcompany.com`. Compare the exact domain or `'.' + domain`.

**Catching every error as quota.** Only reroute on the quota message. Everything else should stop the row and show up in the log.

## The one thing to keep

Decide the route per message from its recipients and the quota at that moment, and write the reason down. `MailApp` stays the free channel for colleagues, the provider handles customers, and the sheet records which was used. Provider setup with the full event round trip is in my [Mailjet guide](https://magesheet.com/blog/google-sheets-mailjet-integration).

Built by the MageSheet team.