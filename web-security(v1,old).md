# Web Vulnerability Assessment Agent  
_A structured checklist for the most common web application vulnerabilities_

This document is designed to be used as the “brain” of an agent that checks websites for common security vulnerabilities. It is organized by vulnerability category and describes:

- What the vulnerability is  
- Why it matters  
- How an automated agent can detect or assist in detecting it  
- What to recommend when it’s found  

It is heavily aligned with industry best practices such as the OWASP Top 10 and common security testing methodologies.

---

## 1. Injection (SQL, NoSQL, OS Command, etc.)

### 1.1 What It Is
Injection flaws occur when untrusted input is sent to an interpreter (database, OS shell, LDAP, NoSQL engine, etc.) as part of a command or query. The interpreter executes unintended commands or accesses data without proper authorization.

### 1.2 Typical Impact
- Data theft, modification, or deletion  
- Authentication bypass  
- Full system compromise (for OS command injection)

### 1.3 Signals & Checks for the Agent

**Input Discovery**
- Enumerate all parameters:
  - Query parameters: `?id=1`
  - POST body fields (form, JSON, XML)
  - Headers: `X-Forwarded-For`, `User-Agent`, `Referer`
  - Cookies
- Identify input sinks that likely reach interpreters:
  - URLs or endpoints that look like they query data: `/item?id=1`, `/search?q=test`
  - Endpoints with filters/sorts/pagination

**SQL Injection Heuristics**
- Send benign SQLi probes in each parameter and compare responses:
  - `'`  
  - `"`  
  - `')`  
  - `1' OR '1'='1`  
  - `1' AND '1'='2`  
  - `1'--`  
- Look for:
  - HTTP 500 errors or stack traces
  - Database error keywords in response:
    - `SQL syntax`, `mysql`, `psql`, `ORA-`, `SQLite`, `ODBC`, `syntax error at or near`
  - Response body/content-length changes between:
    - `1' AND '1'='1` (true) vs `1' AND '1'='2` (false)

**Time-Based Blind SQL Injection**
- For parameters that do not show errors but are suspected:
  - Use payloads that introduce delays:
    - MySQL: `1' AND SLEEP(5)--`
    - PostgreSQL: `1' AND pg_sleep(5)--`
    - SQL Server: `1'; WAITFOR DELAY '0:0:5'--`
  - Measure round-trip time for baseline vs injected requests
  - If consistent delays >= threshold (e.g., +4 seconds) only when payload present, flag as probable SQLi

**NoSQL Injection**
- Look for JSON-based endpoints, e.g. `Content-Type: application/json`
- Try MongoDB-style or similar payloads:
  - `{"username": {"$ne": null}, "password": {"$ne": null}}`
  - `{"$where": "sleep(5000)"}`
- Look for:
  - Authentication bypass (login success with obvious payloads)
  - Time delays when using `$where` or equivalent constructs

**OS Command Injection**
- Look for parameters that might be passed to `ping`, `nslookup`, `ffmpeg`, `convert`, etc.
- Try chaining operators:
  - `; sleep 5`
  - `&& sleep 5`
  - `| sleep 5`
- Detect:
  - Time delays
  - Error messages containing command output

### 1.4 Agent Recommendations
When a parameter is flagged:
- Classify severity:  
  - Data read only? High  
  - Potential RCE? Critical
- Provide remediation guidance:
  - Use parameterized queries / prepared statements
  - Avoid string concatenation for building queries/commands
  - Implement strict allow-lists for command arguments
  - Centralize input validation and encoding

---

## 2. Broken Authentication & Session Management

### 2.1 What It Is
Weaknesses in login, logout, session handling, and credential management that allow attackers to impersonate other users.

### 2.2 Typical Impact
- Account takeover  
- Privilege escalation  
- Persistent access via stolen sessions

### 2.3 Signals & Checks for the Agent

**Transport Security**
- Check if login forms are served over HTTPS only:
  - If the login page or form action is `http://`, flag as high risk.
- Check secure cookie flags:
  - Session cookies should have `Secure` and `HttpOnly` set.
  - For modern apps, `SameSite` should be set (`Lax` or `Strict`).

**Password Handling (Black-box Heuristics)**
- Test for weak password policy indicators:
  - Accepts very short passwords (e.g., `a` or `123456`)
  - No rate limits on login attempts (perform controlled small bursts)
- Look for password reset flows:
  - Does the system send passwords in clear text via email? (if observable in UI docs)
  - Are reset tokens long, opaque, and one-time?

**Session Management**
- Verify:
  - Session ID changes after login (session fixation check)
  - Session ID changes after logout/login again
- Check cookie properties:
  - `Set-Cookie` should not expose IDs that look predictable (e.g., small integers, short base64 strings)
- Attempt:
  - Use an old session ID after logout to see if it still works (where safely testable)

**Account Enumeration**
- Submit invalid usernames or emails and observe:
  - Distinct error messages: `User not found` vs `Incorrect password`
  - Timing differences

### 2.4 Agent Recommendations
If issues are detected:
- Enforce HTTPS for all auth-related pages and APIs
- Ensure:
  - `Secure`, `HttpOnly`, `SameSite` flags on cookies
  - Session regeneration on login and privilege change
  - Automatic session invalidation on logout and inactivity
- Implement:
  - Rate limiting, lockouts, and MFA
  - Uniform error messages for login failures

---

## 3. Sensitive Data Exposure / Cryptographic Failures

### 3.1 What It Is
Improper protection of sensitive data (credentials, PII, financial data, tokens).

### 3.2 Typical Impact
- Credential theft  
- Identity theft  
- Regulatory non-compliance (GDPR, PCI-DSS, etc.)

### 3.3 Signals & Checks for the Agent

**Transport Layer**
- Check TLS:
  - Ensure `https://` is used site-wide for authenticated areas
  - Warn if mixed content detected (HTTPS page including HTTP scripts/images)
- Note if HSTS is present:
  - `Strict-Transport-Security` header

**Data in Responses**
- Search responses for patterns:
  - Plain-text passwords or secrets
  - Credit card patterns (e.g., 13–19 digit numbers with Luhn check)
  - Government ID formats (where known and allowed)
- Look for:
  - Exposed API keys, access tokens, secret keys in HTML, JS, or error pages

**Caching & Headers**
- On sensitive endpoints (login, account settings, payment):
  - Check `Cache-Control`, `Pragma`, and `Expires`:
    - Should have `no-store` or `no-cache` for highly sensitive content
- Check for:
  - `X-Content-Type-Options: nosniff`
  - `Content-Security-Policy` presence (also helps other categories)

### 3.4 Agent Recommendations
- Require HTTPS with strong TLS configurations
- Remove sensitive data from URLs and client-side logs
- Ensure:
  - Proper encryption of stored sensitive data using modern algorithms
  - Appropriate security headers on sensitive responses
- Avoid exposing secrets in front-end code or configuration files

---

## 4. Broken Access Control

### 4.1 What It Is
When users can access data or actions they are not authorized for, often due to missing or flawed permission checks.

### 4.2 Typical Impact
- Horizontal privilege escalation (user A sees user B’s data)  
- Vertical privilege escalation (normal user becomes admin)

### 4.3 Signals & Checks for the Agent

**IDOR (Insecure Direct Object Reference)**
- Detect endpoints with identifiers:
  - `/user/123`, `/invoice?id=1001`, `/orders/42`
- After authenticating as a low-privilege user:
  - Attempt to change `123` to `124`, etc.
  - Compare:
    - HTTP status codes
    - Response size and key content markers
- If other users’ data becomes visible, flag as IDOR.

**Function-Level Access Control**
- Enumerate all links and actions visible to a low-privilege user.
- Guess admin or management endpoints:
  - `/admin`, `/manage`, `/config`, `/backup`
- Try accessing them as non-admin:
  - If accessible or partially accessible, flag.

**Method & Parameter Tampering**
- For operations like updating roles or prices:
  - Modify parameters:
    - Change `role=user` to `role=admin`
    - Change `price=10` to `price=1`
- If the server accepts unauthorized changes, flag.

### 4.4 Agent Recommendations
- Enforce server-side access checks for every request, not just client-side UI checks.
- Use centralized authorization logic rather than scattered checks.
- Avoid relying on obscurity (hidden links, client-side conditionals).

---

## 5. Cross-Site Scripting (XSS)

### 5.1 What It Is
Injection of malicious scripts into web pages viewed by other users.

### 5.2 Typical Impact
- Session hijacking  
- Credential theft  
- Browser-side malware injection

### 5.3 Signals & Checks for the Agent

**Reflected XSS**
- For each parameter:
  - Inject harmless scripts or markers:
    - `<script>alert(1)</script>`
    - `"><svg onload=alert(1)>`
    - `"><img src=x onerror=alert(1)>`
  - Observe responses:
    - Does the payload appear unencoded?
    - Are special characters (`<`, `>`, `"`, `'`) HTML-encoded?

**Stored XSS**
- Look for input fields that save data:
  - Comments, profile fields, messages, support tickets
- Submit benign payloads and revisit affected pages:
  - If payload is stored and executed, flag stored XSS.

**DOM-Based XSS**
- Identify pages with heavy JavaScript and client-side templating.
- Check for:
  - Sinks: `innerHTML`, `document.write`, `eval`, `location.hash` reflection
- Inject markers into fragments like URL hash and query and observe DOM changes.

**CSP & Defenses**
- Check for `Content-Security-Policy`:
  - Absence of CSP is not a vulnerability itself but reduces protection.
- Check for output encoding:
  - Ensure user-supplied data is encoded according to context (HTML, attribute, JS, URL).

### 5.4 Agent Recommendations
- Recommend context-appropriate output encoding/escaping.
- Suggest:
  - Using safe templating engines
  - Avoiding `eval` and dangerous sinks
  - Deploying a strict CSP where feasible

---

## 6. Cross-Site Request Forgery (CSRF)

### 6.1 What It Is
Forcing a user’s browser to send unwanted requests to a site where they are authenticated.

### 6.2 Typical Impact
- Unauthorized state changes:
  - Changing passwords
  - Transferring funds
  - Modifying account details

### 6.3 Signals & Checks for the Agent

**CSRF Protection Tokens**
- Identify state-changing requests:
  - POST/PUT/PATCH/DELETE to endpoints like `/update`, `/transfer`, `/change-password`
- Check if:
  - A CSRF token parameter or header is present (e.g., `_csrf`, `X-CSRF-Token`)
  - The token is unique per session/request and not predictable

**SameSite Cookies**
- Check `SameSite` attribute for session cookies:
  - `SameSite=Lax` or `Strict` mitigates many CSRF attacks
- If `SameSite=None` is used:
  - Ensure `Secure` is also set

**Idempotent Methods**
- Flag forms that perform critical actions using `GET` or without tokens.

### 6.4 Agent Recommendations
- Require CSRF tokens for all state-changing requests.
- Enforce:
  - `SameSite` on cookies
  - Use of POST (or other non-GET verbs) for mutations
- Consider double-submit cookie or other defense-in-depth patterns.

---

## 7. Security Misconfiguration

### 7.1 What It Is
Insecure default settings, verbose error messages, left-over debug endpoints, or incomplete hardening.

### 7.2 Typical Impact
- Information leakage  
- Easier exploitation of other flaws  
- Direct compromise via exposed admin tools

### 7.3 Signals & Checks for the Agent

**Error Handling**
- Trigger errors with malformed input:
  - Overly long parameters
  - Invalid types like letters where numbers are expected
- Look for:
  - Stack traces
  - Framework names and versions
  - Database error messages

**Default/Debug Endpoints**
- Probe for common paths:
  - `/phpinfo.php`, `/server-status`, `/admin/`, `/debug`, `/console`
- If found and accessible, flag.

**Headers & Options**
- Check security-related headers:
  - Missing or weak:
    - `X-Frame-Options` / `Content-Security-Policy` for clickjacking
    - `X-Content-Type-Options`
    - `Referrer-Policy`
- Look for:
  - Directory listings on `/uploads/`, `/logs/`, `/backup/`

### 7.4 Agent Recommendations
- Disable verbose error pages in production.
- Remove or protect debug/admin interfaces with strong auth.
- Configure recommended security headers consistently.

---

## 8. Using Components with Known Vulnerabilities

### 8.1 What It Is
Reliance on outdated libraries/frameworks/platforms with publicly known security issues.

### 8.2 Typical Impact
- Exploitation via known CVEs  
- Full compromise using public exploit code

### 8.3 Signals & Checks for the Agent

**Technology Fingerprinting**
- Inspect:
  - HTTP headers (`Server`, `X-Powered-By`, `X-AspNet-Version`)
  - HTML comments
  - JavaScript and CSS file names with versions (e.g., `jquery-1.8.3.js`)
- Infer:
  - Framework versions (WordPress, Drupal, Joomla, Django, Rails, etc.)
  - Library versions (jQuery, AngularJS, React, Vue, Bootstrap)

**Version Evaluation (Conceptual)**
- Maintain an internal knowledge base (outside this file) mapping:
  - Component name + version → known severe vulnerabilities
- Flag:
  - End-of-life versions
  - Versions with critical remote exploitation issues

### 8.4 Agent Recommendations
- Keep an up-to-date inventory of components and versions.
- Apply security patches promptly.
- Replace unsupported or EOL components.

---

## 9. Insufficient Logging & Monitoring

### 9.1 What It Is
Failure to detect, log, or alert on suspicious activities.

### 9.2 Typical Impact
- Undetected breaches  
- Forensics difficulties

### 9.3 Signals & Checks for the Agent

**Indirect Detection (Black-box)**
- Observe how the app responds to:
  - Multiple failed logins
  - Suspicious parameter values
- If:
  - There are no visible responses like CAPTCHAs, account lockouts, or alerts, note potential lack of monitoring.
- The agent can:
  - Suggest but not fully verify logging practices (requires internal access).

### 9.4 Agent Recommendations
- Log:
  - Login success/failure, privilege changes, critical operations
- Use centralized logging and alerting
- Regularly review logs and test incident response

---

## 10. Insecure Direct File & Path Handling

### 10.1 What It Is
File upload and path handling vulnerabilities, including path traversal and arbitrary file upload.

### 10.2 Typical Impact
- Defacement or malware hosting  
- Remote code execution  
- Disclosure of sensitive files

### 10.3 Signals & Checks for the Agent

**Path Traversal**
- Detect endpoints that accept file paths or names:
  - `?file=report.pdf`, `/download?name=backup.zip`
- Try payloads:
  - `../../../../etc/passwd`
  - `..\..\..\..\windows\win.ini`
- Compare:
  - HTTP status codes and error messages
  - If content suggests internal files, flag.

**File Upload**
- Identify upload points:
  - Forms with `<input type="file">`
- Try uploading:
  - Disallowed or renamed extensions (e.g., `.php.txt`, `.jsp.jpg`)
  - Oversized files
- If:
  - The app doesn’t validate content type or extension and allows uploading to web-accessible directories, flag.

### 10.4 Agent Recommendations
- Use allow-lists for file types and enforce size limits.
- Store uploads outside the web root.
- Normalize and validate all file paths server-side.

---

## 11. Agent Architecture & Workflow (High-Level)

To implement an automated agent around this checklist:

### 11.1 Core Capabilities
- HTTP client supporting:
  - GET/POST/PUT/PATCH/DELETE
  - Cookie and session handling
  - TLS configuration and header inspection
- Input/endpoint discovery:
  - Parse HTML, JSON, JS for links, forms, and API endpoints
- Response analysis:
  - Body diffing
  - Status code and header inspection
  - Timing measurements

### 11.2 Typical Workflow

1. **Reconnaissance**
   - Crawl target domain within defined scope.
   - Collect:
     - Endpoints
     - Parameters per endpoint
     - Forms, cookies, and headers.

2. **Classification**
   - For each endpoint:
     - Classify as:
       - Public/Unauthenticated
       - Authenticated (if given credentials)
       - Possibly sensitive (login, profile, payments, admin)

3. **Testing Phase**
   - For each vulnerability class above:
     - Apply only relevant tests (e.g., no destructive payloads without permission).
     - Record:
       - Request details
       - Response snapshots
       - Timing metrics

4. **Correlation & Deduplication**
   - Group findings by:
     - Endpoint + parameter
     - Type of vulnerability
   - Assign:
     - Confidence levels (low/medium/high)
     - Impact levels (low/medium/high/critical)

5. **Reporting**
   - Output:
     - Brief description of each finding
     - Evidence (request/response excerpts, timings)
     - Clear remediation guidance (as described per section above)

---

## 12. Output Format for Findings (Template)

The agent should produce structured findings, e.g.:

```yaml
- id: VULN-001
  type: "SQL Injection"
  endpoint: "POST /api/orders"
  parameter: "order_id"
  severity: "High"
  confidence: "High"
  evidence:
    request_sample: "order_id=1' AND SLEEP(5)--"
    baseline_response_time_ms: 200
    injection_response_time_ms: 5200
    error_indicators: []
  recommendation: |
    Use parameterized queries for all database access involving `order_id`.
    Validate that `order_id` is strictly numeric and within expected ranges.