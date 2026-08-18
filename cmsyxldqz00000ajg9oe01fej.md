---
title: "Apps Script Session Auth: Your Cache Expires Before Your Session Does"
seoTitle: "Apps Script Session Auth: The 6-Hour CacheService Limit"
seoDescription: "Sessions built on CacheService die at six hours and can vanish sooner. The tested Apps Script pattern: hashed tokens, a Sessions row, real revocation."
datePublished: 2026-08-18T17:23:10.448Z
cuid: cmsyxldqz00000ajg9oe01fej
slug: apps-script-session-auth-cacheservice
canonical: https://magesheet.com/blog/google-apps-script-complete-guide
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/f19b8e78-22a7-47ec-b2cb-63445cd6e132.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/49e89fba-92d3-4107-ab21-6fc2756d6d59.jpg
tags: authentication, javascript, google-sheets, googleappsscript, magesheet

---

A user signs in at 09:05, works through the morning, and at 14:40 a click comes back with "your session has expired". They sign in again and the app behaves perfectly. The next day it happens at 11:20 instead, to somebody else, on a different screen.

Nothing in the execution log is red. The session policy in the code says thirty days.

It was never thirty days. `CacheService.put()` documents a maximum expiration of 21600 seconds, and the same class is documented as a best-effort store whose entries can be dropped before they expire. A session that exists only in the cache has a ceiling of six hours and no floor at all.

## The cache is a fast path, not a session store

Apps Script gives you no login system, so you build one, and every guide reaches the same shape: issue a token, put a record in `CacheService`, check it on the next call. The shape is right. The storage assumption is what fails, and it fails on documented behaviour rather than on a bug.

| CacheService property | Documented behaviour | What it means for a session |
| --- | --- | --- |
| Maximum expiration | 21600 seconds, six hours | A larger number does not buy a longer login |
| Eviction | entries can be dropped before expiry | A live session can vanish mid-afternoon |
| Value size | 100 KB per key | Fine for a session record, not for the user's data |
| Key length | 250 characters | A hash fits, a serialised object does not |

Read those together and the conclusion is narrow: the cache can tell you a session is valid, but it can never tell you a session is gone. A missing entry means *expired, revoked, evicted, or the cache was flushed*, and those four need different answers.

So the record lives in two places with two different jobs. A `Sessions` sheet is the record of truth: durable, slow, one read per cache miss. The cache is the fast path in front of it, holding the same record for as long as the platform will hold anything. Cache miss becomes the normal case rather than the error case, and the code is written for it.

The first block is the plumbing: hashing, keys, and the two stores.

```javascript
/**
 * Session auth for an Apps Script web app.
 * The cache is the fast path. The Sheet row is the record.
 * Nothing here ever stores the raw token.
 */

var SESSION = {
  sheet: 'Sessions',
  users: 'Users',
  idleMinutes: 30,     // sliding window
  absoluteHours: 12,   // hard cap, never extended
  keyPrefix: 'ses:'
};

// CacheService.put documents 21600 seconds (6 hours) as the
// maximum expiration. A longer number is not a longer session.
var CACHE_MAX_SECONDS = 21600;

/**
 * Utilities.computeDigest returns SIGNED bytes (-128..127).
 * Without the & 0xFF, every byte over 127 renders as negative
 * hex and the key stops matching the one you stored.
 */
function toHex_(bytes) {
  var out = '';
  for (var i = 0; i < bytes.length; i++) {
    var b = bytes[i] & 0xFF;
    out += (b < 16 ? '0' : '') + b.toString(16);
  }
  return out;
}

function sha256Hex_(text) {
  return toHex_(Utilities.computeDigest(
    Utilities.DigestAlgorithm.SHA_256,
    String(text), Utilities.Charset.UTF_8));
}

/** The store only ever sees the hash, never the token. */
function sessionKey_(token) {
  return SESSION.keyPrefix + sha256Hex_(token);
}

/**
 * Script cache, not user cache: a web app deployed to
 * "Anyone" has no signed-in user to key a user cache on.
 */
function cache_() {
  return CacheService.getScriptCache();
}

function readCache_(key) {
  var raw = cache_().get(key);
  if (!raw) return null;
  var rec = JSON.parse(raw);
  rec.fromCache = true;
  return rec;
}

function writeCache_(rec, ttlSeconds) {
  if (ttlSeconds <= 0) return;
  cache_().put(rec.key, JSON.stringify({
    key: rec.key, email: rec.email, role: rec.role,
    issuedAt: rec.issuedAt, lastSeenAt: rec.lastSeenAt,
    rowLastSeenAt: rec.rowLastSeenAt
  }), ttlSeconds);
}

function sheet_(name) {
  var ss = SpreadsheetApp.getActive();
  return ss ? ss.getSheetByName(name) : null;
}

/**
 * Sessions columns:
 * A key | B email | C role | D issuedAt | E lastSeenAt
 * Epoch milliseconds, written to a plain-text column so a
 * long number is never reformatted into a date.
 */
function findSessionRowIndex_(key) {
  var sh = sheet_(SESSION.sheet);
  if (!sh || sh.getLastRow() < 2) return 0;
  var hit = sh.getRange(2, 1, sh.getLastRow() - 1, 1)
    .createTextFinder(key).matchEntireCell(true).findNext();
  return hit ? hit.getRow() : 0;
}

function readSessionRow_(key) {
  var i = findSessionRowIndex_(key);
  if (!i) return null;
  var row = sheet_(SESSION.sheet)
    .getRange(i, 1, 1, 5).getValues()[0];
  return {
    key: String(row[0]),
    email: String(row[1]),
    role: String(row[2]),
    issuedAt: Number(row[3]),
    lastSeenAt: Number(row[4]),
    rowLastSeenAt: Number(row[4]),
    rowIndex: i,
    fromCache: false
  };
}
```

Two decisions in there are worth naming. The store holds `sha256Hex_(token)` and never the token, so a person with read access to the Sheet has a list of hashes rather than a list of working logins. And it is the script cache, not the user cache: a web app deployed to "Anyone" has no signed-in Google identity to key a user cache on, so `getUserCache()` in that deployment is not the isolation it looks like.

## The expiry rule is one function with no service calls in it

Three questions decide whether a session is alive, and the order they are asked in is part of the answer.

**Was it revoked?** A "log out everywhere" button stamps a revoke instant on the user, and every session issued before that instant is dead. The comparison is against `issuedAt`, not `lastSeenAt`. Compare against `lastSeenAt` and the sessions that survive your revocation are exactly the active ones: the attacker clicking every few seconds keeps a fresh `lastSeenAt`, sails past the check, and the button you pressed logged out only the idle tabs.

**Is it past the absolute cap?** Twelve hours from issue, regardless of activity. This is the one an attacker with a stolen token cannot extend by using it.

**Is it past the idle window?** Thirty minutes since the last request, refreshed on every request.

Written as a pure function, all three are testable without touching a Google service, which is most of what makes this code auditable at all.

![](https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/6010d322-8723-4255-bc7f-00b34fae7e4b.jpg align="center")

```javascript
/**
 * The whole expiry decision, with no service calls in it.
 * revokedAt is compared against issuedAt, not lastSeenAt:
 * a session touched after the revoke instant must still die.
 */
function decideSession_(rec, now, revokedAt) {
  if (!rec) return { state: 'unknown' };
  var idleMs = SESSION.idleMinutes * 60 * 1000;
  var absMs = SESSION.absoluteHours * 60 * 60 * 1000;
  if (!(rec.issuedAt > 0) || !(rec.lastSeenAt > 0)) {
    return { state: 'unknown' };
  }
  if (revokedAt > 0 && rec.issuedAt <= revokedAt) {
    return { state: 'revoked' };
  }
  if (now - rec.issuedAt >= absMs) {
    return { state: 'expired-absolute' };
  }
  if (now - rec.lastSeenAt >= idleMs) {
    return { state: 'expired-idle' };
  }
  return { state: 'ok', email: rec.email, role: rec.role };
}

/**
 * How long the session may still live, in seconds, clamped to
 * what the cache will actually hold. A 12-hour session gets a
 * 6-hour entry and rehydrates from the row afterwards.
 */
function cacheTtlSeconds_(rec, now) {
  var idleMs = SESSION.idleMinutes * 60 * 1000;
  var absMs = SESSION.absoluteHours * 60 * 60 * 1000;
  var leftAbsolute = rec.issuedAt + absMs - now;
  var ms = Math.min(idleMs, leftAbsolute);
  if (ms <= 0) return 0;
  return Math.min(Math.ceil(ms / 1000), CACHE_MAX_SECONDS);
}

/**
 * Writing lastSeenAt on every request turns one click into
 * one Sheet write. Refresh the row only once it has drifted
 * more than half the idle window behind.
 */
function needsRowTouch_(rowLastSeenAt, now) {
  var halfIdleMs = SESSION.idleMinutes * 30 * 1000;
  return (now - Number(rowLastSeenAt || 0)) >= halfIdleMs;
}

/**
 * Cache miss is the normal case, not the error case: entries
 * are evicted before their TTL and the ceiling is six hours.
 */
function validateSession_(token, now) {
  if (!token) return { state: 'unknown' };
  var key = sessionKey_(token);
  var rec = readCache_(key) || readSessionRow_(key);
  if (!rec) return { state: 'unknown' };
  var verdict = decideSession_(rec, now, revokedAtFor_(rec.email));
  if (verdict.state !== 'ok') {
    dropSession_(rec);
    return verdict;
  }
  rec.lastSeenAt = now;
  if (needsRowTouch_(rec.rowLastSeenAt, now)) {
    if (touchSessionRow_(rec, now)) rec.rowLastSeenAt = now;
  }
  writeCache_(rec, cacheTtlSeconds_(rec, now));
  return verdict;
}

/**
 * Users columns: A email | B role | C revokedAt
 * Cached briefly, so "log out everywhere" takes effect within
 * that window rather than on the next request.
 */
var REVOKE_TTL_SECONDS = 60;

function revokedAtFor_(email) {
  var addr = String(email || '').trim().toLowerCase();
  if (!addr) return 0;
  var key = 'rev:' + addr;
  var hit = cache_().get(key);
  if (hit !== null && hit !== undefined) return Number(hit);
  var sh = sheet_(SESSION.users);
  var value = 0;
  if (sh && sh.getLastRow() > 1) {
    var found = sh.getRange(2, 1, sh.getLastRow() - 1, 1)
      .createTextFinder(addr).matchEntireCell(true).findNext();
    if (found) {
      value = Number(sh.getRange(found.getRow(), 3).getValue()) || 0;
    }
  }
  cache_().put(key, String(value), REVOKE_TTL_SECONDS);
  return value;
}
```

`cacheTtlSeconds_` is where the six-hour ceiling stops being a surprise. The session policy stays whatever you wrote it to be, twelve hours here, and the cache entry asks for the smallest of three numbers: the idle window, whatever is left of the absolute cap, and 21600. A twelve-hour session gets a six-hour entry, then a row read, then another entry. The policy and the storage stop being the same number, which is the whole fix.

`needsRowTouch_` is the counterweight. Refreshing `lastSeenAt` in the Sheet on every request turns one click into one write, and a web app has 30 seconds to answer. Refreshing it never is worse: after an eviction, the row is the only thing left, and a row that is 25 minutes stale reads as an idle timeout for a user who was active four seconds ago. Half the idle window is the compromise, and it is the reason a continuously active session survives eviction in the tests below.

## The gate in front of every server function

`google.script.run` has no request object and no headers. Every value the server sees is an argument, so the token is an argument, and the only thing that keeps it from being optional is that nothing runs before the gate.

```javascript
var ROLE_RANK = { viewer: 1, staff: 2, admin: 3 };

/** An unrecognised role is not a role. Blank is not admin. */
function normalizeRole_(role) {
  var r = String(role || '').trim().toLowerCase();
  return ROLE_RANK[r] ? r : '';
}

function canAccess_(role, minimumRole) {
  var have = ROLE_RANK[normalizeRole_(role)] || 0;
  var need = ROLE_RANK[normalizeRole_(minimumRole)] || 0;
  if (!have || !need) return false;
  return have >= need;
}

function issueSession_(email, role, now) {
  var token = Utilities.getUuid() + Utilities.getUuid();
  var rec = {
    key: sessionKey_(token),
    email: String(email).trim().toLowerCase(),
    role: normalizeRole_(role),
    issuedAt: now,
    lastSeenAt: now,
    rowLastSeenAt: now
  };
  if (!rec.role) throw new Error('Unknown role for ' + rec.email);
  sheet_(SESSION.sheet).appendRow([rec.key, rec.email, rec.role,
    rec.issuedAt, rec.lastSeenAt]);
  writeCache_(rec, cacheTtlSeconds_(rec, now));
  return { token: token, role: rec.role };
}

/**
 * A record served from cache carries no row index, so the row
 * is located by key on the rare pass that needs it.
 */
function touchSessionRow_(rec, now) {
  var i = rec.rowIndex || findSessionRowIndex_(rec.key);
  if (!i) return false;
  rec.rowIndex = i;
  sheet_(SESSION.sheet).getRange(i, 5).setValue(now);
  return true;
}

function dropSession_(rec) {
  cache_().remove(rec.key);
  var i = rec.rowIndex || findSessionRowIndex_(rec.key);
  if (i) sheet_(SESSION.sheet).deleteRow(i);
}

/**
 * Every server function the page can call goes through this.
 * google.script.run sends no headers, so the token travels as
 * an argument like any other value.
 */
function guarded_(token, minimumRole, fn) {
  var now = Date.now();
  var v = validateSession_(token, now);
  if (v.state !== 'ok') return { ok: false, reason: v.state };
  if (!canAccess_(v.role, minimumRole)) {
    return { ok: false, reason: 'forbidden' };
  }
  return { ok: true, data: fn(v) };
}

function listOrders(token) {
  return guarded_(token, 'staff', function (session) {
    return ordersFor_(session.email);
  });
}

/**
 * Log out everywhere. The revoke instant is stamped on the
 * user, so every session issued before it dies, this one
 * included.
 */
function logOutEverywhere(token) {
  return guarded_(token, 'viewer', function (session) {
    var sh = sheet_(SESSION.users);
    var found = sh.getRange(2, 1, sh.getLastRow() - 1, 1)
      .createTextFinder(session.email)
      .matchEntireCell(true).findNext();
    if (!found) return { revoked: false };
    sh.getRange(found.getRow(), 3).setValue(Date.now());
    cache_().remove('rev:' + session.email);
    return { revoked: true };
  });
}
```

`canAccess_` denies on both sides. An unrecognised role gives zero rank, and so does an unrecognised requirement, so a typo in the gate closes the door instead of opening it. That matters more than it sounds: the role comes out of a spreadsheet cell that a human edits, and `Administrator`, `ADMIN` , and an empty cell are all things that cell will eventually contain.

The refusal is typed rather than boolean. `unknown` means log in again, `expired-idle` means log in again but say why, `revoked` means the account was signed out elsewhere, and `forbidden` means the login is fine and the role is not. A single `false` sends all four to the login screen, including the one where the user is already signed in and merely lacks permission.

## What breaks in practice

**Signed bytes from** `computeDigest`**.** Apps Script returns the digest as bytes in the range -128 to 127. Convert them with `b.toString(16)` and every byte over 127 comes out as something like `-6f`, so the key is stable but wrong: wrong length, wrong alphabet, and different from what any other tool computes for the same token. `b & 0xFF` is the fix, and it is worth the assertion that pins the output to a known SHA-256 hex string.

**A 30-day number in a 6-hour argument.** Passing 2592000 to `put()` does not create a 30-day entry, because the documented maximum is 21600 seconds. Clamping explicitly, as `cacheTtlSeconds_` does, means the behaviour is decided by your code rather than by what the platform chooses to do with an out-of-range number.

`getUserCache()` **in a public deployment.** A web app deployed to run as you and be accessible by "Anyone" has no signed-in user to scope a user cache to. The script cache with a hashed key is the honest version of what the user cache looked like it was doing.

**Sessions in** `localStorage`**.** Any script that runs in the page can read it, so one XSS is every session on that machine. Keeping the token in a JavaScript variable for the lifetime of the tab costs the user a re-login when they refresh, and removes the exfiltration path entirely.

**Tokens from** `Math.random()`**.** It is not a cryptographic generator and its output is predictable given enough samples. `Utilities.getUuid()` is the platform's identifier generator; two of them concatenated, then hashed for storage, is a token that costs nothing to produce and nothing to store safely.

**The** `Sessions` **sheet grows forever.** Every login appends a row and only expired-on-access sessions get deleted, so a sheet that is never pruned turns `createTextFinder` into a full-column scan against a 30-second response budget. A daily trigger that deletes rows older than the absolute cap keeps it small, and that job is a good candidate for the [chunked, self-rescheduling pattern](https://magesheet.com/blog/apps-script-6-minute-limit) once the table is large.

**Revocation lags by the revoke cache TTL.** Reading the `Users` sheet on every request is a second Sheet read per click, so the revoke instant is cached for 60 seconds. That is a deliberate trade: "log out everywhere" takes effect within a minute rather than instantly. Set `REVOKE_TTL_SECONDS` to 0 if a minute is not acceptable, and pay for it in reads.

**Epoch milliseconds in a date-formatted column.** A cell holding 1750000000000 will render as a date if the column is formatted for dates, and `getValues()` then hands back a `Date` rather than a number. `Number()` survives both, but formatting the timestamp columns as plain text keeps the sheet readable by the humans who will inevitably open it.

The pure logic here was checked with 113 assertions in Node against mocked `Utilities`, `CacheService` and `SpreadsheetApp` services, including a `computeDigest` mock that returns signed bytes so the hex conversion is tested rather than assumed. The window behaviour was driven with a fake clock: a session used every 20 minutes survives past three hours and dies at the twelve-hour cap on the first request after it, an evicted-but-active session rehydrates from its row, and an evicted session with a stale row expires. Apps Script services cannot run outside the platform, so the call sites themselves were reviewed by hand against their documented return types.

## The takeaway

Sessions in Apps Script fail on storage, not on logic. The token, the sliding window and the role check are the easy half; the half that produces the 14:40 support ticket is treating a six-hour best-effort cache as though it were the place the session lives. Give the session a row, put the cache in front of it, and write the code so that a cache miss is Tuesday afternoon rather than an incident.

If your web app is public and has no login at all, the guards it needs are a different set, and [the booking-page build](https://magesheet.com/blog/apps-script-calendar-booking-calendly-alternative) covers those. The rest of the production picture, from the UI layer down to when to leave the platform, is in [the complete guide to building on Apps Script](https://magesheet.com/blog/google-apps-script-complete-guide).