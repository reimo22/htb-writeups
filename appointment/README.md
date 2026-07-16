# Appointment

**OS:** Linux
**Difficulty:** Very Easy
**Goal:** Bypass login authentication via SQL injection.

## Reconnaissance

> [!NOTE]
> Skipped recon (`nmap`, `gobuster`) this time and went straight to SQLi, worked this time,
> but bad habit.

For reference, what a normal pass here would look like:

```bash
nmap -sC -sV <target-ip>
gobuster dir -u http://<target-ip> -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

This maps open ports/services and any hidden directories before touching the login form
directly.

## Exploitation

The login form was the only thing on offer, so testing it for SQLi first was a reasonable
shortcut given the box's difficulty tier, even without recon confirming there was nothing
else to check.

Payload, email field:

```sql
' or 0=0 #
```

Password can be anything.

**How it works:** The app concatenates input directly into the query:

```sql
-- Original
SELECT * FROM users WHERE username = 'USER_INPUT';

-- After injection
SELECT * FROM users WHERE username = '' OR 0=0 #';
```

- `'` closes the string
- `OR 0=0` always true, returns all rows / bypasses auth
- `#` comments out the rest (`--` also works depending on the SQL engine)

Once the bypass worked, I also confirmed a narrower payload holds up, which points at the
same underlying flaw (no input sanitization, no parameterized queries) rather than a fluke.

> Simpler payload that also works: `Admin'#` in the username field with any password.

## Lessons Learned

- Skipping recon happened to work here because the attack surface was small enough (single
  login form) that guessing paid off, but that's the exception, not something to rely on.
  Enumerate first, even on very easy boxes.
- Classic auth-bypass SQLi is worth testing on any login form with no other visible
  restriction (rate limiting, CAPTCHA), it's a near-zero-cost check.