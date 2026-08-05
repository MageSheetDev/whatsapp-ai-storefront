---
title: "Claude Prompt Caching in Apps Script: Why Your Cache Read Count Is Zero"
seoTitle: "Claude Prompt Caching in Apps Script: Fix a Zero Hit Rate"
seoDescription: "Prompt caching fails silently. The per-model minimum prefix, the Apps Script bytes that invalidate it, and the log line that shows you which one hit."
datePublished: 2026-08-05T19:44:21.263Z
cuid: cmsghwv7600000aj86svw94sy
slug: claude-prompt-caching-in-apps-script-why-your-cache-read-count-is-zero
canonical: https://magesheet.com/blog/claude-api-apps-script-tool-use-prompt-caching
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/008c724d-015b-474e-b474-e26bde721e6a.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/900a23c0-b488-44ae-ac49-d22eb9d9a083.jpg
tags: ai, automation, google-apps-script, claude, magesheet

---

I added `cache_control` to a WhatsApp handler's system prompt, deployed it, and watched the Apps Script logs for a week. Every single response came back with `cache_read_input_tokens: 0`. No error, no warning, no 400. The API accepted the `cache_control` block on every call and charged me the 1.25x write premium every time, then read nothing back.

Prompt caching fails silently by design. If the prefix is too short, or one byte of it moved, the API just doesn't cache — it does not tell you. And Apps Script has three specific ways to trip that wire that you won't hit in a Python service.

Here's the guard-railed version of the integration, with the checks that turn a silent miss into something you can see.

## The three silent invalidators

**1\. The prefix is under the model's minimum, and the minimum is per-model.** Below the threshold, `cache_control` is accepted and ignored. The numbers as of August 2026:

| Model | Input $/MTok | Output $/MTok | Min cacheable prefix |
| --- | --- | --- | --- |
| `claude-haiku-4-5` | $1 | $5 | 4,096 tokens |
| `claude-sonnet-4-6` | $3 | $15 | 1,024 tokens |
| `claude-sonnet-5` | $3 | $15 | 1,024 tokens |
| `claude-opus-4-7` | $5 | $25 | 2,048 tokens |
| `claude-opus-4-8` | $5 | $25 | 1,024 tokens |
| `claude-opus-5` | $5 | $25 | 512 tokens |

The minimum is not monotonic across generations. A 2,000-token system prompt caches on Sonnet 4.6 and on Opus 4.8, does not cache on Opus 4.7, and does not cache on Haiku 4.5 — the cheap model has the highest bar, at 4,096 tokens. That combination bites hardest in a router: you send cheap-path traffic to Haiku precisely because it's cheap, and that's the one model where your prefix is most likely to fall under the line.

**2\. A volatile byte moved.** Caching is a prefix match on exact bytes, rendered in the order `tools` → `system` → `messages`. Apps Script projects interpolate dynamic values into system prompts constantly — `new Date()`, a `Session.getActiveUser().getEmail()`, a row count pulled from the Sheet at the top of `doPost`. Any of these at the front of the prompt invalidates everything after them on every request.

**3\. The tool list was rebuilt in a different order.** Tools render at position 0, ahead of the system prompt. If your tool array is assembled from an object's keys, or filtered per user, the prefix changes and the system prompt cache goes with it.

## The integration, with the guard rails

The cacheability check runs before the request, so you never pay a write premium for a prefix that can't be read back:

```javascript
// Minimum cacheable prefix per model, in tokens. Below this, the API
// accepts cache_control and silently does not cache.
var CACHE_MIN_TOKENS = {
  'claude-haiku-4-5': 4096,
  'claude-sonnet-4-6': 1024,
  'claude-sonnet-5': 1024,
  'claude-opus-4-7': 2048,
  'claude-opus-4-8': 1024,
  'claude-opus-5': 512
};

// Rough guard rail for English text, not billing truth. Confirm the
// real number with usage.cache_read_input_tokens on a live call.
function estimateTokens(text) {
  return Math.ceil(text.length / 3.6);
}

function isCacheablePrefix(model, prefixText) {
  var min = CACHE_MIN_TOKENS[model];
  if (min === undefined) return false;
  return estimateTokens(prefixText) >= min;
}

function pickRoute(intent, options) {
  options = options || {};
  if (options.forceModel) return options.forceModel;
  if (intent === 'classification' || intent === 'extraction') {
    return 'claude-haiku-4-5';
  }
  if (intent === 'audit' || intent === 'planning') {
    return 'claude-opus-4-8';
  }
  return 'claude-sonnet-4-6';
}
```

`pickRoute` returns a model ID, not a route label. That matters because the model ID is the key into `CACHE_MIN_TOKENS` — routing and cacheability have to agree, and keeping them in one vocabulary is what makes that checkable in a test.

The request layer reads the response code before parsing, and logs the cache counters on every call:

```javascript
function isRetryable(statusCode) {
  return statusCode === 408 || statusCode === 429 || statusCode >= 500;
}

function backoffMs(attempt) {
  return Math.min(1000 * Math.pow(2, attempt), 16000);
}

function callClaude(model, systemText, messages, tools) {
  var system = [{ type: 'text', text: systemText }];
  if (isCacheablePrefix(model, systemText)) {
    system[0].cache_control = { type: 'ephemeral' };
  }

  var payload = {
    model: model,
    max_tokens: 4096,
    system: system,
    messages: messages
  };
  if (tools && tools.length) payload.tools = tools;

  for (var attempt = 0; attempt < 4; attempt++) {
    var res = UrlFetchApp.fetch(
      'https://api.anthropic.com/v1/messages', {
        method: 'post',
        contentType: 'application/json',
        headers: {
          'x-api-key': PropertiesService.getScriptProperties()
            .getProperty('ANTHROPIC_API_KEY'),
          'anthropic-version': '2023-06-01'
        },
        payload: JSON.stringify(payload),
        muteHttpExceptions: true
      });

    var code = res.getResponseCode();
    var body = res.getContentText();

    if (code === 200) {
      var data = JSON.parse(body);
      Logger.log('cache_read=%s cache_write=%s input=%s',
        data.usage.cache_read_input_tokens,
        data.usage.cache_creation_input_tokens,
        data.usage.input_tokens);
      return data;
    }
    if (!isRetryable(code) || attempt === 3) {
      throw new Error('Claude ' + code + ': ' + body.slice(0, 300));
    }
    Utilities.sleep(backoffMs(attempt));
  }
}
```

Two things there are not decoration. `muteHttpExceptions: true` means a 429 or a 529 returns normally instead of throwing, so `getResponseCode()` is the only thing standing between you and `JSON.parse`\-ing an error object into a variable your tool loop will later index as `data.content`. And the `Logger.log` line is the entire diagnostic: `cache_read_input_tokens` staying at 0 across repeated calls with the same prefix is the signal that one of the three invalidators is active.

The retry here is deliberately minimal — four attempts, capped at 16 seconds, 7 seconds of total sleep across three retries, which stays well inside the 6-minute trigger ceiling. For the rate limiter and the dead-letter queue that belong around it in a real webhook, see the [UrlFetchApp quotas and retries](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries) breakdown.

The tool loop keeps `tool_use_id` paired with its result and turns a thrown handler into a structured error the model can recover from:

```javascript
function runToolLoop(model, systemText, userText, tools, handlers) {
  var messages = [{ role: 'user', content: userText }];

  for (var turn = 0; turn < 5; turn++) {
    var res = callClaude(model, systemText, messages, tools);
    messages.push({ role: 'assistant', content: res.content });

    if (res.stop_reason !== 'tool_use') {
      return res.content
        .filter(function (b) { return b.type === 'text'; })
        .map(function (b) { return b.text; })
        .join('\n');
    }

    var results = res.content
      .filter(function (b) { return b.type === 'tool_use'; })
      .map(function (block) {
        var out;
        try {
          out = handlers[block.name](block.input);
        } catch (err) {
          return {
            type: 'tool_result',
            tool_use_id: block.id,
            content: 'Error: ' + err.message,
            is_error: true
          };
        }
        return {
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(out)
        };
      });

    messages.push({ role: 'user', content: results });
  }
  throw new Error('Tool loop did not converge in 5 turns');
}
```

All tool results for one assistant turn go back in a **single** user message. Splitting them across several messages is accepted by the API and quietly trains the model to stop issuing parallel tool calls.  
  

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/e25a3d8a-cab7-4123-bb43-ccd0eeac0d9b.png align="center")

*Diag<s>r</s>am: the request path, with the cacheability check gating* `cache_control` *before the fetch and the response-code branch gating* `JSON.parse` *after it.*

## What the caching is actually worth

Take a handler at 3,000 messages a day with a 7,000-token cached prefix on Sonnet 4.6 at $3 per million input tokens. That's 630M input tokens a month:

*   Uncached: 630 MTok x $3 = **$1,890/month**
    
*   Cached, assuming 90% of calls land inside a warm 5-minute window: 63 MTok of writes at 1.25x ($236) plus 567 MTok of reads at 0.1x ($170) = **$406/month**
    

The 90% figure is the assumption doing the work, and it's a property of your traffic shape, not of the API. Bursty webhook traffic clears it easily; a trigger that fires once an hour never gets a single hit on the 5-minute cache. Measure it from `cache_read_input_tokens` before you put it in a business case.

## Pitfalls

**The 1-hour cache break-even is three requests, not two.** A 5-minute write costs 1.25x base input and a read costs 0.1x, so two requests beat two uncached ones (1.35x vs 2x). A 1-hour write costs **2x**, so two requests lose (2.1x vs 2x) and you only come out ahead at three. Reach for `ttl: "1h"` when the gap between requests exceeds five minutes, not because it sounds safer.

**Four breakpoints, and automatic caching eats one.** A request is capped at four explicit `cache_control` markers. If you also set top-level automatic caching, that consumes a slot.

**Switching models mid-conversation throws the cache away.** Caches are model-scoped. A router that escalates from Haiku to Opus mid-thread pays a fresh cold write on the escalation, on the full accumulated history. Budget for it, or keep the escalation on a fresh short prompt.

**Apps Script cannot consume streaming.** `UrlFetchApp.fetch` returns the whole body or nothing; it has no SSE path. This is fine in practice, because the 30-second `doPost` ceiling means you couldn't hold the connection anyway. If you need token-by-token output for a UI, that belongs in a separate streaming-capable service, with Apps Script out of the request path and the Sheet still the system of record.

**Keep the router's return value and the cache table in the same vocabulary.** The version of this router I first wrote returned labels like `'claude-haiku'` and mapped them to model IDs one layer down. That layer is where the two drift apart: the router says `claude-haiku`, the cache table has a key for `claude-haiku-4-5`, and `isCacheablePrefix` falls through to `undefined` and returns `false` for every request. Caching silently switches itself off across the whole cheap path, and nothing errors. Returning the model ID directly makes the mismatch impossible, and a one-line test over every intent catches it if someone adds a route later.

I ran the pure logic in Node before shipping it — `estimateTokens`, `isCacheablePrefix`, `pickRoute`, `isRetryable`, `backoffMs` are all plain functions with no `UrlFetchApp` in them. 32 assertions, including the boundary case where a 3,682-character prefix estimates to 1,023 tokens and correctly fails the 1,024-token Sonnet minimum, and the assertion that every intent `pickRoute` can return has a key in `CACHE_MIN_TOKENS`. That last one is the test that would have caught the label bug.

## The takeaway

Prompt caching has no failure signal, so the integration has to supply one. Check the prefix against the model's minimum before you set `cache_control`, log `cache_read_input_tokens` on every call, and treat a run of zeros as a bug rather than a cost you haven't optimized yet.

The production version — with the six-layer webhook architecture, the dead-letter queue, and the full router — is on the [MageSheet blog](https://magesheet.com/blog/claude-api-apps-script-tool-use-prompt-caching).

Built by the MageSheet team.