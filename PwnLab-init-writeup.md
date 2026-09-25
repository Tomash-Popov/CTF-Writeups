# PwnLab: init — Writeup

**Target:** PwnLab: init (VulnHub)
**Difficulty:** Beginner/Intermediate
**Techniques:** LFI via `php://filter`, directory fuzzing, credential reuse, file upload bypass, LFI-to-RCE via cookie inclusion, PATH hijacking, custom root-triggered script abuse

---

## 1. Recon

A web application was found running on port 80. Initial testing pointed toward a Local File Inclusion (LFI) vulnerability, but a classic `../../../etc/passwd`-style path traversal did not work — the application was filtering or restricting simple traversal payloads.

## 2. Source Disclosure via `php://filter`

Since raw traversal failed, the `php://filter` wrapper was used to read the source of an included file as base64 instead of having it executed:

```
php://filter/convert.base64-encode/resource=config
```

This bypassed the direct-execution problem entirely — instead of the LFI trying to *run* `config.php`, it returned the file's *source code*, base64-encoded.

## 3. Directory Fuzzing

In parallel, fuzzing the web root for hidden files/directories turned up `config.php` directly on the server, confirming the file existed and giving a second path to the same source.

## 4. Decoding Credentials

The decoded `config.php` source contained a set of base64-encoded credentials tied to three usernames:

| User | Encoded            | Decoded        |
|------|---------------------|----------------|
| kent | `Sld6WHVCSkpOeQ==`  | `JWzXuBJJNy`   |
| mike | `U0lmZHNURW42SQ==`  | `SIfdsTEn6I`   |
| kane | `aVN2NVltMkdSbw==`  | `iSv5Ym2GRo`   |

These credentials were used to authenticate to the application as each respective user.

## 5. File Upload Bypass

The application exposed a file upload feature intended for images only, with upload validation that proved non-trivial to bypass (extension/content checks). After working around the restriction, a PHP payload was uploaded disguised as a `.jpg`:

```
/upload/d29afda72e984ab307f8f0f685ca1ac4.jpg
```

## 6. LFI-to-RCE via the `lang` Cookie

The application's `index` page accepted a `lang` cookie to select an include file (a classic LFI sink via a language-switcher parameter). By pointing that cookie at the uploaded "image" containing PHP code, the file was included and executed server-side rather than rendered as an image:

```http
GET / HTTP/1.1
Host: 192.168.1.163
Cookie: lang=../upload/d29afda72e984ab307f8f0f685ca1ac4.jpg
```

This combination — upload bypass + LFI inclusion of the uploaded file — gave code execution and an initial low-privilege shell.

## 7. Lateral Movement: www-data → kane → mike

From the initial shell, a file named `msg` was discovered readable/writable in a way that mike's account would process it. Using **PATH hijacking** (placing a malicious binary earlier in `$PATH` than the one mike's process would call), execution was redirected to spawn a shell as **mike** when mike's script/cron ran.

## 8. Privilege Escalation: mike → root (`msg2root`)

On mike's account, a setup was found (`msg2root`) where writing a message would later be picked up and processed with root privileges. Testing this by writing:

```
hi && id
```

into the message channel and running `./ms2root` confirmed that the payload executed as **root**:

```
mike (0) root
```

## 9. Full Root Shell

With confirmed root command execution, the escalation was upgraded from one-shot command injection to a persistent root shell by setting the SUID bit on `/bin/bash`:

```bash
hi && chmod +s /bin/bash
```

Then invoking it with the `-p` flag to preserve privileges:

```bash
/bin/bash -p
```

This dropped into a root shell (`#`).

## 10. Flag

```bash
bash-4.3# cat flag.txt
```

Root flag retrieved — box complete.

---

## Summary / Chain

1. LFI blocked by naive traversal filter → bypassed via `php://filter` source disclosure
2. Directory fuzzing confirmed `config.php` existed
3. Base64 credentials decoded → valid logins for `kent`, `mike`, `kane`
4. Upload filter bypassed → PHP shell uploaded as `.jpg`
5. `lang` cookie LFI sink used to include and execute the uploaded file → initial foothold
6. PATH hijacking → pivot to `mike`
7. Root-executing helper script (`msg2root`) abused via command injection in a message file → root
8. SUID `chmod` on `/bin/bash` → stable root shell

## Lessons / Takeaways

- Blacklist-based traversal filters are frequently bypassed with PHP wrappers (`php://filter`, `php://input`, `data://`, `expect://`) rather than raw `../` sequences.
- Any user-controlled cookie/parameter that selects a file to `include()` is a full LFI sink, regardless of how "cosmetic" it looks (e.g. a language switcher).
- Upload filters that only check extension or a shallow content signature can usually be bypassed by disguising executable content as an image and triggering inclusion through a separate LFI vector.
- Never trust `$PATH` inside scripts/cron jobs run by higher-privileged users — always use absolute paths.
- Any root-privileged helper that processes user-writable input (files, messages, logs) without sanitization is a command injection vector.
