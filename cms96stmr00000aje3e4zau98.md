---
title: "Handle UrlFetchApp Quotas &amp; Retries in Apps Script (Backoff, Rate Limits, DLQ)"
seoTitle: "UrlFetchApp Quotas &amp; Retries in Apps Script (Done Right)"
seoDescription: "Production UrlFetchApp in Apps Script: exponential backoff with jitter, per-source rate limiting, fetchAll batching, and a dead-letter queue."
datePublished: 2026-07-31T16:58:53.596Z
cuid: cms96stmr00000aje3e4zau98
slug: handle-urlfetchapp-quotas-amp-retries-in-apps-script-backoff-rate-limits-dlq
canonical: https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/334d27c3-4446-4f95-abb9-cecf2dc47e8a.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/918892ad-8b6b-4e4e-ae17-99b085fdb3d9.jpg
tags: api, javascript, google-sheets, google-apps-script, magesheet

---

A nightly product-enrichment job I built quietly dropped ~300 rows one night. No error, no alert. An upstream API had rate-limited us with a 429, `UrlFetchApp.fetch` threw, the loop died halfway, and the next run started fresh — the skipped rows just never came back.

Every `UrlFetchApp` integration hits this eventually. The fix isn't one clever function; it's four small layers that keep a transient failure from becoming silent data loss. Here's the stack I use, with the code.

## The quotas you'll actually hit

Before the patterns, know the walls. `UrlFetchApp` allows **20,000 calls/day** on consumer accounts and **100,000/day** on Workspace. Each call times out at **60 seconds** and caps at 50 MB each way. And the one that surprises people: concurrent calls per script are limited (undocumented, but roughly **10–20**), so naive parallelism throws "Service invoked too many times."

The math sneaks up on you: 5,000 SKUs × 4 enrichment calls each = 20,000 — your entire daily budget in one run.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/996e1bc0-c231-4746-93ed-e6cc57bd56e5.jpg align="center")

## Pattern 1: exponential backoff with jitter

The foundation. The single most important line is `muteHttpExceptions: true` — without it, a 429 throws before your retry logic can even look at the status code. Retry only on 429 and 5xx; a 400 or 401 won't fix itself, so surface it immediately. The jitter (a random 0–500 ms) stops a batch of scripts from all retrying on the same beat and hammering the API in sync.

```javascript
function fetchWithRetry(url, options = {}, maxAttempts = 5) {
  options.muteHttpExceptions = true; // critical — without this, retries never run

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const response = UrlFetchApp.fetch(url, options);
      const code = response.getResponseCode();

      if (code >= 200 && code < 300) return response;
      if (code === 429 || code >= 500) {
        // Retryable — fall through to backoff
      } else {
        // 4xx other than 429 — do not retry, surface the error
        throw new Error(
          `HTTP ${code}: ${response.getContentText().slice(0, 200)}`
        );
      }
    } catch (err) {
      if (attempt === maxAttempts) throw err;
    }

    // Exponential backoff with jitter: 1s, 2s, 4s, 8s, 16s + random 0-500ms
    const delay = Math.pow(2, attempt - 1) * 1000 + Math.random() * 500;
    Utilities.sleep(Math.min(delay, 30000));
  }

  throw new Error(`fetchWithRetry exhausted ${maxAttempts} attempts on ${url}`);
}
```

I unit-tested the decision logic before shipping — a 429 retries, a 404 does not, and the backoff delay always lands within its band (base to base + 500 ms) and never exceeds the 30-second cap.

## Pattern 2: parallel calls with `fetchAll`

When you have many independent calls, `UrlFetchApp.fetchAll` runs them in one batch instead of a slow serial loop — and it counts against your quota more efficiently. Keep each batch to ~10–20 requests to respect the concurrency limit.

```javascript
function enrichProductsBatch(productIds) {
  const requests = productIds.map(id => ({
    url: 'https://api.openai.com/v1/chat/completions',
    method: 'post',
    headers: { Authorization: 'Bearer ' + OPENAI_KEY },
    contentType: 'application/json',
    payload: JSON.stringify(buildEnrichPrompt(id)),
    muteHttpExceptions: true
  }));

  const responses = UrlFetchApp.fetchAll(requests);

  return productIds.map((id, i) => {
    const code = responses[i].getResponseCode();
    if (code >= 200 && code < 300) {
      return { id, ok: true, data: JSON.parse(responses[i].getContentText()) };
    }
    return { id, ok: false, code, error: responses[i].getContentText() };
  });
}
```

## Pattern 3: per-source rate limiting with CacheService

Backoff reacts to a 429 *after* it happens. Rate limiting stops you from sending it in the first place. This keeps a per-minute counter per API source in `CacheService`, and sleeps until the next minute if you'd exceed it — so you never trip the upstream limit that gets accounts banned.

```javascript
function rateLimitedFetch(source, url, options, maxPerMinute) {
  const cache = CacheService.getScriptCache();
  const key = `rl:${source}:${Math.floor(Date.now() / 60000)}`;
  const count = parseInt(cache.get(key) || '0', 10);

  if (count >= maxPerMinute) {
    Utilities.sleep(60000 - (Date.now() % 60000)); // wait until next minute
    return rateLimitedFetch(source, url, options, maxPerMinute);
  }

  cache.put(key, String(count + 1), 90); // 90s TTL covers the boundary
  return fetchWithRetry(url, options);
}
```

The key is bucketed by minute (`Math.floor(Date.now() / 60000)`), so each new minute starts a fresh count automatically — no cron, no cleanup.

## Pattern 4: a dead-letter queue for exhausted retries

This is the layer that would have saved my 300 rows. When all retries fail, don't throw into the void — write the failed call to a sheet so nothing is lost and you can replay it later.

```javascript
function safeFetch(source, url, options) {
  try {
    return rateLimitedFetch(source, url, options, RATE_LIMITS[source]);
  } catch (err) {
    appendToDeadLetterQueue({
      timestamp: new Date(),
      source,
      url,
      payload: options.payload || '',
      error: err.message,
      attempts: 5
    });
    return null;
  }
}

function appendToDeadLetterQueue(entry) {
  const sheet = SpreadsheetApp.openById(DLQ_SHEET_ID).getSheetByName('failed_calls');
  sheet.appendRow([
    entry.timestamp, entry.source, entry.url,
    entry.payload, entry.error, entry.attempts
  ]);
}
```

`safeFetch` is the only function the rest of your code calls. Rate limiting, retries, and failure capture all sit behind it — your business logic just calls `safeFetch('openai', url, opts)` and gets a response or `null`.

## Anti-patterns that cause outages

*   **Retrying without** `muteHttpExceptions`**.** The exception fires before your code sees the status; your retry loop never runs.
    
*   **Retrying 4xx.** A 400/401/403 is a bug or a bad key — retrying just wastes quota and delays the real fix.
    
*   **Backoff with no jitter.** A fleet of scripts retrying on the exact same 1s/2s/4s beat is a self-inflicted DDoS on the upstream API.
    
*   **A serial loop for thousands of calls.** You'll hit the 6-minute execution limit long before you finish — batch with `fetchAll`.
    
*   **Swallowing failures silently.** Without a dead-letter queue, an exhausted retry is data you'll never know you lost.
    

## Wrap-up

Resilient outbound calls in Apps Script aren't about one function — they're four layers: rate-limit before you send, back off with jitter when you're throttled, batch what you can, and capture what still fails. Wrap them behind a single `safeFetch` and every integration you write inherits it. In steady state that turns a scary "the job died again" into a boring ~0.5% of calls landing in a queue you replay on your own schedule.

The production version — quota-burndown alerts, a DLQ replay job, and idempotency keys for safe retries — is written up on the [MageSheet blog](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries).

Built by the MageSheet team.