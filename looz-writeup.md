# Looz — CTF Writeup

## Summary

Looz is a CTF machine whose web application exposes a WordPress instance that initially looks like the way in, but turns out to be a decoy. The real foothold comes from brute-forcing SSH credentials harvested from the website itself, followed by privilege escalation through a vulnerable SUID `pkexec` binary (PwnKit, CVE-2021-4034).

## Enumeration

- Initial recon pointed to a WordPress site running on the target.
- `wp-config.php` was accessible, revealing that the application loads its database credentials from environment variables via a custom `getenv_docker()` helper rather than hardcoding them — a pattern common to the official Dockerized WordPress image.

## The Docker/MySQL Trap

Dumping the process environment (`env` / `printenv`) leaked the real values behind those `getenv_docker()` calls:

```
WORDPRESS_DB_USER=dbadmin
MYSQL_ENV_MYSQL_PASSWORD=Ba2k3t
MYSQL_ENV_MYSQL_ROOT_PASSWORD=root-password
WORDPRESS_DB_NAME=wpdb
MYSQL_PORT_3306_TCP_ADDR=172.17.0.2
```

No `mysql` client was available on the box, so the database was queried directly with PHP's `mysqli`:

```php
php -r '$m=new mysqli("172.17.0.2","dbadmin","Ba2k3t","wpdb");
$r=$m->query("SELECT user_login,user_pass FROM wp_users");
while($row=$r->fetch_assoc()){print_r($row);}'
```

This confirmed database access (and even root MySQL credentials), but it turned out to be a dead end for gaining a shell on the host — a deliberate rabbit hole rather than the intended path.

## Actual Foothold: SSH Brute-Force

The real attack path came from the website content itself, which disclosed a valid username (`gandalf`). Using that username against SSH with **Hydra** and a password wordlist produced valid credentials, giving direct SSH access to the box as `gandalf`.

## Privilege Escalation: PwnKit (CVE-2021-4034)

Standard SUID enumeration:

```
find / -perm -4000 -type f 2>/dev/null
```

turned up a vulnerable `pkexec` binary. This is exploitable via **PwnKit** (CVE-2021-4034), a local privilege escalation bug in `polkit`'s `pkexec` that allows any local user to gain root through a memory-corruption flaw in argument parsing. A pre-compiled PwnKit exploit was run against the binary, yielding a root shell.

## Root

```
root@looz:/tmp# cd /root
root@looz:~# ls -la
...
-rw-r--r--  1 root root   33 Jun  7  2021 root.txt
-rw-r--r--  1 root root   50 Jun  7  2021 rundocker.sh
root@looz:~# cat root.txt
ab17850978e36aaf6a2b8808f1ded971
```

## Lessons Learned

- Don't assume the most obvious lead (exposed `wp-config.php` / Docker env leak) is the intended path — enumerate the website content itself for usernames before sinking time into a service that may be a decoy.
- Always check for leaked usernames on public-facing pages before brute-forcing blind.
- `pkexec`/PwnKit (CVE-2021-4034) remains a fast, reliable privesc check on any box with `polkit` installed — always worth a SUID sweep early.

---
*Flag: `ab17850978e36aaf6a2b8808f1ded971`*
