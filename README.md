
WEB APPLICATION SECURITY ASSESSMENT — README
OWASP Top 10 (2021) Review — OWASP Juice Shop
==============================================================================


  Prepared by     | Ede Peter Bethran
  NextGen Cohort  | FE/24/6932350572
  Assessment date | 21 July 2026
  Environment     | Isolated VirtualBox lab (Kali Linux, host-only network)
  Classification  | Confidential — lab exercise


SCOPE & RULES
-------------

This assessment was conducted entirely within an isolated, self-hosted VirtualBox lab environment against OWASP Juice Shop, a deliberately vulnerable demo application maintained by the OWASP Foundation for training purposes. No production systems, third-party infrastructure, or live user data were accessed or tested at any point.


  Item              | Detail
  Target            | OWASP Juice Shop, running as a Docker container (bkimminich/juice-shop) at http://localhost:3000
  Attacker host     | Kali Linux 2026.1 (VirtualBox VM)
  Network isolation | VirtualBox-managed VM network; testing confined to localhost within the VM
  Testing window    | 21 July 2026
  Authorisation     | Self-authorised lab exercise
  Out of scope      | Any system other than the named Juice Shop container


EXECUTIVE SUMMARY
-----------------

This report presents the findings of a security assessment of OWASP Juice Shop, performed against the OWASP Top 10 (2021) standard. The assessment was carried out in an isolated lab environment as part of a Nextgen cybersecurity training and project addressing a common problem in Nigeria's web application ecosystem: applications repeatedly shipping known, well-documented classes of vulnerability.


Key Findings
~~~~~~~~~~~~

Testing identified 7 confirmed vulnerabilities across 6 OWASP Top 10 categories, including a Critical SQL injection flaw enabling full authentication bypass as the administrator account, a Broken Access Control flaw (IDOR) exposing other users' basket data, and a publicly exposed directory leaking a password database file and internal business documents.

<img width="975" height="565" alt="image" src="https://github.com/user-attachments/assets/bc90bb44-46ad-4257-b7b3-70a842ad3f7a" />


Overall Risk Rating
~~~~~~~~~~~~~~~~~~~

Critical — the combination of an unauthenticated SQL injection admin bypass (F-01) and an exposed credential store (F-07) means the application's most sensitive assets are trivially reachable by an unauthenticated attacker.


Findings at a Glance
~~~~~~~~~~~~~~~~~~~~


  OWASP Category                           | Findings       | Highest Severity
  A01 - Broken Access Control              | 1 (F-03)       | High
  A02 - Cryptographic Failures             | 1 (F-06)       | High
  A03 - Injection                          | 2 (F-01, F-02) | Critical
  A04 - Insecure Design                    | 0 — not tested | —
  A05 - Security Misconfiguration          | 1 (F-04)       | Medium
  A06 - Vulnerable & Outdated Components   | 1 (F-07)       | High
  A07 - Identification & Authentication Failures | 1 (F-05)       | High
  A08 - Software & Data Integrity Failures | 0 — not tested | —
  A09 - Security Logging & Monitoring Failures | 0 — not tested | —
  A10 - Server-Side Request Forgery (SSRF) | 0 — not tested | —


METHODOLOGY
-----------

Testing followed a structured pass through the OWASP Top 10 (2021) categories, combining automated checks with manual verification to avoid false positives:

  - Reconnaissance:  HTTP header and response inspection via curl

  - Manual exploitation: direct testing of injection points, access control boundaries, and authentication flows

  - API-level testing: browser DevTools and curl used to interact with REST endpoints directly, bypassing frontend-only validation

  - Manual verification: every finding below was independently reproduced before being recorded, to confirm exploitability and eliminate false positives

Each confirmed finding was logged with category, description, reproduction steps, evidence, severity, business impact, and a recommended fix.


Severity Rating Scale
~~~~~~~~~~~~~~~~~~~~~


  Rating   | Definition
  Critical | Directly exploitable, leads to full compromise (e.g. auth bypass, RCE)
  High     | Significant impact, may require some conditions to exploit
  Medium   | Limited impact or requires user interaction / specific conditions
  Low      | Minor issue, defence-in-depth or best-practice gap


DETAILED FINDINGS
-----------------


Finding 1: SQL Injection Login Bypass (F-01)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A03 - Injection
  Severity           | Critical
  Affected Component | /#/login authentication form

Description

The login form's email field does not sanitize input, allowing a classic ' OR 1=1-- payload to bypass authentication entirely and log in as the administrator account without valid credentials.

Steps to Reproduce

1) Navigate to /#/login 2) Enter ' OR 1=1-- in the email field, any value in the password field 3) Submit 4) Successfully authenticated as the administrator (confirmed by the application's own "Login Admin" challenge banner)

Evidence

Business Impact

Full account takeover; an attacker can log in as any user, including the administrator, without any valid credentials.

Recommended Fix

Use parameterized queries / prepared statements for all database access; never concatenate user input into SQL strings. Add input validation as defence-in-depth.


Finding 2: DOM-Based Cross-Site Scripting (XSS) (F-02)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A03 - Injection
  Severity           | High
  Affected Component | Product search field

Description

The search feature reflects user input directly into the DOM without sanitization, allowing execution of arbitrary JavaScript via an iframe javascript: URI payload.

Steps to Reproduce

1) Click the search icon 2) Enter <iframe src="javascript:alert('xss')">  3) Press Enter 4) JavaScript alert executes in the browser

Evidence

Business Impact

Attacker-controlled JavaScript execution in victims' browsers; could enable session hijacking, credential theft, or defacement.

Recommended Fix

Sanitize/escape all user input before rendering; avoid innerHTML for untrusted content, use textContent or a sanitization library (e.g. DOMPurify); implement a strict Content-Security-Policy.


Finding 3: Broken Access Control — IDOR on Basket API (F-03)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A01 - Broken Access Control
  Severity           | High
  Affected Component | /rest/basket/{id} API endpoint

Description

The /rest/basket/{id} endpoint does not verify that the authenticated user owns the requested basket ID. Any valid token can retrieve any other user's basket simply by changing the ID.

Steps to Reproduce

1) Authenticate and obtain a valid JWT  2) Send GET /rest/basket/2 (or any ID) with your own token in the Authorization header 3) Server returns another user's basket data (confirmed: UserId 2, not the tester's own account)

Evidence

Business Impact

Any authenticated user can enumerate and view other users' basket/order contents, a privacy and data-exposure issue that could extend to modifying others' baskets.

Recommended Fix

Enforce object-level authorization: verify basket. UserId matches the authenticated user server-side before returning data, on every request.


Finding 4: Permissive CORS Policy & Missing Security Headers (F-04)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A05 - Security Misconfiguration
  Severity           | Medium
  Affected Component | Global HTTP response headers

Description

The server responds with Access-Control-Allow-Origin: *, permitting any origin to make cross-domain requests to the API. No Content-Security-Policy or Strict-Transport-Security headers are set, and malformed requests return verbose stack traces revealing internal file paths.

Steps to Reproduce

1) Run curl -I http://localhost:3000  2) Observe Access-Control-Allow-Origin: * 3) Observe no CSP/HSTS headers present 4) Send a malformed request path and observe a full Express stack trace returned in the response

Evidence

Business Impact

Wildcard CORS combined with the IDOR (F-03) means a malicious third-party site could exfiltrate other users' data via a victim's authenticated browser session; missing CSP leaves the XSS (F-02) with no mitigating control; stack traces aid attacker reconnaissance.

Recommended Fix

Restrict Access-Control-Allow-Origin to explicitly trusted origins; add a strict Content-Security-Policy; enforce HSTS once served over HTTPS; disable verbose error/stack trace output in production.


Finding 5: No Brute-Force Protection on Login (F-05)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A07 - Identification & Authentication Failures
  Severity           | High
  Affected Component | /rest/user/login API endpoint

Description

The /rest/user/login endpoint does not implement rate limiting, account lockout, or CAPTCHA after repeated failed login attempts.

Steps to Reproduce

Sent 10 consecutive POST requests to /rest/user/login using a valid email and an incorrect password; all 10 responses returned identical 401 status codes with no increasing delay, lockout, or challenge.

Evidence

Business Impact

Enables unrestricted brute-force or credential-stuffing attacks against user accounts, including the admin account already compromised via SQL injection (F-01). An attacker without that shortcut could still eventually guess weak passwords unimpeded.

Recommended Fix

Implement account lockout after N failed attempts (e.g. 5), add exponential backoff / rate limiting per IP and per account, and introduce CAPTCHA after repeated failures.


  OWASP Category     | A02 - Cryptographic Failures
  Severity           | High
  Affected Component | Entire application (served over HTTP)

Evidence

Business Impact

Credentials and session tokens sent over the network can be intercepted via basic traffic sniffing (e.g. on a shared/public network), leading to account takeover.

Recommended Fix

Serve the application exclusively over HTTPS/TLS; add a Strict-Transport-Security header with a long max-age; redirect all HTTP traffic to HTTPS.


Finding 7: Exposed Sensitive Files via Public Directory Listing (F-07)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


  OWASP Category     | A06 - Vulnerable & Outdated Components
  Severity           | High
  Affected Component | /ftp/ directory

Description

The /ftp/ directory allows unauthenticated directory listing and file access, exposing backup files, internal business documents, and a KeePass password database file (incident-support.kdbx). Exposed package.json.bak files reveal exact dependency versions, usable to cross-reference known CVEs.

Steps to Reproduce

Ran curl -s http://localhost:3000/ftp/ | grep -oP '(?<=href=")[^"]+' — returned a full listing of exposed files; any file is then directly downloadable via /ftp/<filename>.

Evidence

Business Impact

Exposure of a password database file and internal business documents (acquisitions, legal, coupon data) represents a serious confidentiality breach; version disclosure via backup files accelerates targeted exploitation of known component vulnerabilities.

Recommended Fix

Disable directory listing on the web server; remove backup/legacy files from publicly served directories entirely; enforce authentication on any internal-facing static assets; add dependency scanning (e.g. npm audit, Dependabot) to catch outdated packages proactively.


REMEDIATION ROADMAP
-------------------

Recommended remediation order, prioritised by severity and real-world exploit chaining (several findings compound one another):


  Priority | Finding    | Recommended Action
  1        | F-01       | Fix SQL injection — adopt parameterized queries across all DB access
  2        | F-07       | Remove exposed /ftp/ directory and all backup/sensitive files immediately
  3        | F-03       | Add object-level authorization checks to all /rest/basket/{id} and similar endpoints
  4        | F-02       | Sanitize search input; add Content-Security-Policy
  5        | F-05, F-06 | Add rate limiting/lockout on auth endpoints; move to HTTPS with HSTS
  6        | F-04       | Restrict CORS to trusted origins; disable verbose error output in production


General Hardening Recommendations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  - Enforce HTTPS everywhere and set Secure/HttpOnly/SameSite flags on cookies

  - Adopt parameterised queries / ORM usage as the only accepted pattern for DB access

  - Apply a Content Security Policy (CSP) to mitigate XSS impact

  - Introduce dependency scanning (e.g. npm audit, Dependabot) into the build pipeline

  - Add centralised logging and alerting for authentication failures and access-control violations

  - Establish a patch/update cadence for third-party libraries and frameworks

  - Remove all backup and legacy files from any publicly served directory


APPENDIX A: LAB ENVIRONMENT EVIDENCE
------------------------------------

Screenshots confirming the assessment was performed against the target application running within the isolated VirtualBox lab (Kali Linux attacker VM, Docker-hosted target).

OWASP Juice Shop landing page, confirmed running at http://localhost:3000 inside the Kali VM.

Docker service confirmed active and running within the Kali VM.

docker ps output confirming the Juice Shop container running with port 3000 mapped.


APPENDIX B: TOOLING
-------------------


  Tool              | Purpose
  VirtualBox        | Isolated virtual lab environment
  Kali Linux 2026.1 | Attacker/testing platform
  Docker            | Hosting the OWASP Juice Shop target application
  curl              | HTTP header inspection and direct API testing
  Firefox DevTools  | Console-based API testing with authenticated requests

==============================================================================
End of document. Converted from Web_Application_Security_Assessment.docx
==============================================================================
