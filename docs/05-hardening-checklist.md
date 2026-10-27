# 05. Prioritized Security Hardening Checklist: SecureShop

## Document Control

| Attribute | Details |
| :--- | :--- |
| **Document** | Prioritized Technical Hardening Specification & Checklist |
| **System** | SecureShop E-Commerce Web Application |
| **Methodology** | Defense-in-Depth, OWASP ASVS v4.0.3, NIST SP 800-53 Rev. 5 |
| **Total Controls** | 22 Structured Security Controls across 3 Priority Tiers |
| **Baseline Status** | **Proposed / Not Implemented** (Design-Level Specification) |

---

## 1. Hardening Architecture & Prioritization Strategy

A threat model is only as effective as the engineering safeguards it inspires. This **Security Hardening Checklist** translates the 15 risks identified in `docs/04-risk-register.md` into concrete, actionable engineering requirements.

Security controls are organized into **three strict priority tiers**:
- **Priority 1 — Critical Security Protections (Controls 01 – 08)**: Core defense mechanisms that neutralize existential threats: authentication, server-side authorization, anti-IDOR, secure session handling, secrets protection, database security, TLS transport, and webhook verification.
- **Priority 2 — Important Protections (Controls 09 – 16)**: Essential application defenses that prevent common web vulnerabilities: input validation, parameterized queries, CSRF defenses, HTTP security headers, rate limiting, error handling, audit logging, and dependency updates.
- **Priority 3 — Additional Hardening (Controls 17 – 22)**: Operational, continuous monitoring, and infrastructure governance controls: SIEM alerting, encrypted backups, CIS benchmark reviews, CI/CD SAST scanning, periodic access reviews, and penetration testing.

> [!IMPORTANT]
> **Implementation Status Declaration:**
> Because SecureShop is an educational architectural design model, **all controls in this checklist are designated as `Proposed` or `Not implemented`.** No controls are fabricated as "Implemented and verified" without empirical test evidence from a live production environment.

---

## 2. Priority 1 — Critical Security Protections

---

### Control SEC-CTRL-01: Strong Authentication & Adaptive Password Hashing
- **Control ID**: `SEC-CTRL-01`
- **Security Recommendation**: Enforce robust password hashing using modern memory-hard key derivation algorithms; mandate Multi-Factor Authentication (MFA) for administrative staff; screen new credentials against known breach dictionaries.
- **Threats Addressed**: `THREAT-S-01` (Credential Stuffing), `THREAT-E-02` (Admin Compromise) $\to$ `RISK-01`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Hash all passwords using **Argon2id** (configured with memory cost $m=65536$, time cost $t=3$, parallelism $p=4$) or **bcrypt** (work factor $\ge 12$) with a unique cryptographic salt per user.
  - Require passwords to have a minimum length of 12 characters with no arbitrary character composition rules (per NIST SP 800-63B).
  - Integrate an automated breach check (e.g., HaveIBeenPwned API using k-Anonymity SHA-1 model) to reject compromised passwords during registration and resets.
  - Enforce mandatory Time-based One-Time Passwords (TOTP via RFC 6238) for all users with `role: admin`.
- **Verification Method**: Submit a test registration with password `Password123!` and verify immediate rejection due to breached list; verify in database that stored hash begins with `$argon2id$` or `$2b$12$`; verify login to `/admin` prompts for TOTP token.
- **Status**: `Proposed`

---

### Control SEC-CTRL-02: Server-Side Role-Based Access Control (RBAC) & Default-Deny
- **Control ID**: `SEC-CTRL-02`
- **Security Recommendation**: Implement centralized, server-side authorization middleware enforcing strict Role-Based Access Control (RBAC) on all administrative routes, following a default-deny security policy.
- **Threats Addressed**: `THREAT-E-01` (Broken Access Control) $\to$ `RISK-14`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Implement a dedicated authentication and authorization middleware (e.g., `requireRole('admin')`) executed before any administrative route handler.
  - Establish a default-deny architecture: all new routes require explicit permission declaration; unauthenticated or under-privileged requests automatically receive HTTP 403 Forbidden.
  - Never rely on frontend UI hiding (CSS `display: none` or conditional React/Vue rendering) to restrict administrative functions.
- **Verification Method**: Authenticate as a standard customer user (`role: customer`); issue an HTTP POST request via `curl` to `https://secureshop.example/api/admin/products`; verify the server terminates execution and returns `403 Forbidden` with an audit event emitted to SIEM.
- **Status**: `Proposed`

---

### Control SEC-CTRL-03: Protection Against Unauthorized Data Access (Anti-IDOR)
- **Control ID**: `SEC-CTRL-03`
- **Security Recommendation**: Enforce server-side object ownership validation on every data access endpoint, and replace sequential integer primary keys with non-sequential UUIDs.
- **Threats Addressed**: `THREAT-I-01` (Insecure Direct Object Reference) $\to$ `RISK-09`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - On every request requesting user-specific records (e.g., `GET /api/orders/:id`), the backend SQL query must explicitly bind the authenticated session user ID:
    ```sql
    SELECT * FROM orders WHERE id = :order_id AND user_id = :session_user_id;
    ```
  - If no record matches both conditions, return `404 Not Found` (to prevent attackers from discovering valid order IDs).
  - Migrate all public-facing object identifiers from auto-incrementing integers (`1, 2, 3...`) to cryptographically random UUIDv4 identifiers (`f47ac10b-58cc-4372-a567-0e02b2c3d479`).
- **Verification Method**: Using two distinct test accounts (User A and User B), obtain an order created by User B; attempt to retrieve it using User A's active session cookie; verify the response is `404 Not Found` or `403 Forbidden`.
- **Status**: `Proposed`

---

### Control SEC-CTRL-04: Secure Session Handling & Cookie Security Attributes
- **Control ID**: `SEC-CTRL-04`
- **Security Recommendation**: Issue high-entropy session tokens stored in strictly hardened HTTP cookies with automated rotation and idle timeouts.
- **Threats Addressed**: `THREAT-S-02` (Session Hijacking), `THREAT-I-03` (Cookie Theft) $\to$ `RISK-02`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Generate session tokens using a cryptographically secure pseudorandom number generator (CSPRNG) yielding at least 128 bits of entropy (e.g., `crypto.randomBytes(32)`).
  - Configure session cookies with mandatory security flags:
    ```http
    Set-Cookie: session_id=...; Secure; HttpOnly; SameSite=Strict; Path=/; Max-Age=1800
    ```
  - Enforce an idle session timeout of 30 minutes and an absolute session lifespan of 12 hours.
  - Automatically regenerate session IDs upon login, privilege change, and logout to neutralize session fixation attacks.
- **Verification Method**: Inspect HTTP response headers on `/api/auth/login` using `curl -I`; verify `HttpOnly`, `Secure`, and `SameSite=Strict` flags are present; verify session token value changes immediately before and after login.
- **Status**: `Proposed`

---

### Control SEC-CTRL-05: Protection of Cryptographic Secrets & Environment Separation
- **Control ID**: `SEC-CTRL-05`
- **Security Recommendation**: Isolate all cryptographic keys, database passwords, and third-party API tokens within dedicated secrets managers; eliminate hardcoded secrets in source code.
- **Threats Addressed**: `THREAT-E-02` (JWT Key Compromise), `THREAT-S-03` (Webhook Secret Leak) $\to$ `RISK-15`, `RISK-03`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Store payment gateway API keys, webhook signing secrets, database credentials, and JWT signing keys in a managed secrets vault (e.g., HashiCorp Vault, AWS Secrets Manager).
  - Inject secrets into application runtimes via ephemeral environment variables at container startup; never commit `.env` files to Git.
  - Integrate pre-commit hooks and CI/CD secret scanning (e.g., `git-secrets`, `trufflehog`) to block accidental repository commits.
  - Rotate cryptographic signing keys on a scheduled 90-day cycle.
- **Verification Method**: Execute automated repository scan with `trufflehog --regex --entropy=False .`; verify zero secrets detected; verify that environment variables are populated from vault during build.
- **Status**: `Proposed`

---

### Control SEC-CTRL-06: Secure Database Access & Network Segmentation
- **Control ID**: `SEC-CTRL-06`
- **Security Recommendation**: Confine the Primary Database to private, non-routable subnets, and configure dedicated database service accounts operating under least privilege.
- **Threats Addressed**: `THREAT-T-02` (SQLi), `THREAT-I-01` (Mass Data Leakage) $\to$ `RISK-05`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Host database instances within an isolated private VPC subnet with no public IP address, reachable only from application servers via security group rules on port 5432/3306.
  - Enforce TLS encryption for all database connections in transit (`sslmode=verify-full`).
  - Create separate database users for each service:
    - Application user: Granted `SELECT`, `INSERT`, `UPDATE` on specific operational tables.
    - Explicitly deny `DROP`, `ALTER`, `TRUNCATE`, and administrative superuser privileges to the web application's database account.
- **Verification Method**: Attempt to connect to database host from an external IP address; verify connection times out; execute `DROP TABLE products;` using the web application's database credentials; verify database server returns permission denied.
- **Status**: `Proposed`

---

### Control SEC-CTRL-07: Transport Layer Security (TLS) & Transport Hardening
- **Control ID**: `SEC-CTRL-07`
- **Security Recommendation**: Enforce modern TLS 1.3 transport encryption across all external and administrative communication channels with HTTP Strict Transport Security (HSTS).
- **Threats Addressed**: `THREAT-I-03` (Eavesdropping & SSL Stripping) $\to$ `RISK-11`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Disable legacy TLS 1.0, 1.1, and 1.2; mandate **TLS 1.3** (with fallback to TLS 1.2 using secure ECDHE cipher suites only).
  - Configure the reverse proxy to permanently redirect all inbound plaintext HTTP port 80 traffic to HTTPS port 443 via HTTP 301.
  - Send the HSTS header on all responses:
    ```http
    Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
    ```
- **Verification Method**: Run `curl -I http://secureshop.example`; verify 301 redirect to HTTPS; execute `openssl s_client -connect secureshop.example:443 -tls1_1` and verify handshake failure.
- **Status**: `Proposed`

---

### Control SEC-CTRL-08: Cryptographic Verification of External Payment Webhooks
- **Control ID**: `SEC-CTRL-08`
- **Security Recommendation**: Require cryptographic HMAC-SHA256 signature verification and strict timestamp freshness checks on all inbound payment gateway webhooks.
- **Threats Addressed**: `THREAT-S-03` (Fake Payment Webhook Spoofing) $\to$ `RISK-03`
- **Priority**: **Priority 1 (Critical)**
- **Implementation Guidance**:
  - Compute the HMAC-SHA256 digest of the raw webhook request body using the shared webhook secret:
    ```javascript
    const expectedSig = crypto.createHmac('sha256', WEBHOOK_SECRET).update(rawBody).digest('hex');
    ```
  - Compare signatures using a constant-time comparison algorithm (`crypto.timingSafeEqual`) to prevent timing side-channel attacks.
  - Reject any webhook requests with timestamps older than 300 seconds (5 minutes) to block replay attacks.
  - Implement an out-of-band verification query directly to the payment processor's API before updating order fulfillment status.
- **Verification Method**: Send a synthetic HTTP POST to `/api/webhooks/payment` with an invalid signature in the header; verify server rejects request with HTTP 401 Unauthorized; send identical valid webhook twice and verify idempotency prevents duplicate order updates.
- **Status**: `Proposed`

---

## 3. Priority 2 — Important Protections

---

### Control SEC-CTRL-09: Strict Input Validation & Allowlisting
- **Control ID**: `SEC-CTRL-09`
- **Security Recommendation**: Validate all incoming user inputs against strict, allowlist-based schemas on the server side prior to processing.
- **Threats Addressed**: `THREAT-T-01` (Cart Tampering), `THREAT-D-01` (Search DoS) $\to$ `RISK-04`, `RISK-12`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Implement schema validation libraries (e.g., Zod, Joi, or Pydantic) on all API endpoints.
  - Enforce type coercion, length boundaries, and character set allowlists (e.g., product search queries restricted to 3–50 alphanumeric characters; order quantities restricted to integers between 1 and 99).
  - Reject unexpected or extraneous JSON fields to prevent Mass Assignment vulnerabilities.
- **Verification Method**: Submit payload `{"qty": -5, "price": 0.01}` to checkout route; verify schema validator returns HTTP 400 Bad Request with field validation errors.
- **Status**: `Proposed`

---

### Control SEC-CTRL-10: Universal Parameterized Database Queries (Prepared Statements)
- **Control ID**: `SEC-CTRL-10`
- **Security Recommendation**: Eliminate all dynamic SQL string concatenation by mandating parameterized queries or ORM query builders across all data persistence layers.
- **Threats Addressed**: `THREAT-T-02` (SQL Injection) $\to$ `RISK-05`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Strictly prohibit raw string concatenation, string formatting, or template literals when assembling SQL statements.
  - Ensure all database queries utilize parameter placeholders (e.g., `?` or `$1`):
    ```javascript
    // SECURE IMPLEMENTATION
    const result = await db.query('SELECT * FROM products WHERE category = $1 AND search_tsv @@ plainto_tsquery($2)', [category, query]);
    ```
  - Enforce static analysis rules (ESLint `security/detect-object-injection`, SonarQube) in CI/CD pipelines to automatically fail builds containing unsanitized SQL strings.
- **Verification Method**: Audit database repository layer using automated SAST; execute test queries with SQL payloads (`' OR '1'='1`) and verify database treats payload strictly as literal data.
- **Status**: `Proposed`

---

### Control SEC-CTRL-11: Cross-Site Request Forgery (CSRF) Protection
- **Control ID**: `SEC-CTRL-11`
- **Security Recommendation**: Protect all state-changing HTTP endpoints against CSRF using the Synchronizer Token Pattern or strict SameSite cookie attributes.
- **Threats Addressed**: `THREAT-S-02` (Session Impersonation) $\to$ `RISK-02`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - For cookie-authenticated traditional forms, issue cryptographically unique, per-session anti-CSRF tokens embedded in hidden form inputs and validated on POST/PUT/DELETE handlers.
  - For single-page APIs, enforce custom request headers (e.g., `X-Requested-With` or `X-CSRF-Token`) combined with `SameSite=Strict` or `SameSite=Lax` cookie configurations.
- **Verification Method**: Submit a state-changing POST request from an external origin HTML page; verify request is blocked or rejected due to missing CSRF token.
- **Status**: `Proposed`

---

### Control SEC-CTRL-12: Comprehensive HTTP Security Headers
- **Control ID**: `SEC-CTRL-12`
- **Security Recommendation**: Deploy defense-in-depth HTTP security headers to instruct modern browsers to enforce strict content and framing restrictions.
- **Threats Addressed**: `THREAT-S-02` (XSS), `THREAT-I-02` (Banner Disclosure) $\to$ `RISK-02`, `RISK-10`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Configure the reverse proxy to emit the following headers on all responses:
    ```http
    Content-Security-Policy: default-src 'self'; script-src 'self' https://js.stripe.com; frame-src https://js.stripe.com; object-src 'none'; base-uri 'self';
    X-Frame-Options: DENY
    X-Content-Type-Options: nosniff
    Referrer-Policy: strict-origin-when-cross-origin
    Permissions-Policy: geolocation=(), camera=(), microphone=()
    ```
  - Strip software identification headers: remove `Server` and `X-Powered-By`.
- **Verification Method**: Run `curl -I https://secureshop.example`; inspect headers to confirm presence of `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, and verify absence of `X-Powered-By`.
- **Status**: `Proposed`

---

### Control SEC-CTRL-13: Multi-Tiered Rate Limiting & Abuse Prevention
- **Control ID**: `SEC-CTRL-13`
- **Security Recommendation**: Deploy granular IP- and account-based sliding window rate limits on all public and authentication endpoints.
- **Threats Addressed**: `THREAT-S-01` (Brute Force), `THREAT-D-01`, `THREAT-D-02` (DoS) $\to$ `RISK-01`, `RISK-12`, `RISK-13`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Implement rate limiting in the reverse proxy / cache layer (e.g., Redis sliding window):
    - `/api/auth/login`: Max 5 requests per minute per IP and per account.
    - `/api/auth/register`: Max 3 requests per hour per IP.
    - `/api/products/search`: Max 30 requests per minute per IP.
    - Global application routes: Max 120 requests per minute per IP.
  - Return HTTP 429 Too Many Requests with a `Retry-After` header when thresholds are exceeded.
- **Verification Method**: Execute an automated curl loop sending 10 rapid POST requests to `/api/auth/login`; verify request 6 returns `429 Too Many Requests` with a valid `Retry-After` header.
- **Status**: `Proposed`

---

### Control SEC-CTRL-14: Standardized Safe Error Handling & Information Hiding
- **Control ID**: `SEC-CTRL-14`
- **Security Recommendation**: Intercept all unhandled application exceptions to prevent internal stack traces, database schemas, or server paths from leaking to clients.
- **Threats Addressed**: `THREAT-I-02` (Verbose Errors) $\to$ `RISK-10`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Implement global error-handling middleware.
  - Catch all uncaught exceptions, generate a unique opaque error correlation ID (e.g., `ERR-9912`), log the full error and stack trace to **DS3** (SIEM), and return a sanitized JSON response:
    ```json
    {
      "success": false,
      "message": "An unexpected error occurred. Please contact support.",
      "error_id": "ERR-9912"
    }
    ```
  - Ensure development debug modes (e.g., `NODE_ENV=development` or `DEBUG=True`) are strictly disabled in staging and production configurations.
- **Verification Method**: Trigger an intentional error (e.g., invalid JSON body); verify response contains only generic message and correlation ID without file paths or SQL syntax.
- **Status**: `Proposed`

---

### Control SEC-CTRL-15: Centralized Immutable Security Audit Logging
- **Control ID**: `SEC-CTRL-15`
- **Security Recommendation**: Establish an append-only, tamper-resistant audit logging pipeline streaming security events to a centralized SIEM platform.
- **Threats Addressed**: `THREAT-R-01` (Repudiation), `THREAT-R-02` (Admin Actions) $\to$ `RISK-07`, `RISK-08`
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Log all high-value security events: successful logins, failed auth attempts, password resets, admin price changes, order status transitions, and permission denials.
  - Format log records in structured JSON containing: `timestamp` (ISO 8601 UTC), `event_type`, `user_id`, `source_ip`, `user_agent`, `status`, and `resource_id`.
  - Stream logs asynchronously to **DS3** via Syslog-over-TLS or HTTPS.
  - Restrict SIEM storage to write-only for application services; deny modification and deletion privileges to all application service accounts.
- **Verification Method**: Execute an admin product price modification; query the SIEM dashboard to confirm an event was logged containing the admin User ID, old price, new price, and timestamp.
- **Status**: `Proposed`

---

### Control SEC-CTRL-16: Automated Dependency Updates & Patch Management
- **Control ID**: `SEC-CTRL-16`
- **Security Recommendation**: Establish automated vulnerability scanning and automated pull request generation for all third-party software dependencies.
- **Threats Addressed**: Known CVE Exploitation, Supply Chain Compromise
- **Priority**: **Priority 2 (Important)**
- **Implementation Guidance**:
  - Integrate automated dependency scanning tools (e.g., GitHub Dependabot, Snyk, or Renovate) into the repository.
  - Configure daily scans of lockfiles (`package-lock.json`, `poetry.lock`, etc.).
  - Mandate that critical and high-severity vulnerability alerts require patch deployment within 72 hours.
- **Verification Method**: Trigger a manual Dependabot scan in GitHub; verify it accurately identifies known vulnerable library versions in test branch.
- **Status**: `Proposed`

---

## 4. Priority 3 — Additional Hardening

---

### Control SEC-CTRL-17: Real-Time Security Monitoring & SIEM Alerting
- **Control ID**: `SEC-CTRL-17`
- **Security Recommendation**: Configure real-time correlation rules and automated alerting in the SIEM platform to detect and notify on suspicious security anomalies.
- **Threats Addressed**: `THREAT-S-01`, `THREAT-D-02`, `THREAT-R-02` $\to$ `RISK-01`, `RISK-08`, `RISK-13`
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Establish alerting rules in the SIEM:
    - Rule 1: More than 10 failed login attempts from a single IP within 5 minutes triggers an automated Slack/PagerDuty notification.
    - Rule 2: Any product price reduction exceeding 20% by an admin generates an immediate management audit alert.
    - Rule 3: Spike in HTTP 500 errors exceeding 5% of total traffic triggers an on-call engineer page.
- **Verification Method**: Generate 12 rapid failed logins using a test script; verify SIEM fires the expected webhook alert to the security operations channel.
- **Status**: `Proposed`

---

### Control SEC-CTRL-18: Automated Backup, Encryption, and Disaster Recovery Drills
- **Control ID**: `SEC-CTRL-18`
- **Security Recommendation**: Implement automated daily encrypted database snapshots, off-site replication, and quarterly disaster recovery restore drills.
- **Threats Addressed**: Ransomware, Hardware Failure, Catastrophic Data Loss
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Automate nightly database snapshots stored in an encrypted off-site cloud storage bucket with immutability locks (Object Lock / WORM).
  - Encrypt all backup archives using AES-256 with KMS managed keys.
  - Establish a documented Recovery Time Objective (RTO $\le$ 2 hours) and Recovery Point Objective (RPO $\le$ 24 hours).
  - Conduct quarterly restore simulations into an isolated staging environment.
- **Verification Method**: Restore the most recent automated database backup into an isolated staging environment; execute data integrity checks to verify all customer orders are intact.
- **Status**: `Proposed`

---

### Control SEC-CTRL-19: Baseline Configuration Reviews & CIS Benchmark Hardening
- **Control ID**: `SEC-CTRL-19`
- **Security Recommendation**: Harden underlying operating systems, container hosts, web servers, and databases against Center for Internet Security (CIS) benchmarks.
- **Threats Addressed**: Insecure System Configurations, Unnecessary Attack Surface
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Audit Linux server configurations against the **CIS Ubuntu/Debian Benchmark**.
  - Disable unnecessary default OS services, remove compiler tools (`gcc`, `make`) from production containers, and ensure containers run as non-root users (`USER node` or `USER nobody`).
  - Restrict file system permissions: application directories set to read-only except for explicitly designated temporary upload directories mounted `noexec`.
- **Verification Method**: Run automated CIS compliance scanner (e.g., Lynis or Docker Bench for Security); verify zero critical findings.
- **Status**: `Proposed`

---

### Control SEC-CTRL-20: CI/CD Pipeline Static Application Security Testing (SAST)
- **Control ID**: `SEC-CTRL-20`
- **Security Recommendation**: Embed automated static security analysis and secret detection scanners into the CI/CD pipeline to block insecure code before merging.
- **Threats Addressed**: Insecure Coding Patterns, Hardcoded Secrets, SQLi Flaws
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Configure GitHub Actions to execute automated SAST tools (e.g., Semgrep, SonarQube, CodeQL) on every pull request.
  - Enforce rules that automatically block merging if high- or critical-severity security flaws or hardcoded API keys are discovered.
- **Verification Method**: Create a test pull request introducing an unsanitized SQL string; verify the CI pipeline fails and leaves an automated review comment blocking merge.
- **Status**: `Proposed`

---

### Control SEC-CTRL-21: Periodic Administrative Access Reviews
- **Control ID**: `SEC-CTRL-21`
- **Security Recommendation**: Establish a mandatory quarterly review process to audit and prune administrative access rights and revoke dormant accounts.
- **Threats Addressed**: `THREAT-E-01`, Insider Threats, Privilege Creep
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Automatically flag administrative accounts that have been dormant for more than 45 days.
  - Require engineering leadership to review and re-authorize active administrative permissions once every 90 days.
  - Automatically revoke access immediately upon employee offboarding via Single Sign-On (SSO) integration.
- **Verification Method**: Review quarterly access review audit sign-off sheet; verify dormant accounts are locked in the database.
- **Status**: `Proposed`

---

### Control SEC-CTRL-22: Routine Vulnerability Assessments & Annual Penetration Testing
- **Control ID**: `SEC-CTRL-22`
- **Security Recommendation**: Perform recurring automated dynamic application security scans (DAST) and engage independent third-party penetration testers annually.
- **Threats Addressed**: Undiscovered Zero-Day Vulnerabilities, Complex Logic Flaws
- **Priority**: **Priority 3 (Additional Hardening)**
- **Implementation Guidance**:
  - Schedule automated monthly DAST scans (e.g., OWASP ZAP or Burp Suite Enterprise) against pre-production staging environments.
  - Contract certified third-party ethical hackers (e.g., OSCP/CREST certified) to conduct an annual gray-box penetration test focusing on business logic flaws, payment workflows, and access controls.
  - Track all findings to remediation in Jira with strict SLA remediation windows.
- **Verification Method**: Inspect annual third-party penetration testing report and remediation verification re-test certificate.
- **Status**: `Proposed`

---

## 5. Beginner-Friendly Mentorship Section

### 5.1 Purpose of the Hardening Checklist
The **Security Hardening Checklist** is the engineering blueprint that instructs software developers and DevOps engineers on *exactly how to build the defenses* that eliminate the vulnerabilities uncovered during threat modeling.

### 5.2 Why It Matters
Telling developers "your application is vulnerable to SQL injection" is not enough. A professional cybersecurity engineer must provide concrete implementation guidance (e.g., "use parameterized queries with placeholders") and give Quality Assurance (QA) engineers a repeatable **Verification Method** so they can prove the control actually works.

### 5.3 How It Was Created
1. Gathered all 15 risks from `docs/04-risk-register.md`.
2. Grouped required defenses into three priority tiers based on risk scores (P1: Critical, P2: Important, P3: Additional).
3. Grounded the controls in established frameworks like **OWASP ASVS v4.0.3** and **NIST SP 800-53**.
4. Authored technical implementation code snippets and step-by-step verification commands.

### 5.4 How to Review This Document
- Confirm that every single P1 and P2 risk in the Risk Register maps to at least one control in this checklist.
- Check that every control contains a clear, practical verification method.
- Ensure all statuses are accurately marked as `Proposed` or `Not implemented` to maintain scientific integrity.

---

### 5.5 Cybersecurity Reviewer Questions & Suggested Answers

**Q1: Why are controls prioritized into three distinct tiers instead of implementing everything at once?**
> *Answer:* Real-world engineering teams have sprint constraints, release deadlines, and budget limits. Priority 1 controls (such as RBAC, parameterized queries, and webhook verification) neutralize catastrophic business-ending threats and *must* block launch. Priority 2 controls harden the application against standard automated attacks. Priority 3 controls establish mature, continuous operational security. Tiering ensures critical defenses are never compromised by scope fatigue.

**Q2: Why should all controls in this assessment be marked as `Proposed`?**
> *Answer:* SecureShop is a design-level assessment of a fictional web application architecture. Marking a control as "Implemented and verified" without running actual tests against a live server would be fraudulent and violate ethical engineering standards. Professional security documentation must always be honest about the empirical status of controls.

**Q3: How does Server-Side Authorization (SEC-CTRL-02) differ from Authentication (SEC-CTRL-01)?**
> *Answer:* **Authentication** answers the question: *"Who are you?"* (e.g., verifying a user's password and MFA code). **Authorization** answers the question: *"What are you allowed to do?"* (e.g., determining whether user ID 501 has permission to modify product prices). A user can be successfully authenticated as a customer, but they must be unauthorized to access administrative APIs.
