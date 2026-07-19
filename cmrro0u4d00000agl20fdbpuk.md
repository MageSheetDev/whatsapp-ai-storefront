---
title: "Build a Commission Attribution Engine in Google Sheets: Matching, Splits &amp; Clawbacks"
seoTitle: "Build a Commission Attribution Engine in Google Sheets"
seoDescription: "Most commission errors happen at matching, not math. Build the attribution engine in Google Sheets: email normalization, splits, clawback windows."
datePublished: 2026-07-19T10:41:09.790Z
cuid: cmrro0u4d00000agl20fdbpuk
slug: build-a-commission-attribution-engine-in-google-sheets-matching-splits-amp-clawbacks
canonical: https://magesheet.com/blog/b2b-sales-commission-tracking
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/7b940a00-a5b2-44af-8087-78006e75c4a6.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/4665acc9-a577-4bdf-98ca-c44868f277b0.jpg
tags: saas, automation, google-apps-script, googlesheets, magesheet

---

A rep pings you: "My payout is $340 short." You open the sheet, and three hours later you find it — the customer paid from `ali.veli+billing@gmail.com` instead of the address on the account, so the sale never got attributed to anyone.

The math was never wrong. In the commission cleanups I've done for clients, nearly every dispute traced back to this: the payment didn't match the right rep. I'd put it around 95% — matching, not arithmetic. Yet almost everyone builds the calculator first.

I already wrote up the calculator — tiered rates, sales that span bands, refund reversal — in [the tiered commission engine post](https://magesheet.com/blog/whatsapp-automation-small-businesses-2026). This post is the layer *underneath* it: the attribution engine that decides whose sale it was in the first place.

## Why matching is where it breaks

A payment arrives from Stripe, Gumroad, or a WhatsApp deal. To pay commission on it you have to answer one question: **which rep owns this?** And the payment record fights you:

*   The buyer checked out with a personal address, not the one on the account.
    
*   They used a `+tag` alias, or Gmail dot variations (`a.b@` vs `ab@`).
    
*   It's a renewal, and nobody set an owner on the original.
    
*   Two reps worked it — a BDR who sourced it and an AE who closed it.
    

Get this wrong and the calculator faithfully computes the wrong person's commission. That's why the matcher comes first.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/1ec3e179-3fea-416d-bbc5-66e751fdea7c.jpg align="center")

## 1\. Normalize the email before you compare anything

Half the misses disappear here. Strip `+tags` everywhere, and strip dots **only for Gmail** — because Gmail ignores dots in the local part, while other providers treat them as significant. Getting that distinction backwards silently merges different people.

```javascript
function normalizeEmail(email) {
  const parts = String(email || '').toLowerCase().trim().split('@');
  if (parts.length !== 2 || !parts[1]) return '';
  let local = parts[0].split('+')[0];                              // drop +tags
  if (parts[1] === 'gmail.com') local = local.replace(/\./g, '');   // gmail ignores dots
  return local + '@' + parts[1];
}

function domainOf(email) {
  return String(email || '').toLowerCase().split('@')[1] || '';
}
```

`Ali.Veli+shop@Gmail.com` and `aliveli@gmail.com` are the same person. `a.b@company.com` and `ab@company.com` are not.

## 2\. Match in tiers, and never trust a free domain

Exact email is a certain match. Company domain is a good guess. A free provider domain is **not a signal at all** — and this is the trap that quietly ruins commission data: if you allow domain matching on `gmail.com`, every consumer purchase in your system gets assigned to whichever rep happens to own one Gmail contact.

```javascript
const FREE_DOMAINS = new Set([
  'gmail.com', 'outlook.com', 'hotmail.com', 'yahoo.com', 'icloud.com',
]);

// accounts: [{ email, domain, repId }]
function matchRep(payment, accounts) {
  const email = normalizeEmail(payment.email);
  if (!email) return { repId: null, confidence: 0, method: 'no-email' };

  const byEmail = accounts.find(a => normalizeEmail(a.email) === email);
  if (byEmail) return { repId: byEmail.repId, confidence: 1, method: 'email' };

  const dom = domainOf(email);
  // NEVER match on a free provider — it would assign every consumer sale to one rep.
  if (dom && !FREE_DOMAINS.has(dom)) {
    const byDomain = accounts.find(a => a.domain === dom);
    if (byDomain) return { repId: byDomain.repId, confidence: 0.8, method: 'domain' };
  }
  return { repId: null, confidence: 0, method: 'unmatched' };
}
```

Returning a **confidence** rather than a bare `repId` is what makes this usable. Auto-post anything at `1`, route `0.8` to a dispatcher for a one-click confirm, and park `0` in an "unmatched" tab that someone clears weekly. An unmatched row is a known unknown; a wrongly-matched row is a dispute three months later.

## 3\. Clawback windows and split credit

A refund shouldn't always claw back commission. Every commission policy needs a **window** — typically 30–90 days for refunds and 6–12 months for churn on annual contracts. Outside the window, the rep keeps it. Put that in the policy on day one, and encode it:

```javascript
function withinClawbackWindow(saleDate, refundDate, windowDays) {
  const days = (new Date(refundDate) - new Date(saleDate)) / 86400000;
  return days >= 0 && days <= windowDays;   // negative = refund dated before the sale
}

function splitCredit(amount, splits) {
  const total = Object.keys(splits).reduce((s, k) => s + splits[k], 0);
  if (Math.abs(total - 1) > 1e-9) {
    throw new Error('splits must sum to 1, got ' + total);
  }
  const out = {};
  Object.keys(splits).forEach(k => { out[k] = amount * splits[k]; });
  return out;
}
```

`splitCredit` throws instead of silently normalizing. That's deliberate: a BDR/AE split that adds up to 0.9 means somebody fat-fingered the rules table, and you want to hear about it now rather than discover 10% of a quarter's credit evaporated.

I unit-tested all of this before shipping — the cases that matter are the Gmail dot/`+tag` variants, a free-provider address refusing to domain-match, a refund dated before its sale, and splits that don't sum to 1.

## Pitfalls

*   **Build the matcher before the calculator.** A perfect tier engine on wrong attribution just pays the wrong person precisely.
    
*   **Never domain-match a free provider.** It's the single most damaging default in a commission system.
    
*   **Store confidence, not just a match.** Unmatched rows are cheap; wrong matches are disputes.
    
*   **Write the clawback window into the policy first.** Retro-applying one to sales already paid out destroys trust faster than a wrong number.
    
*   **Splits drift.** Re-check the rules table each quarter; a deal that changed hands mid-cycle is the usual culprit.
    
*   **Give each rep a view of only their own rows.** Full visibility into the ledger turns into salary politics.
    
*   **Know the ceiling.** This holds comfortably to roughly 15 reps on Sheets; past that the ledger wants a real database.
    

## Wrap-up

Commission disputes almost never start in the math. They start when a payment can't find its rep — an alias, a personal address, a renewal with no owner. Normalize the email, match in tiers with an explicit confidence, refuse to guess on free domains, and keep unmatched rows visible. The calculator can be simple once the attribution is honest.

The production version — Stripe/Gumroad ingestion, the per-rep tokenized dashboard, and the finance export — is written up on the [MageSheet blog](https://magesheet.com/blog/b2b-sales-commission-tracking).

Built by the MageSheet team.