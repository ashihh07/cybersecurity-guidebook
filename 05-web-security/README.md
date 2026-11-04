# Chapter 5: Web Security

**Cybersecurity Guide Book | Roadmap step 5 of 12** | Author: [ashihh07](https://github.com/ashihh07)

[Back to roadmap](../README.md) | [Previous: Chapter 4 Web Technologies](../04-web-technologies/README.md) | [Download the PDF version](Chapter-05-Web-Security.pdf)

> Web attacks are not magic. Each one is a trusted-input mistake you already know how to spot. This chapter shows the mistake, the fix, and the footprint it leaves in the logs.

---

## About this chapter

Chapter 4 showed how a page, a cookie and a web server work. Now we break them on purpose, in a safe lab, so you can recognise the same flaws in code, in a scan report and in a SOC alert.

The roadmap lists four items:

- **SQL injection:** user input rewrites a database query.
- **XSS and CSRF:** attacker-controlled script runs in a victim's browser, or a victim's browser is made to act.
- **Authentication and authorization:** proving who you are, and what you may do.
- **OWASP Top 10:** the industry's shared list of the most serious web application risks.

### How each topic is built

- **Concept**: the flaw, in plain language, with a tiny example.
- **Attacker view**: how it is abused. Examples are classic textbook probes, for use only against targets you own or are allowed to test.
- **Defender view**: the fix in code, plus what to watch for in logs.
- **Labs**: run on code you wrote or on deliberately vulnerable practice apps.

### Contents

| # | Section | Roadmap item covered |
|---|---------|----------------------|
| 1 | [The OWASP Top 10](#1-the-owasp-top-10) | OWASP Top 10 |
| 2 | [Practise safely and legally](#2-practise-safely-and-legally) | Orientation |
| 3 | [SQL injection](#3-sql-injection) | SQL Injection |
| 4 | [Cross-site scripting (XSS)](#4-cross-site-scripting-xss) | XSS, CSRF |
| 5 | [Cross-site request forgery (CSRF)](#5-cross-site-request-forgery-csrf) | XSS, CSRF |
| 6 | [Authentication and authorization](#6-authentication-and-authorization) | Authentication and Authorization |
| 7 | [Spotting web attacks in logs](#7-spotting-web-attacks-in-logs) | SOC view of all of the above |
| 8 | [Hands-on labs](#8-hands-on-labs) | Practice for all of the above |
| 9 | [Quick revision sheet](#9-quick-revision-sheet) | Tables and commands |
| 10 | [Self-check quiz and answers](#10-self-check-quiz) | Test yourself |

> [!WARNING]
> **Ground rule. This chapter is the one where it matters most.**
> - Only attack applications you built yourself, or that exist to be attacked (OWASP Juice Shop, DVWA, PortSwigger Web Security Academy, TryHackMe, Hack The Box).
> - Never test a real website, a school or company system, or a friend's project without written permission. Even a "harmless" probe can be illegal.
> - Run vulnerable practice apps on `127.0.0.1` inside a VM. Never expose them to a network.

---

## 1. The OWASP Top 10

### Concept

The **OWASP Top 10** is a consensus list of the most critical web application security risks, published by the Open Worldwide Application Security Project. It is an awareness document, not a test checklist, and it appears in job descriptions, audit reports and interview questions. The current edition is **OWASP Top 10:2025**, released in November 2025 with the final version in January 2026. It replaced the 2021 edition.

| ID | Category (2025) | In plain words | Where in this book |
|----|-----------------|----------------|--------------------|
| **A01** | **Broken Access Control** | Users can do or see things they should not. SSRF (making the server fetch attacker-chosen URLs) is now part of this category | [Section 6](#6-authentication-and-authorization) |
| **A02** | **Security Misconfiguration** | Insecure defaults, open directories, verbose errors, missing headers | Chapter 4 |
| **A03** | **Software Supply Chain Failures** | Compromised or outdated dependencies, build systems and distribution | Chapter 3 (SRI, packages), Chapter 4 |
| **A04** | **Cryptographic Failures** | Weak or missing encryption, bad key handling, plain-text secrets | Chapter 1 (TLS) |
| **A05** | **Injection** | Input treated as code: SQL injection, XSS, command injection | Sections [3](#3-sql-injection) and [4](#4-cross-site-scripting-xss) |
| **A06** | **Insecure Design** | The design itself lacks a control, so no code fix is enough | Threat modeling, every chapter |
| **A07** | **Authentication Failures** | Weak login, credential stuffing, broken sessions and resets | [Section 6](#6-authentication-and-authorization) |
| **A08** | **Software or Data Integrity Failures** | Trusting code or data without verifying it (unsigned updates, unsafe deserialization) | Chapter 3 |
| **A09** | **Security Logging and Alerting Failures** | Attacks happen and nobody sees or acts on them | [Section 7](#7-spotting-web-attacks-in-logs) |
| **A10** | **Mishandling of Exceptional Conditions** | Poor error handling, logic errors, failing open when something breaks | Sections [3](#3-sql-injection) and [6](#6-authentication-and-authorization) |

> [!NOTE]
> Older books and tools still use the **2021** list (where SSRF was A10, and Injection was A03). If a report names "A03:2021", it means Injection. Always check which year a document uses. Source: [owasp.org/Top10](https://owasp.org/Top10/), licensed CC BY 3.0.

CSRF is not a separate Top 10 item. It sits under access control and authentication failures, but you must know it because it appears in every web assessment.

---

## 2. Practise safely and legally

### Where to practise

| Target | What it is | Setup |
|--------|-----------|-------|
| **PortSwigger Web Security Academy** | Free guided labs for every topic in this chapter, hosted by PortSwigger | Browser only, free account |
| **OWASP Juice Shop** | A deliberately insecure modern web shop with many challenges | Docker, or Node.js |
| **DVWA** (Damn Vulnerable Web Application) | Classic PHP/MySQL app with adjustable difficulty | Follow the project's install guide |
| **TryHackMe / Hack The Box** | Guided rooms and machines | Browser plus VPN |

### Start Juice Shop in your VM, on localhost only

```bash
sudo apt install docker.io
sudo systemctl enable --now docker
sudo docker run --rm -p 127.0.0.1:3000:3000 bkimminich/juice-shop
# open http://127.0.0.1:3000 in the browser inside the VM
```

`127.0.0.1:3000:3000` binds the app to your own machine only. Without `127.0.0.1:` Docker would publish it to your whole network.

### If you find a real bug

Stop. Do not read other people's data, do not "prove" it further. Look for `/.well-known/security.txt` or a bug bounty or vulnerability disclosure page on the site, and report it there. Good-faith researchers still get into legal trouble when they go beyond the minimum.

---

## 3. SQL injection

### Concept

Applications store data in databases and ask for it with **SQL**:

```sql
SELECT username, role FROM users WHERE username = 'alice' AND password = 'secret';
```

**SQL injection (SQLi)** happens when the application builds that text by gluing user input into it. The input then changes the **structure** of the query instead of being treated as data.

```python
# VULNERABLE: input becomes part of the SQL text
query = f"SELECT username, role FROM users WHERE username = '{u}' AND password = '{p}'"
```

If `u` is `admin' --`, the query becomes:

```sql
SELECT username, role FROM users WHERE username = 'admin' --' AND password = 'anything'
```

`--` starts a comment, so the password check disappears and the attacker is logged in as `admin`. The quote character was the attacker's way out of the data and into the code.

### What attackers can do with it

| Technique | Idea |
|-----------|------|
| **Authentication bypass** | Make the login condition always true or comment it out |
| **Data extraction (UNION)** | Append a second `SELECT` so the response carries data from other tables |
| **Blind SQLi** | The page shows nothing useful, so the attacker asks yes/no questions through page differences or response delays and rebuilds data bit by bit |
| **Error-based** | Database error messages leak table names or values |
| **Beyond data** | Depending on the database and its privileges, reading or writing files and sometimes running commands |

SQL injection is old and well understood, and it still appears. It is high impact because the database holds the crown jewels.

### A safe way to see it: your own code

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE users (username TEXT, password TEXT, role TEXT)")
db.execute("INSERT INTO users VALUES ('alice','s3cret','user'), ('admin','Adm1n!','admin')")

def login_vulnerable(u, p):
    q = f"SELECT username, role FROM users WHERE username = '{u}' AND password = '{p}'"
    print("SQL:", q)
    return db.execute(q).fetchone()

def login_safe(u, p):
    q = "SELECT username, role FROM users WHERE username = ? AND password = ?"
    return db.execute(q, (u, p)).fetchone()          # input is bound as DATA, never as SQL

print(login_vulnerable("alice", "s3cret"))            # normal login works
print(login_vulnerable("admin' --", "anything"))      # comment removes the password check
print(login_vulnerable("x", "' OR '1'='1"))           # condition becomes always true
print(login_safe("admin' --", "anything"))            # None: treated as a strange username
```

This demo stores plain-text passwords only to keep it short. Real applications store salted hashes (section 6).

> [!CAUTION]
> **Attacker view**
> - Add a single quote `'` to a parameter and watch for a database error or a changed page. That is the first test.
> - Try `' OR '1'='1`, a comment (`--` or `#`), and `UNION SELECT` to line up columns and pull data from other tables.
> - Use delay functions to test blind injection when nothing is displayed.
> - Automated tools (such as `sqlmap`) do all of this at speed and announce themselves in the `User-Agent` unless changed.

> [!TIP]
> **Defender view**
> - **Parameterised queries (prepared statements) everywhere.** Pass values separately from the SQL, as `login_safe` does. This is the real fix.
> - ORMs help, but raw-SQL escape hatches inside them can be injectable too.
> - **Least privilege:** the application's database account should not be an admin and should not read tables it does not need.
> - **Hide errors:** show users a generic message and log details privately (A10 and A02).
> - Validate the shape of input (an ID must be digits) as an extra layer, not as the main defence.
> - A WAF can block obvious probes, but it is a safety net, not a fix.
> - **Detect:** quote characters, `UNION SELECT`, `information_schema` and sleep functions in URLs, bursts of server errors (500) after odd input, and `sqlmap` user agents.

---

## 4. Cross-site scripting (XSS)

### Concept

In Chapter 3 you saw `innerHTML` run attacker text. **XSS** is that mistake at scale: an application puts untrusted input into a page without making it safe for that spot, so the browser runs the attacker's script **with the victim's session and the site's trust**.

| Type | Where the payload lives | Example |
|------|------------------------|---------|
| **Reflected** | In the request (usually the URL). The server echoes it straight back | A search page showing `You searched for: ...` |
| **Stored** | Saved by the server (comment, profile, ticket) and shown to every visitor | A forum post that runs script for each reader |
| **DOM-based** | Never reaches the server. Client-side JavaScript writes it into the page | `innerHTML = location.hash` |

A script running in the victim's page can read anything on that page, act as the user, change what the page shows, and steal tokens that are not protected (Chapter 4: `HttpOnly`, `localStorage`).

### Context decides the fix

The same input needs different handling depending on where it lands:

| Output location | Danger | Correct handling |
|-----------------|--------|------------------|
| **HTML text** | `<` and `>` start tags | HTML-encode (`&lt;`, `&gt;`) |
| **HTML attribute** | A quote closes the attribute and adds `onclick=...` | Encode quotes, and always quote attributes |
| **Inside JavaScript** | Closing the string or the script block | Avoid; pass data as JSON, not by string-gluing |
| **URL** | `javascript:` links | Allow only `http` and `https` schemes |

### A safe way to see it: your own server

Save as `xss_server.py` and run `python3 xss_server.py`:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs
import html

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        url = urlparse(self.path)
        q = parse_qs(url.query).get("q", [""])[0]
        out = html.escape(q) if url.path == "/safe" else q    # /safe encodes, / does not
        page = f"<h1>Search results</h1><p>You searched for: {out}</p>".encode()
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(page)))
        self.end_headers()
        self.wfile.write(page)

HTTPServer(("127.0.0.1", 8000), H).serve_forever()
```

Open `http://127.0.0.1:8000/?q=<img src=x onerror=alert(1)>` and compare with `http://127.0.0.1:8000/safe?q=<img src=x onerror=alert(1)>`. The first runs the script, the second shows harmless text.

> [!CAUTION]
> **Attacker view**
> - Probe every place input comes back: search boxes, error messages, profile fields, file names, URL fragments.
> - A reflected link is sent to the victim by email or chat. A stored payload waits for visitors, including admins.
> - Goal examples: act as the logged-in user, show a fake login box inside the real site, capture keystrokes, or take a session token that lacks `HttpOnly`.

> [!TIP]
> **Defender view**
> - **Encode output for its context.** Modern frameworks (React, Django and Jinja templates, Angular) escape by default. Bypassing them (`dangerouslySetInnerHTML`, `|safe`, `innerHTML`) is where bugs appear.
> - If you must allow rich HTML, sanitise it with a maintained library (such as DOMPurify), never with your own regex.
> - **Content-Security-Policy** limits what an injected script can do. Example header: `Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'`
> - Mark session cookies `HttpOnly; Secure; SameSite`, so a successful XSS cannot simply take the session.
> - **Detect:** `<script`, `%3Cscript`, `onerror=`, `javascript:` in request URLs, plus CSP violation reports sent by browsers.

---

## 5. Cross-site request forgery (CSRF)

### Concept

Browsers attach cookies automatically to every request to a site, **no matter which page triggered the request**. CSRF abuses that: a malicious page makes the victim's browser send a request to a site where the victim is already logged in.

```html
<!-- on the attacker's page (bank.example is a lab target) -->
<form action="http://bank.example/transfer" method="post" id="f">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="500">
</form>
<script>document.getElementById("f").submit();</script>
```

The victim only has to open the page. Their browser sends the POST with their session cookie, and the bank sees a valid, logged-in request.

CSRF needs three things: **cookie-based authentication**, a **state-changing action** (transfer, password change, delete), and **parameters the attacker can predict**.

### XSS vs CSRF

| | XSS | CSRF |
|---|-----|------|
| **Attacker's script runs** | In the victim's page, on the real site | On the attacker's own site |
| **Can read the response** | Yes | No (same-origin policy blocks it) |
| **Typical damage** | Full account actions, data theft | One forged action per request |
| **Main defence** | Output encoding, CSP | Tokens and `SameSite` cookies |

> [!CAUTION]
> **Attacker view**
> - Find an action that changes something and needs only cookies. Build a form or image tag that fires it.
> - Deliver it through email, a chat link, or any page the victim visits (including one hosted by a stored-XSS bug on another site).
> - Changing an email address or password via CSRF often leads to full account takeover.

> [!TIP]
> **Defender view**
> - **`SameSite=Lax` or `Strict`** on session cookies. Browsers now often default to `Lax`, but set it explicitly.
> - **Anti-CSRF tokens:** a random, per-session (or per-request) secret embedded in each form and checked on the server. The attacker's page cannot read it. Example field: `<input type="hidden" name="csrf_token" value="b7e1...random">`. This hidden field is the **opposite** of the tamperable `role` field from Chapter 4: it is an unguessable secret the server verifies.
> - Check the `Origin` or `Referer` header on state-changing requests.
> - Never change state with `GET`. Require re-entering the password for sensitive actions.
> - **Detect:** state-changing requests with a foreign or missing `Origin`/`Referer`, or without a valid token.

---

## 6. Authentication and authorization

### Concept

| | Authentication (authN) | Authorization (authZ) |
|---|------------------------|------------------------|
| **Question** | Who are you? | What are you allowed to do? |
| **Examples** | Password, one-time code, passkey | Can this user view invoice 1043? Open `/admin`? |
| **OWASP** | A07 Authentication Failures | A01 Broken Access Control |
| **Typical failure** | Guessable or stolen credentials, broken resets | Missing checks on the server |

### Authentication attacks

| Attack | What it is | Telltale sign |
|--------|-----------|---------------|
| **Brute force** | Many passwords against one account | Many failures for one username |
| **Password spraying** | One or a few common passwords against many accounts | One IP, many usernames, few tries each |
| **Credential stuffing** | Leaked username and password pairs from other breaches replayed | Many accounts, many IPs, a few successes |
| **Account enumeration** | Different messages or timings reveal which users exist | "User not found" vs "Wrong password" |
| **Weak reset flow** | Guessable or reusable reset tokens, security questions | Resets followed by logins from new places |
| **MFA fatigue / phishing** | Spamming push prompts, or relaying codes through a fake site | Many MFA prompts, then one approved |

### Doing authentication properly

- **Store passwords as salted hashes** with a slow algorithm built for it: **Argon2id**, **bcrypt** or **scrypt**. Never plain text, MD5, SHA-1 or a bare fast hash.
- **Favour length over complexity rules.** Check new passwords against lists of breached passwords, and do not force frequent rotation.
- **Multi-factor authentication.** Authenticator apps are good; hardware keys and **passkeys (FIDO2)** resist phishing.
- **Rate limit and monitor** login attempts, per account and per IP. Be careful with hard lockouts, which let attackers lock real users out.
- **Same message for every failure:** "Invalid username or password."
- **Reset tokens** must be random, single-use, short-lived, and sent only to the account's verified contact.
- Session handling from Chapter 4: random IDs, new ID on login, short expiry, destroy on logout.

### Authorization attacks

**Broken access control** is the top risk because it is easy to write and hard to spot. The application checks that you are logged in, but not that this particular thing is yours.

| Flaw | Example | Kind |
|------|---------|------|
| **IDOR** (insecure direct object reference) | Change `/api/invoices/1042` to `/api/invoices/1043` and see someone else's invoice | Horizontal: same level, other user |
| **Privilege escalation** | A normal user calls `/admin/delete-user` and it works | Vertical: to a higher role |
| **Forced browsing** | Opening `/admin`, `/backup`, `/debug` directly because no link points there | Missing function-level check |
| **Parameter tampering** | Sending `role=admin` or `isAdmin=true` and the server believes it | Trusting client data (Chapter 4) |
| **SSRF** | A "fetch this URL" feature makes the server request internal addresses or cloud metadata | Server tricked into acting for the attacker |

```python
# VULNERABLE: trusts the id in the URL
def get_invoice(invoice_id, current_user):
    return db.query("SELECT * FROM invoices WHERE id = ?", (invoice_id,))

# FIXED: ownership checked on the server, on every request
def get_invoice(invoice_id, current_user):
    row = db.query("SELECT * FROM invoices WHERE id = ? AND owner_id = ?",
                   (invoice_id, current_user.id))
    if row is None:
        abort(404)               # same answer whether it is missing or not yours
    return row
```

> [!CAUTION]
> **Attacker view**
> - Log in as two ordinary users. Take a request from user A and replay it with user B's session, or change the ID. If it works, there is no ownership check.
> - Try admin URLs and API methods with a low-privilege account. Try removing the session cookie entirely.
> - Run credential stuffing with leaked lists, or spray one weak password across every account.
> - For SSRF: submit internal addresses (such as `127.0.0.1` or a cloud metadata IP) to any feature that fetches a URL.

> [!TIP]
> **Defender view**
> - **Deny by default.** Every endpoint checks both "who are you" and "may you do this to this object", on the server, every time.
> - Centralise authorization in one place instead of repeating checks. Write tests that log in as user A and try user B's data.
> - Unguessable IDs (random UUIDs) are good **depth**, never a replacement for the ownership check.
> - For features that fetch URLs: allow-list destinations, block internal ranges, and do not return raw responses.
> - Fail closed: if the permission check errors, the answer is "no" (A10).
> - **Detect:** one account walking sequential IDs, bursts of 401/403, many usernames from one IP, logins from impossible locations, and new devices after a password reset.

---

## 7. Spotting web attacks in logs

### Concept

From Chapter 4 you can read an access log. Here are the patterns left by this chapter's attacks. These are starting points: attackers encode and vary payloads, so tune and expect false positives.

| Attack | What the log shows | Starter search |
|--------|--------------------|----------------|
| **SQL injection** | Quotes, `UNION SELECT`, comments, sleep calls in URLs | `grep -iE "union(%20\|\+\|\s)+select\|%27\|information_schema\|sleep\(" access.log` |
| **XSS** | Script tags and event handlers in URLs | `grep -iE "<script\|%3cscript\|onerror=\|onload=\|javascript:" access.log` |
| **Scanners and tools** | Known tool names, huge numbers of 404s | `grep -iE "sqlmap\|nikto\|gobuster\|wpscan\|nmap" access.log` |
| **Brute force** | Many `POST /login` with `401` from one IP | `awk '$6=="\"POST" && $7=="/login" && $9==401 {print $1}' access.log \| sort \| uniq -c \| sort -nr \| head` |
| **Rate bursts** | Many requests per IP per minute | `awk '{print $1, substr($4,2,17)}' access.log \| sort \| uniq -c \| sort -nr \| head` |
| **IDOR walking** | One client requests consecutive IDs | `awk '$7 ~ /^\/api\/invoices\/[0-9]+/ {print $1, $7}' access.log \| sort -u \| head -50` |
| **Path traversal** | `../` or `%2e%2e` sequences | `grep -E "\.\./\|%2e%2e" access.log` |

> [!IMPORTANT]
> **Know the limits of access logs.** They record the URL, not the **POST body**. A SQL injection or credential-stuffing attempt sent by POST will not show its payload. You need application logs, a WAF, or request-body logging on sensitive endpoints. Also, an app that returns `200` on a failed login hides brute force from status-code rules, so compare response sizes or use application authentication logs.

> [!TIP]
> **Defender view: what makes a good web alert**
> - Combine signals: probe strings **followed by** a `500` or a changed response size is much stronger than a probe alone.
> - Correlate across layers (Chapter 4, section 7): WAF, proxy, application and database logs for the same time and IP.
> - Alerting matters as much as logging. OWASP A09:2025 is about attacks nobody was alerted to.

---

## 8. Hands-on labs

Use only your own code and deliberately vulnerable practice apps. Write up every lab (goal, steps, screenshot, the fix, what you learned). A good write-up explains the **defence**, not just the exploit.

### Lab 1: Set up the practice lab

Start Juice Shop on localhost (section 2). Create a free PortSwigger Web Security Academy account.

**Goal:** Juice Shop loads at `http://127.0.0.1:3000`, and you have confirmed with `ss -tulpn` that it listens only on `127.0.0.1`.

### Lab 2: SQL injection in your own code

Run the code from [section 3](#3-sql-injection). Then add one more function that uses string formatting and a `LIKE` search (`WHERE username LIKE '%{term}%'`) and break it, then fix it with a bound parameter.

**Goal:** each vulnerable function is broken by one input, and each safe version resists the same input.

### Lab 3: SQL injection on a practice app

Complete the PortSwigger lab **"SQL injection vulnerability in WHERE clause allowing retrieval of hidden data"** and the Juice Shop login challenge.

**Goal:** explain which character broke out of the data, and write the one-line fix (parameterised query).

### Lab 4: Reflected XSS in your own server

Run `xss_server.py` from [section 4](#4-cross-site-scripting-xss). Trigger the vulnerable path and the `/safe` path. Check the response with `curl -i`. Then add a CSP header to the server and see how the browser's behaviour changes.

**Goal:** the alert fires on `/` but not on `/safe`, and you can explain what `html.escape` changed in the page source.

### Lab 5: XSS on a practice app

Complete the PortSwigger lab **"Reflected XSS into HTML context with nothing encoded"** and the Juice Shop DOM XSS challenge.

**Goal:** identify the sink (where input reached the page) and the correct encoding or API (`textContent`) for it.

### Lab 6: CSRF

Complete the PortSwigger lab **"CSRF vulnerability with no defenses"**. Then, in DevTools, look at the `SameSite` attribute on the lab's session cookie and explain what would change if it were `Strict`.

**Goal:** a short note describing the three CSRF conditions and the two main defences.

### Lab 7: Authentication and access control

Complete the PortSwigger labs **"Username enumeration via different responses"**, **"Unprotected admin functionality"** and **"User ID controlled by request parameter"**.

**Goal:** for each, write the missing server-side check in one sentence.

### Lab 8: Catch yourself in the logs

```bash
python3 xss_server.py 2> access.log &            # the server logs requests to stderr
curl "http://127.0.0.1:8000/?q=%3Cscript%3Ealert(1)%3C/script%3E"
curl "http://127.0.0.1:8000/?q=1%27%20OR%20%271%27%3D%271"
curl -A "sqlmap/1.7" "http://127.0.0.1:8000/?q=test"
cat access.log
grep -iE "<script|%3cscript|onerror=|javascript:" access.log
grep -iE "union(%20|\+|\s)+select|%27|information_schema|sleep\(" access.log
grep -iE "sqlmap|nikto|gobuster|nmap" access.log
kill %1
```

**Goal:** all three probes are caught by your searches. Write the detection rule for each in plain English, and note one way an attacker could evade it (for example by different encoding).

> [!NOTE]
> **Next step on TryHackMe:** work through the OWASP Top 10 and Juice Shop rooms, then the Web Fundamentals path, and write each up the same way. Keep your PortSwigger progress in your notes. It shows steady practice.

---

## 9. Quick revision sheet

### The attacks

| Attack | Root cause | Best fix |
|--------|-----------|----------|
| **SQL injection** | Input glued into SQL | Parameterised queries, least-privilege DB account |
| **XSS** | Input placed in a page without context-aware encoding | Output encoding, safe APIs, CSP, `HttpOnly` |
| **CSRF** | Cookies sent automatically on cross-site requests | `SameSite`, anti-CSRF tokens, origin checks |
| **Broken access control / IDOR** | Server does not check ownership or role | Deny by default, check every request server-side |
| **Authentication failure** | Weak or reusable credentials, poor resets | Argon2id/bcrypt, MFA, rate limits, uniform errors |
| **SSRF** | Server fetches attacker-chosen URLs | Allow-list destinations, block internal ranges |

### OWASP Top 10:2025 in one line each

**A01** Broken Access Control, **A02** Security Misconfiguration, **A03** Software Supply Chain Failures, **A04** Cryptographic Failures, **A05** Injection, **A06** Insecure Design, **A07** Authentication Failures, **A08** Software or Data Integrity Failures, **A09** Security Logging and Alerting Failures, **A10** Mishandling of Exceptional Conditions.

### Commands

| Task | Command |
|------|---------|
| **Probe patterns in a log** | `grep -iE "pattern1\|pattern2" access.log` |
| **Count by client** | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -nr \| head` |
| **Failed logins by IP** | `awk '$9==401 {print $1}' access.log \| sort \| uniq -c \| sort -nr` |
| **Check a response's headers** | `curl -I URL` |
| **Run a practice app on localhost only** | `docker run --rm -p 127.0.0.1:3000:3000 bkimminich/juice-shop` |

### One-line summaries

- **Never build SQL by gluing strings.** Bind parameters.
- **Encode output for where it lands.** The browser should never be able to mistake data for code.
- **Cookies are sent automatically.** That is why CSRF needs tokens and `SameSite`.
- **authN is who, authZ is what.** Checking login is not checking ownership.
- **Logs show URLs, not POST bodies.** Plan other sources.
- **Only attack what you own or are allowed to test.**

---

## 10. Self-check quiz

Answer without looking back, then open the key.

1. In `WHERE username = 'admin' --' AND password = '...'`, what does `--` do and why does it matter?
2. What is the real fix for SQL injection?
3. Name the three types of XSS.
4. Why does `textContent` prevent the DOM XSS that `innerHTML` allows?
5. Why can CSRF not read the victim's data, while XSS can?
6. Name two defences against CSRF.
7. What is an IDOR, and what single server-side check prevents it?
8. How does password spraying differ from brute force?
9. How should passwords be stored?
10. Why can access logs miss a SQL injection sent by POST?

<details>
<summary>Show answer key</summary>

1. It starts a SQL comment, so everything after it (the password check) is ignored. It lets input change the structure of the query.
2. Parameterised queries (prepared statements), so input is always bound as data. Least privilege and error hiding add depth.
3. Reflected, stored and DOM-based.
4. `textContent` inserts text as plain text and never parses it as HTML or runs scripts. `innerHTML` parses the string as live HTML.
5. The same-origin policy stops the attacker's page from reading responses from another site. XSS runs inside the victim site's own page, so it can read everything there.
6. Any two of: `SameSite` cookies, anti-CSRF tokens, `Origin`/`Referer` checks, re-authentication for sensitive actions, not using GET for changes.
7. Insecure direct object reference: an object ID in the request is trusted. The server must check that the logged-in user is allowed to access that specific object, on every request.
8. Brute force tries many passwords against one account. Spraying tries one or a few common passwords against many accounts, to avoid lockouts.
9. As salted hashes using Argon2id, bcrypt or scrypt. Never plain text or fast hashes like MD5 or SHA-1.
10. Access logs record the URL and headers, not the request body, so a POST payload is not visible. You need application, WAF or body logging.

</details>

---

## What comes next

**Chapter 6: Ethical Hacking Basics.** You have seen how individual web flaws work. Next you learn the method that finds them across a whole target: reconnaissance, footprinting, scanning with Nmap, and enumeration, always inside legal scope.

[Back to roadmap](../README.md) | [Previous: Chapter 4 Web Technologies](../04-web-technologies/README.md) | [Download the PDF version](Chapter-05-Web-Security.pdf)
