# 📝 CTF Write-Up: XXE Injection — picoCTF

---

## 🏷️ Challenge Info

| Field | Details |
|---|---|
| **Challenge Name** | XML External Entity |
| **Category** | Web Exploitation |
| **Flag** | `picoCTF{XML_3xtern@l_3nt1t1ty_540f4f1e}` |
| **Difficulty** | Medium |

---

## 🔍 Reconnaissance

The first step was to **intercept HTTP traffic** using a proxy tool (e.g., Burp Suite). While browsing the target web application, I noticed that the application was sending data in **XML format** in the request body — specifically a `<data>` tag containing an `<ID>` field.

**Original request body looked like:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<data><ID>1</ID></data>
```

This immediately raised suspicion — XML parsers can be vulnerable to **XXE (XML External Entity) Injection** if external entities are not disabled.

---

## 🧪 Exploitation

### Step 1 — Identify XML Input Point

By intercepting the request in Burp Suite, I confirmed the application accepts and **parses XML input** without sanitization.

### Step 2 — Craft the XXE Payload

I injected a malicious **DOCTYPE declaration** that defines an external entity pointing to a local file on the server:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<data><ID>&xxe;</ID></data>
```

**How it works:**
- `<!DOCTYPE foo [...]>` — declares a custom DTD (Document Type Definition)
- `<!ENTITY xxe SYSTEM "file:///etc/passwd">` — defines an external entity that reads `/etc/passwd`
- `&xxe;` — references the entity, causing the parser to **substitute it with the file contents**

### Step 3 — Send the Payload

Forwarded the modified request and observed the server **reflected the file contents** in the response.

### Step 4 — Read the Flag

Inside the `/etc/passwd` output, the last line revealed the flag embedded in the `picoctf` user entry:

```
picoctf:x:1001:picoCTF{XML_3xtern@l_3nt1t1ty_540f4f1e}
```

---

## 🚩 Flag

```
picoCTF{XML_3xtern@l_3nt1t1ty_540f4f1e}
```

---

## 🛡️ Mitigation / Prevention

| Fix | Description |
|---|---|
| **Disable external entities** | Configure XML parser to reject `SYSTEM` and `PUBLIC` entity declarations |
| **Use safe parsers** | Use libraries with XXE protection enabled by default |
| **Input validation** | Reject or sanitize XML input with DOCTYPE declarations |
| **Least privilege** | Run the web server with minimal file system access |

**Example fix in Python (lxml):**
```python
from lxml import etree

parser = etree.XMLParser(
    resolve_entities=False,  # Disable entity resolution
    no_network=True          # Block network access
)
tree = etree.fromstring(xml_input, parser)
```

---

## 📚 Lessons Learned

> Always intercept and analyze **all request formats** — not just form data. XML parsers are notoriously dangerous when external entities are enabled. XXE can lead to **LFI, SSRF, and even RCE** in some cases.

---
