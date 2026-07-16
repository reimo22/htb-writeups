# Sequel

**OS:** Linux
**Difficulty:** Very Easy
**Goal:** Exploit a misconfigured MySQL service (no auth) to retrieve the flag.

## Reconnaissance

```bash
nmap <target-ip> -F                 # fast scan, top 100 ports, missed some stuff
nmap <target-ip> -sC -sV -p 3306       
```

> running `nmap -sV -p 3306` alone timed out; instead I had to run `-sC` and `-sV` together.

The fast scan (`-F`) was a quick first pass but only checks the top 100 ports, so it's not
trustworthy on its own for deciding a box has nothing else open. Port 3306 (MySQL) stood out
as the one worth a targeted follow-up scan.

## Exploitation

> Note: trying `mysql -h <target-ip> -u root` with no password was a shot in the dark, prompted by the box's guided questions.

MySQL exposed on a box with no other obvious foothold is a common enough misconfiguration
that testing for no-auth root access first made more sense than deeper enumeration.

```bash
mysql -h <target-ip> --skip-ssl -u root

# Enumerate
SHOW DATABASES;
USE htb;
SHOW TABLES;           # returns: config, users
```

`users` and `config` were both worth a look since there was no indication which one held
the flag; skimming both, `config` turned out to hold it.

```bash
SELECT * FROM config;
```

## Lessons Learned

- Don't assume a scan without `-sV`/`-sC` gives a complete picture, some services (this one
  included) time out or behave inconsistently under a bare version scan and need both flags
  together to return reliably.
- Don't assume the most "obvious" table (`users`) holds the objective. Check all tables in a
  database before concluding a dead end.
- No-auth root MySQL is worth testing early on hosts where MySQL is the only unusual open
  port, it costs one command and rules out the easiest path immediately.