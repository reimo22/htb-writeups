# Crocodile

**OS:** Linux
**Difficulty:** Very Easy
**Goal:** Retrieve credentials via anonymous FTP, then find a hidden admin panel via directory brute-forcing.

## Reconnaissance

```bash
nmap -sC -sV -F <target-ip>
```

FTP showing up in the scan results made anonymous login the natural first thing to rule
out, it's a common enough misconfiguration on beginner boxes that it's worth checking before
any web enumeration.

## Exploitation

> This anonymous FTP attempt was prompted by the box's guided questions.

```bash
# Anonymous FTP login (no credentials)
ftp ftp://anonymous@<target-ip>

# Two files exposed
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
```

`allowed.userlist` contains usernames; `allowed.userlist.passwd` contains matching
passwords. `admin` is the relevant account. I obtained credentials from these userlist files
first, then ran `gobuster` specifically to find a login panel to use them on, not as blind
recon before having credentials, since I already had a specific account to target rather than
enumerating the whole site cold.

```bash
# Directory brute-force
gobuster dir -u <target-ip> -w /path/to/common.txt
```

- Wordlist used: [SecLists/common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt)
- Found `/dashboard`, logged in with `admin` credentials, flag.

## Lessons Learned

- Anonymous FTP is worth checking immediately whenever FTP shows up in a port scan, it's low
  cost and a common misconfiguration on easy-tier boxes.
- Having credentials before brute-forcing directories narrows the search: you're looking for
  *somewhere to use them*, not mapping the whole site blind. Worth sequencing enumeration
  this way when creds surface early.