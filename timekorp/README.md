# TimeKORP

**Event:** HTB Tryout 2026 (originally HTB Cyber Apocalypse 2024)
**Category:** Web
**Difficulty:** Very Easy
**Target:** ephemeral instance (e.g. `http://154.57.164.76:31497`)
**Mentored via:** a custom fork of [0xdf](https://gitlab.com/0xdf/htb-ai-mentor/-/tree/main)'s Socratic HTB-mentoring skill

TimeKORP is a very easy web challenge built around a single-page "what's the time?" app. The page's displayed time/date is driven by a `format` GET parameter that gets reflected straight into an OS `date` command via `exec()` — unsanitized `strftime`-style input turns into OS command injection (CWE-78), leading to arbitrary command execution and the flag.

## Enumeration

Stack: jQuery, nginx, Popper. The displayed time is set once on page load and never updates dynamically — DevTools' network tab catches the actual request driving it: `GET /?format=%H:%M:%S`. A second nav link exists for `?format=%Y-%m-%d` (date view). Arbitrary values placed in `format` get reflected directly into the rendered page.

### Dead End: XSS Tangent

`<script>alert("XSS")</script>` in `format` executed — confirmed reflected, unsanitized input. Dead end as an actual vector though: single-player challenge, no admin bot or second browser context ever loads the URL, so there's no victim for injected JS to run against.

## Identifying the Backend: the `date` Binary

`%H:%M:%S` / `%Y-%m-%d` are `strftime`-style format tokens — the same syntax `date +FORMAT` takes on the command line. Working theory: the app isn't formatting the date itself, it's shelling out to the `date` binary with the raw parameter value. Testing `?format=/flag.txt` reflected the literal string `/flag.txt` back (`It's /flag.txt.`) rather than any file contents or error — consistent with `date` treating input with no `%` specifiers as literal text to print back verbatim, not a file path.

### Dead End: `date -f` Misuse

Tried `?format=' && date -f /flag.txt` (pattern pulled from GTFOBins) expecting it to dump the file. `date -f <file>` doesn't do that — it reads each line of the file *as a date string to parse and format*, it's not a file-read primitive. `cat` is the actual one-word command for that job.

## Breaking Out of the Quote

Underlying template: `date '+<FORMAT>' 2>&1`. A `'` in the input closes that opening quote; `&&` chains a new command — but there's still a trailing `' 2>&1` after the injection point that needs to be neutralized (`#` comments it out) or the resulting shell line is malformed and silently fails.

### Dead End: Unencoded `&&` and `#`

`?format=%27%20&&%20cat%20/flag.txt%20#` produced nothing when pasted straight into a browser address bar. Both characters get eaten a layer before the app ever sees them:

- `&&` unencoded in a URL splits the query string into separate parameters — the server receives a truncated `format` value, not the chained command.
- `#` unencoded is a URL fragment marker, stripped client-side before the request is even sent.

Fix: `%26%26` for `&&`, `%23` for `#`.

## Confirmed RCE

```text
?format=%27%20%26%26%20ls%20-la%20%23
→ drwxr-xr-x. 1 www www 23 Mar 28 2024 views

?format=%27%20%26%26%20cd%20views%20%26%26%20ls%20-la%20%23
→ -rw-r--r--. 1 www www 1920 Mar 28 2024 index.php

?format=%27%20%26%26%20cat%20/etc/passwd%20%23
→ messagebus:x:104:105::/nonexistent:/usr/sbin/nologin
```

`cat /root/flag.txt` and plain `cd && ls -la` both failed at first, read as "stuck inside `www`" — but each `?format=` request spawns a fresh `exec()`/shell process with no persistent working directory between requests. There's nothing to `cd` out of; an absolute path works regardless of per-request cwd.

## Flag

```text
?format=%27%20%26%26%20cat%20%2Fflag%20%23
→ HTB{t1m3_f0r_th3_ult1m4t3_pwn4g3_006114c460d744eac4fbc4fe6998429a}
```

**Flag:** `HTB{t1m3_f0r_th3_ult1m4t3_pwn4g3_006114c460d744eac4fbc4fe6998429a}`

The flag file has no extension (`/flag`, not `/flag.txt`) — the earlier `/flag.txt` / `/root/flag.txt` guesses were both wrong on that basis alone. Once RCE was confirmed via `/etc/passwd`, going straight to `/flag` (the standard HTB convention) would have skipped the `views/index.php` detour entirely.

## Lessons Learned

- Recognize `strftime`-style tokens (`%H:%M:%S`, `%Y-%m-%d`) as a signal the backend may be shelling out to the real `date` binary, rather than assuming app-level string formatting.
- SQL injection reasoning transfers by analogy — break out of the wrapping quote, insert a command, deal with trailing syntax — but the "neutralize the trailing fragment" step doesn't transfer automatically; it has to be worked out for the specific quoting context.
- Don't pull a GTFOBins pattern (`date -f`) without checking what the flag/command actually does first — `date -f <file>` parses the file as date input, it doesn't dump it.
- Each request to a stateless `exec()`-backed endpoint starts a fresh shell — a failed `cd` isn't a wall, it's a non-issue once you default to absolute paths.
- Encode `&&` and `#` before pasting a payload into a browser address bar — both get consumed (query-string split, fragment strip) before the request is even sent.
- `/flag` with no extension is the default HTB convention — try it before guessing `/flag.txt` or `/root/flag.txt`.
