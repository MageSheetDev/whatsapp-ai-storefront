---
title: "One callModel() for Gemini, OpenAI and Claude in Apps Script"
seoTitle: "One AI Router for Gemini, OpenAI and Claude in Apps Script"
seoDescription: "Route each task to the cheapest model that can do it. A tested Apps Script router for Gemini, OpenAI and Claude, with a per-call cost ledger."
datePublished: 2026-09-01T15:52:53.512Z
cuid: cmtiuj7cd00000ahybqqxdn3w
slug: cross-provider-ai-router-apps-script
canonical: https://magesheet.com/blog/google-workspace-ai-automation
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/39d42aa6-7409-402c-a284-001b3f64db6e.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/6c60bb77-184c-4b0f-8f8d-e6cea6057b75.jpg
tags: ai, google-sheets, google-apps-script, gemini, magesheet

---

A lead-classification job I built ran every inbound row through the flagship model because that was the model already wired up. 60,000 calls a month, about 800 input and 60 output tokens each. On `claude-opus-5` list rates that arithmetic comes to $330 a month. The same calls on `gemini-2.5-flash-lite` come to $6.24. The prompt didn't change, the sheet didn't change, the accuracy on that particular task didn't measurably change — only which model answered.

Getting from $330 to $6.24 means one call site can reach more than one provider. The usual framing is that this is easy, because from Apps Script all three are the same shape: a `UrlFetchApp.fetch()` to a REST endpoint with your prompt in the body and your key in a header. That's true at the wire level. At the code level three specific things differ, and each one fails in a way that doesn't raise.

Here's the router I ended up with, and the three places it has to know the difference.

## What actually differs between the three

Not the transport. These:

**Where the system prompt goes.** Gemini takes it as a top-level `systemInstruction` object. OpenAI takes it as a message with `role: "system"` at the front of `messages`. Claude takes it as a top-level `system` string, and a `system` *message* inside `messages` is a different feature with different rules. Send a Gemini-shaped body to OpenAI and the system prompt is silently absent — no error, just a model that ignores instructions you're certain you sent.

**Where the key goes.** Gemini reads `x-goog-api-key`. OpenAI reads `Authorization: Bearer`. Claude reads `x-api-key` and additionally requires an `anthropic-version` header that the other two have no equivalent for.

**Whether the output cap is optional.** `generationConfig.maxOutputTokens` and `max_completion_tokens` are both optional. Claude's `max_tokens` is required — a helper that treats it as optional works against two providers and returns a 400 against the third.

## The money layer

One object holds the routing vocabulary and the prices, so a route can never name a model the cost function doesn't know:

```javascript
// One registry: model id -> provider + list price per 1M tokens,
// in USD. These are published list rates read in September 2026;
// re-check them before you put a number in a business case.
// This object is the only place you edit when a price moves.
var MODELS = {
  'gemini-2.5-flash-lite': { p: 'gemini', in: 0.10, out: 0.40 },
  'gemini-3.1-pro':        { p: 'gemini', in: 2.00, out: 12.00 },
  'gpt-5.6-luna':          { p: 'openai', in: 0.20, out: 1.20 },
  'gpt-5.6-sol':           { p: 'openai', in: 5.00, out: 30.00 },
  'claude-haiku-4-5':      { p: 'claude', in: 1.00, out: 5.00 },
  'claude-opus-5':         { p: 'claude', in: 5.00, out: 25.00 }
};

// Task -> model id. It returns the id, not a label like 'cheap',
// so the string that routes is the same string that keys MODELS.
function pickModel(task) {
  if (task === 'classify' || task === 'tag') {
    return 'gemini-2.5-flash-lite';
  }
  if (task === 'extract') return 'gpt-5.6-luna';
  if (task === 'reply') return 'claude-haiku-4-5';
  if (task === 'audit') return 'claude-opus-5';
  return 'gemini-2.5-flash-lite';
}

// Cache reads bill at 0.1x input and Claude cache writes at
// 1.25x. Summing all three input counters at the base rate
// charges a warm route ten times what its reads actually cost.
function costUsd(model, u) {
  var e = MODELS[model];
  if (!e) throw new Error('Unknown model: ' + model);
  var m = 1000000;
  return (u.inTok / m) * e.in +
         (u.cacheWrite / m) * e.in * 1.25 +
         (u.cacheRead / m) * e.in * 0.10 +
         (u.outTok / m) * e.out;
}
```

`pickModel` returning a model id rather than a label is deliberate, and it's a bug I have already paid for once: a router that returns `'cheap'` and maps it to an id one layer down puts the routing vocabulary and the pricing vocabulary in two places, and they drift. Returning the id makes a mismatch impossible to write, and a one-line test over every task catches it if someone adds a route later. I hit the same class of failure with a model-id-versus-label mismatch in [Claude prompt caching](https://magesheet.com/blog/claude-api-apps-script-tool-use-prompt-caching), where the consequence was caching silently switching itself off.

The prices above are list rates for the models I actually route to, published as of September 2026. Providers move them — Google marks its current Flash rates as effective through the end of 2026, and OpenAI's top-tier rate has a promotional discount with a stated end date — so treat the table as a value you maintain, not a constant.

## The request layer

`buildRequest` is the only function that knows any provider's request shape, and it takes the key as an argument instead of reading `PropertiesService` itself. That one choice is what makes the whole layer testable outside Apps Script:

```javascript
var ENDPOINTS = {
  gemini: 'https://generativelanguage.googleapis.com/v1beta/models/',
  openai: 'https://api.openai.com/v1/chat/completions',
  claude: 'https://api.anthropic.com/v1/messages'
};

function buildRequest(model, systemText, userText, key, maxTokens) {
  var entry = MODELS[model];
  if (!entry) throw new Error('Unknown model: ' + model);

  if (entry.p === 'gemini') {
    return {
      url: ENDPOINTS.gemini + model + ':generateContent',
      headers: { 'x-goog-api-key': key },
      payload: {
        systemInstruction: { parts: [{ text: systemText }] },
        contents: [
          { role: 'user', parts: [{ text: userText }] }
        ],
        generationConfig: { maxOutputTokens: maxTokens }
      }
    };
  }

  if (entry.p === 'openai') {
    return {
      url: ENDPOINTS.openai,
      headers: { Authorization: 'Bearer ' + key },
      payload: {
        model: model,
        messages: [
          { role: 'system', content: systemText },
          { role: 'user', content: userText }
        ],
        max_completion_tokens: maxTokens
      }
    };
  }

  return {
    url: ENDPOINTS.claude,
    headers: {
      'x-api-key': key,
      'anthropic-version': '2023-06-01'
    },
    payload: {
      model: model,
      system: systemText,
      messages: [{ role: 'user', content: userText }],
      max_tokens: maxTokens
    }
  };
}
```

Note that the Gemini branch never puts `model` in the payload — it goes in the URL path. A body that carries `model` to Gemini is not an error there either; the field is simply ignored, which is why a copy-pasted OpenAI body appears to work right up until you notice every response came from whichever model the URL named.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/73adb526-0e61-4b94-9de3-4af22ccb5618.png align="center")

## The response layer

Reading the answer back is where the three shapes diverge hardest, and where the guards matter more than the happy path:

```javascript
function readText(model, json) {
  var p = MODELS[model].p;

  if (p === 'gemini') {
    var cand = (json.candidates || [])[0];
    if (!cand) return '';
    // content.parts is absent, not empty, when the turn stopped
    // on SAFETY or MAX_TOKENS. parts[0].text would throw here.
    var parts = (cand.content || {}).parts || [];
    return parts.map(function (x) {
      return x.text || '';
    }).join('');
  }

  if (p === 'openai') {
    var msg = ((json.choices || [])[0] || {}).message || {};
    // content is null on a refusal or a tool call, not ''.
    return msg.content || '';
  }

  // Claude returns an array of blocks and block 0 is not always
  // the text one, so find by type instead of indexing.
  var blocks = json.content || [];
  var out = '';
  for (var i = 0; i < blocks.length; i++) {
    if (blocks[i].type === 'text') out += blocks[i].text;
  }
  return out;
}

// Normalizes three usage shapes into one billable row. Cached
// input is kept apart because it does not bill at the same rate.
function readUsage(model, json) {
  var p = MODELS[model].p;
  var u = json.usageMetadata || json.usage || {};

  if (p === 'gemini') {
    return {
      inTok: u.promptTokenCount || 0,
      // Thinking tokens bill as output but are reported in their
      // own field, outside candidatesTokenCount.
      outTok: (u.candidatesTokenCount || 0) +
              (u.thoughtsTokenCount || 0),
      cacheWrite: 0,
      cacheRead: u.cachedContentTokenCount || 0
    };
  }

  if (p === 'openai') {
    var det = u.prompt_tokens_details || {};
    var cached = det.cached_tokens || 0;
    return {
      inTok: (u.prompt_tokens || 0) - cached,
      outTok: u.completion_tokens || 0,
      cacheWrite: 0,
      cacheRead: cached
    };
  }

  return {
    inTok: u.input_tokens || 0,
    outTok: u.output_tokens || 0,
    cacheWrite: u.cache_creation_input_tokens || 0,
    cacheRead: u.cache_read_input_tokens || 0
  };
}

function callModel(task, systemText, userText) {
  var model = pickModel(task);
  var provider = MODELS[model].p;
  var key = PropertiesService.getScriptProperties()
    .getProperty('KEY_' + provider.toUpperCase());
  var req = buildRequest(model, systemText, userText, key, 1024);

  var res = UrlFetchApp.fetch(req.url, {
    method: 'post',
    contentType: 'application/json',
    headers: req.headers,
    payload: JSON.stringify(req.payload),
    muteHttpExceptions: true
  });

  var code = res.getResponseCode();
  var body = res.getContentText();
  if (code !== 200) {
    throw new Error(provider + ' ' + code + ': ' +
                    body.slice(0, 200));
  }

  var json = JSON.parse(body);
  var usage = readUsage(model, json);
  SpreadsheetApp.getActive().getSheetByName('AI Ledger')
    .appendRow([new Date(), task, model, provider,
                usage.inTok, usage.outTok,
                costUsd(model, usage)]);
  return readText(model, json);
}
```

The three usage branches are the point of the whole exercise. Without them you have a routing story and no way to check it; with them every call writes a dated row naming the task, the model, the token counts and the dollar figure, and the routing claim becomes something you read off a sheet instead of something you assert.

`prompt_tokens` on the OpenAI side is a total that already includes cached tokens, so subtracting `cached_tokens` out of it is not a correction, it's what stops the same tokens being billed twice in the ledger — once at full rate and once at the cache rate.

## Pitfalls

`muteHttpExceptions` **is what makes an HTTP error readable.** Without it `UrlFetchApp.fetch` throws on a 429 and the exception text is a generic failure message, not the provider's JSON body explaining which quota you hit. With it you read the code, and the provider's own error text goes into your log.

**The cheap model is not cheap on every task.** Routing pays because the small model is genuinely adequate for classification and tagging. It is not a free swap for extraction against a messy document or for anything with a tool loop, where a worse model costs more in retries than the flagship cost in tokens. Route by task and measure the accuracy on that task, not by price alone.

**Caches don't survive a provider switch, and they don't survive a model switch either.** Prompt caches are model-scoped. A router that sends the same system prompt to `claude-haiku-4-5` on one call and `claude-opus-5` on the next pays a cold write on each escalation, on the whole prefix. That's a real cost the routing table can hide from you if you only look at per-token rates.

**Gemini's thinking tokens are in their own counter.** They bill as output but they are not inside `candidatesTokenCount`. A ledger that reads only `candidatesTokenCount` under-reports a reasoning-heavy Gemini route, and under-reporting is worse than not measuring — it produces a number you trust.

**The 6-minute execution ceiling applies to the whole loop, not per call.** A router in a batch job over thousands of rows still has to chunk and resume across triggers, and a cheaper model doesn't buy you more wall clock. That constraint is unchanged by anything here.

I ran the pure logic in Node before shipping it. `pickModel`, `buildRequest`, `readText`, `readUsage` and `costUsd` contain no `UrlFetchApp`, no `PropertiesService` and no `SpreadsheetApp`, which is what made them testable at all — 22 test groups, including one that asserts every task `pickModel` can return has a key in `MODELS`, one that a cache-read-only call bills exactly a tenth of the same tokens read cold, and the two that pin the $6.24 and $330 figures at the top of this article to the registry rather than to my arithmetic.

## The takeaway

Model routing is not a prompt change, it's a boundary: one function that owns every provider difference, one registry that owns every price, and a ledger row per call so the saving is measured instead of assumed. Build those three and switching a task to a cheaper model is a one-line edit to `pickModel`.

The full category map — which workflow to automate first, how the same pattern runs a WhatsApp CRM, invoice extraction and nightly analysis off one Apps Script project — is on the [MageSheet blog](https://magesheet.com/blog/google-workspace-ai-automation).

Built by the MageSheet team.