---
title: "Your Chatbot Verified the Price, Then Pasted It Into a Sentence It Wrote"
seoTitle: "Grounded AI Chat: Render Verified Prices, Don't Patch Them"
seoDescription: "Verifying a model's price is not enough if you paste it into the sentence it wrote. Let the model return slots and have your code render every number."
datePublished: 2026-09-07T14:52:48.072Z
cuid: cmtrd11cn00000agm4cxh91x1
slug: grounded-chatbot-render-verified-numbers
canonical: https://magesheet.com/blog/ground-ai-chatbot-on-catalog-data
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/da4ce356-23dd-44c1-a2ff-7d1778904e4d.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/c4e5a29f-91d8-4ea1-b2b3-71a770614d93.jpg
tags: automation, google-sheets, google-apps-script, no-code-platform, magesheet

---

A grounded chatbot retrieves the right catalog rows, asks the model which price it used, looks that price up, and compares. Mine did all of that and still quoted a stale price to a customer. The retrieval was fine. The verification was fine. The line that broke it was the one that came after:

```javascript
answer = answer.replace(String(out.priceUsed), String(livePrice));
```

The model had reported `priceUsed: 1299` and written `$1,299.00` in its sentence. Those are the same number and not the same string, so the replace matched nothing, returned the original text, and the customer read the old price under a system that had just verified it.

That is the gap this post is about. Verification tells you whether a number is right. It does nothing about how that number reaches the customer, and if the answer is a sentence the model wrote, every verified fact has to survive a string operation to get there.

## Why patching a sentence cannot be made safe

The failure above is not a formatting bug you fix by normalising before the replace. There are at least four ways this line goes wrong and they do not share a fix:

*   **The prose and the field disagree in form.** `1299` against `$1,299.00`, `849.5` against `$849.50`, `1299` against `1.299,00` for a European storefront.
    
*   **The number appears somewhere else first.** `String.prototype.replace` with a string argument replaces the first occurrence only. In `The 500ml flask is 500 in stock`, patching `500` rewrites the product name and leaves the stock figure alone.
    
*   **The answer covers two products.** You verified one price. The other sentence is still whatever the model believed.
    
*   **The number never appears as digits.** "just under thirteen hundred" survives every replace you can write.
    

Each of these is survivable on its own. Together they mean the safety of the answer depends on the model having phrased things conveniently, which is exactly the assumption grounding exists to remove.

So stop patching. The model does not get to write numbers.

## The model writes the sentence, the code writes the numbers

Ask the model for a template instead of a finished answer. Named slots where numbers belong, and nothing else:

```plaintext
{ "grounded": true,
  "productId": "RC-500",
  "template": "The navy raincoat is {price} and we have {stock} in stock." }
```

Your code fills the slots from values it looked up itself. The model supplies word order, tone and the decision about which product is being discussed. It never supplies a digit that reaches a customer.

```javascript
var SLOT_PATTERN = /\{([a-zA-Z][a-zA-Z0-9_]*)\}/g;

function slotsIn(template) {
  var seen = {}, out = [], m;
  SLOT_PATTERN.lastIndex = 0;
  while ((m = SLOT_PATTERN.exec(String(template))) !== null) {
    if (!seen[m[1]]) { seen[m[1]] = true; out.push(m[1]); }
  }
  return out;
}

// A slot with no verified value is a refusal, not an empty string. The model
// built a sentence around a number nobody can confirm, so the sentence goes.
function renderAnswer(template, facts, format) {
  var missing = slotsIn(template).filter(function (name) {
    return facts[name] === undefined || facts[name] === null;
  });
  if (missing.length) {
    return { type: 'refusal',
             reason: 'no verified value for: ' + missing.join(', ') };
  }
  SLOT_PATTERN.lastIndex = 0;
  var text = String(template).replace(SLOT_PATTERN, function (_, name) {
    return format(name, facts[name]);
  });
  return { type: 'answer', text: text };
}
```

Two details in there earn their place. `facts[name] === undefined || facts[name] === null` rather than a falsy check, because a stock level of `0` is a verified fact and "we have 0 in stock" is a true and useful sentence. And formatting happens in your `format` function, so currency and locale are decided by code that knows the store, not by a model guessing from context.

## Then check that the model kept its side of the bargain

A template is an instruction, and instructions get ignored. Nothing stops the model returning `"It is $1,350 and we have {stock} left."` — one slot honoured, one price invented. So the finished text gets swept: every number in it has to be one the code put there.

This is where the naive version of the check has a hole worth knowing about. Product names carry digits — `Model 3`, `500ml bottle`, SKU `RC-500` — so a sweep that refuses every number not in the verified set refuses real answers and gets switched off within a week. The obvious fix is to allow any number that appears in the retrieved rows. That fix is worse than it looks: SKUs are full of small integers, and a small integer is exactly what an invented quantity looks like. With `BT-3` in the catalog, "ships in 3 days" walks straight through.

So a row number is allowed only where it keeps the same company it keeps in the row:

```javascript
function strayNumbers(text, verifiedValues, retrievedRows) {
  var rowText = normalise(JSON.stringify(retrievedRows || []));
  var allowedValues = {};
  (verifiedValues || []).forEach(function (n) {
    allowedValues[Number(n)] = true;
  });

  var stray = [], re = /\d[\d.,]*/g, m;
  while ((m = re.exec(String(text))) !== null) {
    var raw = m[0].replace(/[.,]$/, '');
    var value = numbersIn(raw)[0];
    if (value === undefined) continue;
    if (allowedValues[value]) continue;        // the code put this one here

    var before = String(text).slice(0, m.index);
    var prev = (before.match(/[A-Za-z]+[\s-]?$/) || [''])[0];
    var after = String(text).slice(m.index + raw.length);
    var suffix = (after.match(/^[A-Za-z]+/) || [''])[0];

    // Glued to letters, like 500ml or RC-500, or preceded by the same word it
    // follows in the row, like Model 3. A number standing on its own has no
    // such company and has to come from a verified fact.
    var glued = suffix &amp;&amp; rowText.indexOf(normalise(raw + suffix)) !== -1;
    var kept = prev &amp;&amp; rowText.indexOf(normalise(prev + raw + suffix)) !== -1;
    if (glued || kept) continue;

    stray.push(value);
  }
  return stray;
}
```

`numbersIn` reduces a token to its value before comparing, so `$1,299.00`, `1299` and `1.299,00` are one number rather than three. That matters more than it sounds: comparing the characters instead of the value is the same mistake as the string replace, one layer down.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/86e597cb-1d21-4a22-ae45-485d16da3330.svg align="center")

The order of operations is the whole design. Refuse before rendering, render before sweeping, sweep before anything reaches a customer:

```javascript
function groundedAnswer(modelReply, facts, retrievedRows, deps) {
  if (!modelReply.grounded) {
    return { type: 'refusal',
             reason: 'model could not ground the answer' };
  }

  var rendered = renderAnswer(modelReply.template, facts, deps.format);
  if (rendered.type !== 'answer') return rendered;

  var slots = slotsIn(modelReply.template);
  var verified = slots.map(function (n) { return facts[n]; });
  var stray = strayNumbers(rendered.text, verified, retrievedRows);
  if (stray.length) {
    return { type: 'refusal',
             reason: 'unverified number: ' + stray.join(', ') };
  }
  return rendered;
}
```

Note what happens on a stray number: refusal, not repair. Repair is how the original bug got in. A sentence built around a number you cannot account for is a sentence you do not send, and the handoff to a person is the feature, not the failure.

## Pitfalls

*   **Numbers written as words.** No digit sweep sees "a dozen left". A short list of number words closes the common cases, but leave `one` out of it — in English it is a pronoun far more often than a quantity, and "the cheaper one" would refuse every second answer. The real defence is the template: a model asked for slots rarely spells quantities out.
    
*   **Allowing every number in the retrieved rows.** Covered above, and worth repeating because it is the version most people write first. It feels like the safe default and it silently re-opens the hole for exactly the small integers that matter.
    
*   **Treating a zero as missing.** `if (!facts[name])` refuses "0 in stock", which is the single most important answer an out-of-stock page can give.
    
*   **Confusing provenance with freshness.** This sweep proves a number came from your code. It says nothing about whether your code read a fresh value. If you cache the price for an hour, you will render a verified hour-old number with total confidence. Live lookups are a separate job with their own [quota and retry ceiling](https://magesheet.com/blog/apps-script-urlfetchapp-quotas-retries).
    
*   **Letting the model return prose alongside the template.** If your schema has both an `answer` and a `template` field, someone will render `answer` on a Friday. Return one field, and make the template the only path to text.
    
*   **Skipping structured output.** Parsing a template out of free text puts you back where you started. Tool use and JSON schema modes make the reply a shape rather than a paragraph containing one; the [mechanics on Apps Script](https://magesheet.com/blog/claude-api-apps-script-tool-use-prompt-caching) are worth getting right before this is load-bearing.
    

## The one thing worth taking away

Verification decides whether a number is true. Rendering decides whether the true number is the one the customer reads, and those are different problems with different code. If a verified value has to travel through a sentence the model wrote to reach the screen, you have a formatting coincidence standing where a guarantee should be.

The full grounding stack this sits inside — retrieval, live lookups, refusal paths and the catalog hygiene that has to come first — is on the [MageSheet blog](https://magesheet.com/blog/ground-ai-chatbot-on-catalog-data).