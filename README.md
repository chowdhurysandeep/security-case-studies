# Security Case Studies

A hands-on collection of web vulnerability case studies conducted on intentionally vulnerable applications. Each case study walks through the vulnerability, exploitation steps with screenshots, and key takeaways.

> ⚠️ **Disclaimer:** All techniques demonstrated here are performed on intentionally vulnerable applications in a controlled local environment. Never attempt these on systems you do not own or have explicit permission to test.

---

## Labs

### 1. DVWA (Damn Vulnerable Web Application)

| # | Vulnerability | Description |
|---|--------------|-------------|
| 1 | Brute Force | Automating login attacks using Burp Suite Intruder and a custom Python script to bypass CSRF token protection |
| 2 | Command Injection | Chaining OS commands via `;`, `\|`, and no-space `\|cmd` across all three security levels |
| 3 | CSRF | Forging requests via hidden forms, Referer header bypass, and anti-CSRF token theft via XSS |
| 4 | File Inclusion (LFI) | Reading `/etc/passwd` via direct path, nested `....//` bypass, and `file://` URI scheme |
| 5 | File Upload | Uploading PHP web shells via direct upload, Content-Type spoofing, and exiftool EXIF injection chained with LFI |
| 6 | SQL Injection | UNION-based dump, numeric injection bypass, and pop-up session variable trick |
| 7 | Weak Session IDs | Exploiting sequential integers, Unix timestamps, and MD5 hashes of small numbers |
| 8 | XSS DOM | Script injection via URL, hash fragment bypass, and HTML tag breakout |
| 9 | XSS Reflected | Script tag injection and `img onerror` event handler bypass |
| 10 | XSS Stored | Persistent script injection and `img onerror` bypass stored in the database |

---

### 2. More Coming Soon...

---

## Tools Used

- Kali Linux
- Burp Suite
- SQLMap
- Python
- exiftool
- Firefox Developer Tools
- nano / terminal

---

## Purpose

These case studies are created for educational purposes to understand how common web vulnerabilities work, how they are exploited in real scenarios, and how they can be mitigated.
