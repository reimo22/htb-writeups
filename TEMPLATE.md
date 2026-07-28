# {{Box Name}}

**Difficulty:** Very Easy/Easy/Medium/Hard
**OS:** Linux/Windows
**Target IP:** {{target-ip}}

<!-- One paragraph: what kind of box this is and the chained path, e.g.
"{{Box}} is a very easy Linux box that chains X into Y into Z." -->

## Enumeration

<!-- Full port scan first, then a targeted -sC -sV scan on what's open.
Show the real command AND the real output — not a description of the scan. -->

```bash
nmap -p- --min-rate 5000 --max-retries 9 <target-ip>
```

```text
PORT   STATE SERVICE
```

```bash
sudo nmap -sC -sV <target-ip> -p <open-ports>
```

```text
PORT   STATE SERVICE VERSION
```

<!-- Break out further enumeration into named subsections per service/lead —
e.g. "### Port scan", "### Virtual host discovery", "### Checking known CVEs".
Don't force these three; use whatever the box actually needed. -->

<!-- From here, use top-level (##) sections named after what actually happened —
"## FTP: Anonymous Access", "## Exploiting the S3 bucket", "## Cracking backup.zip" —
not generic headers like "Exploitation" or "Privilege Escalation". Include the
actual payload/request/command and *why* it worked, and any dead ends worth
noting (e.g. "## The sudo Dead End (and why it happened)"). -->

## Root

<!-- How root/system was actually confirmed, e.g. reading the flag. -->

<!-- Optional, worth adding once the box has 3+ distinct stages:
## Summary

| Stage | Technique |
|---|---|
| | |
-->

## Lessons Learned

<!-- Bullet points, each a real insight — a tool default that bit you, a dead
end and why it happened, something you'd do differently next time. Not a
recap of the steps above. -->

-
