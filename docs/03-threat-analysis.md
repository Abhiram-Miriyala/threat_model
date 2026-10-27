# 03. STRIDE Threat Analysis: SecureShop E-Commerce Platform

## Document Control

| Attribute | Details |
| :--- | :--- |
| **Document** | STRIDE Threat Analysis Specification |
| **System** | SecureShop E-Commerce Web Application |
| **Framework** | Microsoft STRIDE (Threats per Element & Interaction) |
| **Threat Count** | 15 Granular Design-Level Threat Scenarios |
| **Evaluation Type** | Pre-Implementation Architectural Threat Identification |

---

## 1. Threat Modeling Methodology: The STRIDE Framework

The **STRIDE** methodology, developed by Praerit Garg and Loren Kohnfelder at Microsoft, is a structured, mnemonic-based threat classification model designed to discover security flaws during the system design phase.

STRIDE aligns with the fundamental security properties of computer systems:

| STRIDE Category | Threat Definition | Violated Security Property | SecureShop Focus Area |
| :--- | :--- | :--- | :--- |
| **S — Spoofing** | Pretending to be an entity (person, process, or system) other than one's own true identity. | **Authentication** | Customer login, Admin session, Payment Gateway webhook callbacks. |
| **T — Tampering** | Modifying data on disk, in memory, or in transit across a network without authorization. | **Integrity** | Cart prices, order state, database records, HTTP parameters. |
| **R — Repudiation** | Denying having performed an action or transaction without the system having adequate proof. | **Non-Repudiation** | Order placement, refunds, product price changes, audit log deletion. |
| **I — Information Disclosure** | Exposing sensitive data to individuals or systems that are not authorized to view it. | **Confidentiality** | Customer PII, order history, credit card tokens, debug stack traces. |
| **D — Denial of Service** | Depriving legitimate users of access to a service, process, or asset through resource exhaustion. | **Availability** | Web server threads, database connection pools, search endpoints. |
| **E — Elevation of Privilege** | Gaining unauthorized higher capabilities or access rights than normally permitted. | **Authorization** | Customer accessing `/admin`, role tampering in JWT, admin impersonation. |

> [!NOTE]
> **Design-Level Classification Notice:**
> Each threat analyzed in this document represents a **theoretical design-level threat** evaluated against the documented architecture of SecureShop. None of these scenarios describe vulnerabilities confirmed through penetration testing against live systems, as SecureShop is an educational design model.

---

## 2. Summary of Identified Threats

The following matrix maps each identified threat to its STRIDE category, target component, affected asset, and initial severity rating:

| Threat ID | STRIDE Category | Target Component | Threat Scenario Summary | Initial Severity |
| :--- | :--- | :--- | :--- | :--- |
| **THREAT-S-01** | Spoofing | P2: Auth Service | Credential stuffing & brute-force account takeover | **High** |
| **THREAT-S-02** | Spoofing | P1: Web App / Cookies | Session hijacking via stolen or predictable session identifiers | **High** |
| **THREAT-S-03** | Spoofing | P1: Web App / Webhooks | Fake payment confirmation webhook spoofing payment gateway | **Critical** |
| **THREAT-T-01** | Tampering | P3: Order Service | Client-side shopping cart price and quantity manipulation | **High** |
| **THREAT-T-02** | Tampering | DS1: Primary Database | SQL Injection via product catalog search and filtering | **Critical** |
| **THREAT-T-03** | Tampering | P3: Order Service | In-transit parameter tampering altering order fulfillment status | **Medium** |
| **THREAT-R-01** | Repudiation | P3: Order Service | Customer repudiates placing high-value order due to missing audit trail | **Medium** |
| **THREAT-R-02** | Repudiation | P3: Order Service | Disgruntled admin alters product prices without non-repudiation logging | **High** |
| **THREAT-I-01** | Information Disclosure | P3: Order Service | Insecure Direct Object Reference (IDOR) leaking customer order history | **High** |
| **THREAT-I-02** | Information Disclosure | P1: Web App Server | Verbose error stack traces disclosing database schema and server versions | **Medium** |
| **THREAT-I-03** | Information Disclosure | Edge Network (TB-01) | Eavesdropping on unencrypted HTTP traffic or missing HSTS | **High** |
| **THREAT-D-01** | Denial of Service | P3: Order Service | Unbounded wildcard database search queries causing CPU/RAM exhaustion | **Medium** |
| **THREAT-D-02** | Denial of Service | P1 & P2: Auth Service | Automated registration and login flood exhausting database connections | **High** |
| **THREAT-E-01** | Elevation of Privilege | P1 & P3: Web App | Broken access control permitting regular customers to call admin APIs | **Critical** |
| **THREAT-E-02** | Elevation of Privilege | P2: Auth Service | Tampering role claims in JWT or session object to gain superuser rights | **Critical** |

---

## 3. Granular STRIDE Threat Analyses

---

### Category 1: Spoofing (S)

#### Threat THREAT-S-01: Credential Stuffing & Automated Brute Force Account Takeover
- **STRIDE Category**: Spoofing (Violates Authentication)
- **Classification Reasoning**: An attacker impersonates a legitimate customer or administrator by guessing or reusing valid authentication credentials.
- **Affected Component / Asset**: `P2: Auth & Session Management Service` / `AST-02: Authentication Credentials`
- **Threat Description**: Threat actors acquire leaked credential dumps from third-party breaches on the dark web and launch automated credential stuffing attacks against SecureShop's `/api/auth/login` endpoint using botnets or proxy rotation to identify accounts reusing the same password.
- **Example Attack Scenario**:
  1. An attacker loads 100,000 username:password pairs into an automated tool like Hydra or OpenBullet.
  2. The tool rotates through residential proxies and submits login requests to SecureShop.
  3. If rate-limiting and account lockout mechanisms are absent, the tool identifies 250 valid customer accounts and 1 administrative account.
  4. The attacker logs into the compromised accounts and drains saved gift balances or accesses private order histories.
- **Potential Consequences**: 
  - *Technical*: Account takeover (ATO), unauthorized access to customer records, credential harvesting.
  - *Business*: Customer churn, severe reputational brand damage, legal liability under privacy regulations.
- **Recommended Mitigation**: Implement adaptive IP- and account-based rate limiting on login routes; enforce multi-factor authentication (MFA) for administrative accounts; integrate breached-password screening (e.g., HaveIBeenPwned API) during registration and password reset; apply CAPTCHA challenges after 3 consecutive failed attempts.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-01** (Customer $\to$ Web App) and **DF-07** (Web App $\to$ Auth Service).

---

#### Threat THREAT-S-02: Session Hijacking via Stolen or Predictable Session Identifiers
- **STRIDE Category**: Spoofing (Violates Authentication)
- **Classification Reasoning**: An attacker assumes the identity of an active user by intercepting or generating a session token that the server trusts as belonging to that user.
- **Affected Component / Asset**: `P1: Web Application Server` & `DS2: Session Store` / `AST-03: Active Session Identifiers`
- **Threat Description**: If session identifiers are generated using weak pseudorandom number generators, transmitted over insecure channels, or stored in cookies lacking the `HttpOnly`, `Secure`, and `SameSite` flags, an attacker can extract or predict the session cookie and impersonate the victim without needing their password.
- **Example Attack Scenario**:
  1. A victim customer accesses SecureShop on a public Wi-Fi network or encounters a Cross-Site Scripting (XSS) vulnerability in a product review comment.
  2. Because the session cookie lacks the `HttpOnly` flag, the malicious JavaScript snippet executes `document.cookie` and sends the session token to the attacker's server.
  3. The attacker injects the stolen cookie into their own browser and immediately accesses the victim's account, orders, and payment profile.
- **Potential Consequences**:
  - *Technical*: Complete session impersonation bypassing password and MFA checks.
  - *Business*: Unauthorized fraudulent purchases, privacy violations, immediate loss of customer trust.
- **Recommended Mitigation**: Issue cryptographically secure pseudorandom session identifiers (at least 128 bits of entropy); mandate cookie security flags: `HttpOnly` (blocks JavaScript access), `Secure` (ensures transmission over TLS only), and `SameSite=Strict` or `Lax` (prevents CSRF); enforce idle session timeouts (30 minutes) and absolute session expiration; regenerate session IDs upon login and privilege elevation.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-02** (Web App $\to$ Customer Browser) and stored in **DS2**.

---

#### Threat THREAT-S-03: Fake Payment Webhook Spoofing Payment Gateway
- **STRIDE Category**: Spoofing (Violates Authentication)
- **Classification Reasoning**: A malicious external actor pretends to be the authorized External Payment Gateway to convince SecureShop that an unpaid order has been settled.
- **Affected Component / Asset**: `P1: Web Application Server` & `P3: Order Service` / `AST-04: Payment Gateway Tokens` & `AST-06: Order Integrity`
- **Threat Description**: SecureShop exposes an HTTP webhook endpoint (`/api/webhooks/payment`) to receive notifications when a customer completes payment at the third-party gateway. If the application does not cryptographically authenticate the origin of this webhook, an attacker can directly submit synthetic `payment.completed` HTTP POST payloads.
- **Example Attack Scenario**:
  1. An attacker places an order for a \$2,500 laptop on SecureShop, generating Order ID `#9981` in `Pending Payment` status.
  2. Instead of completing payment at the payment gateway, the attacker crafts a POST request directly to `https://secureshop.example/api/webhooks/payment` containing `{"order_id": 9981, "status": "succeeded", "amount": 2500.00}`.
  3. Because the endpoint does not verify the payment processor's cryptographic signature, SecureShop accepts the payload and transitions the order status to `Paid - Ready for Shipping`.
  4. The warehouse fulfills and ships the item without SecureShop receiving any payment.
- **Potential Consequences**:
  - *Technical*: Circumvention of financial verification workflows, corrupted transactional state machine.
  - *Business*: Direct catastrophic financial inventory loss through systematic unpaid order fraud.
- **Recommended Mitigation**: Enforce mandatory HMAC-SHA256 signature verification on every incoming webhook payload using a pre-shared webhook secret key; verify timestamps in the webhook header to block replay attacks (reject messages older than 5 minutes); perform an out-of-band verification query (DF-12) to the payment gateway to confirm the transaction status before updating the order to `Paid`.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-03** (Third-Party SaaS Boundary) via data flow **DF-13** (Payment Gateway $\to$ Web App).

---

### Category 2: Tampering (T)

#### Threat THREAT-T-01: Client-Side Shopping Cart Price and Quantity Manipulation
- **STRIDE Category**: Tampering (Violates Integrity)
- **Classification Reasoning**: An attacker modifies financial transaction data transmitted from the client before the server processes the order.
- **Affected Component / Asset**: `P3: Product & Order Service` / `AST-06: Product Catalog & Pricing Integrity`
- **Threat Description**: If the web application relies on the browser to compute or transmit product unit prices in the checkout POST request, an attacker can intercept the HTTP traffic (using an intercepting proxy like OWASP ZAP or Burp Suite) and rewrite the prices to arbitrary lower values (e.g., \$0.01) or change quantities to negative numbers to offset totals.
- **Example Attack Scenario**:
  1. An attacker adds a \$1,200 camera to their cart.
  2. The browser generates a checkout request: `POST /api/orders/checkout` with JSON body: `{"items": [{"id": "CAM-01", "price": 1200.00, "qty": 1}]}`.
  3. The attacker intercepts the request in Burp Suite and modifies the body to: `{"items": [{"id": "CAM-01", "price": 1.00, "qty": 1}]}`.
  4. If the server trusts the submitted price and creates a payment intent for \$1.00, the attacker pays \$1.00 and receives the camera.
- **Potential Consequences**:
  - *Technical*: Manipulation of business data integrity, loss of database pricing consistency.
  - *Business*: Immediate revenue loss, inventory depletion, inability to reconcile financial ledgers.
- **Recommended Mitigation**: Never trust client-submitted pricing, discounts, or tax amounts; client submissions must only contain Product IDs and integer quantities; the Order Service (P3) must look up authoritative prices directly from the Primary Database (DS1) on the server side; validate that quantities are positive integers ($qty \ge 1$ and $qty \le 99$).
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-03** (Customer $\to$ Web App) and processed by **P3** via **DF-10** and **DF-11**.

---

#### Threat THREAT-T-02: SQL Injection via Product Catalog Search and Filtering
- **STRIDE Category**: Tampering (Violates Integrity; also enables Information Disclosure and Elevation of Privilege)
- **Classification Reasoning**: Direct injection of unauthorized database commands into application queries, allowing modification and destruction of data records.
- **Affected Component / Asset**: `DS1: Primary Relational Database` / `AST-01: PII`, `AST-02: Credentials`, `AST-06: Catalog`
- **Threat Description**: If user-supplied input from the product search bar or category query parameters (e.g., `?category=electronics&search=phone`) is concatenated directly into SQL statements rather than parameterized, an attacker can break the query syntax and execute arbitrary SQL commands.
- **Example Attack Scenario**:
  1. An attacker navigates to `https://secureshop.example/products?search=' OR 1=1; UPDATE products SET price=0.01 WHERE id=5;--`.
  2. The backend server executes the unsanitized string against the PostgreSQL database.
  3. The database updates the product table, tampering with retail prices, or executes `UNION SELECT` to dump password hashes and customer PII into the rendered search results.
- **Potential Consequences**:
  - *Technical*: Full database compromise, arbitrary data modification, extraction of all customer PII and password hashes, potential remote command execution on the database server.
  - *Business*: Catastrophic data breach, mandatory notification under GDPR/CCPA, massive regulatory fines, irreversible brand damage.
- **Recommended Mitigation**: Strictly mandate the use of parameterized queries (Prepared Statements) or an Object-Relational Mapper (ORM) with query binding across all database interactions; enforce a least-privilege database user account that prevents DDL commands (`DROP`, `ALTER`); implement strict input validation on search queries using an allowlist of alphanumeric characters.
- **Relevant Trust Boundary / Data Flow**: Originates across **TB-01** (DF-01) and reaches **DS1** across **TB-02** via data flow **DF-11**.

---

#### Threat THREAT-T-03: In-Transit Parameter Tampering Altering Order Fulfillment Status
- **STRIDE Category**: Tampering (Violates Integrity)
- **Classification Reasoning**: Altering the state variables of an order during update requests to bypass business workflow steps.
- **Affected Component / Asset**: `P3: Product & Order Processing Service` / `AST-05: Order & Transaction Records`
- **Threat Description**: When orders transition through lifecycle stages (`Pending` $\to$ `Paid` $\to$ `Shipped` $\to$ `Delivered`), endpoints accepting status updates without validating the current state machine rules can allow attackers to force invalid transitions (e.g., from `Cancelled` to `Refunded` twice, or from `Pending` directly to `Shipped`).
- **Example Attack Scenario**:
  1. An attacker places an order, cancels it, and receives a legitimate refund.
  2. The attacker replays a manipulated order update request: `PUT /api/orders/9981/status` with payload `{"status": "Refunded"}`.
  3. Due to missing transition validation, the system issues a second refund through the payment gateway API.
- **Potential Consequences**:
  - *Technical*: Corruption of transactional state logic, double-processing of financial actions.
  - *Business*: Direct monetary loss through double-refund fraud.
- **Recommended Mitigation**: Implement a strict finite state machine (FSM) in the Order Service that only permits valid state transitions (e.g., `Cancelled` can never transition to `Refunded` more than once); cryptographically sign or server-side validate all state update triggers.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** or **TB-04** via data flow **DF-05** or **DF-10**.

---

### Category 3: Repudiation (R)

#### Threat THREAT-R-01: Customer Repudiates Placing High-Value Order Due to Missing Transactional Audit Log
- **STRIDE Category**: Repudiation (Violates Non-Repudiation)
- **Classification Reasoning**: A user denies having placed a purchase order, and the organization cannot technically prove that the user authorized the transaction.
- **Affected Component / Asset**: `P3: Order Service` & `DS3: Security Logging & SIEM` / `AST-05: Order Records` & `AST-08: Audit Logs`
- **Threat Description**: If the checkout workflow does not log non-repudiation metadata (such as the authenticated User ID, timestamp, client IP address, User-Agent, and terms-of-service agreement confirmation), a customer can purchase high-value merchandise, receive delivery, and subsequently dispute the charge with their bank as unauthorized fraud.
- **Example Attack Scenario**:
  1. A customer logs into their account, orders a \$3,000 workstation, and has it delivered to a secondary address.
  2. After delivery, the customer calls their credit card company and files a "friendly fraud" chargeback claiming their account was hacked and they never placed the order.
  3. SecureShop's fraud team investigates but finds the application only stored `order_total` and `product_id`, with no log of the session ID, IP address, device fingerprint, or checkout timestamp.
  4. The bank rules in favor of the customer, and SecureShop loses both the merchandise and the \$3,000.
- **Potential Consequences**:
  - *Technical*: Inability to conduct forensic analysis or provide evidence in financial dispute proceedings.
  - *Business*: Unrecoverable chargeback fees, loss of merchandise, increased merchant processing penalty rates.
- **Recommended Mitigation**: Record comprehensive, immutable transactional audit logs for every order placement; capture authenticated User ID, timestamp (UTC), source IP address, session ID, cart contents, and payment intent reference; ship logs to **DS3** with append-only permissions.
- **Relevant Trust Boundary / Data Flow**: Traverses **TB-05** via data flow **DF-14c** (Order Service $\to$ Audit Store).

---

#### Threat THREAT-R-02: Malicious Administrator Alters Product Prices Without Immutable Audit Trail
- **STRIDE Category**: Repudiation (Violates Non-Repudiation)
- **Classification Reasoning**: An insider or compromised administrative account performs unauthorized actions, and the system cannot attribute the modification to a specific actor.
- **Affected Component / Asset**: `P3: Order Service` & `DS1: Database` / `AST-06: Catalog Integrity` & `AST-08: Audit Logs`
- **Threat Description**: If administrative operations (such as changing product prices, issuing refunds, or updating inventory counts) are executed without individual user attribution in the audit logs, a rogue employee or an attacker with compromised admin credentials can lower prices, purchase items, and deny any wrongdoing.
- **Example Attack Scenario**:
  1. An employee with admin access changes the retail price of an expensive television from \$1,500 to \$15.00.
  2. An accomplice purchases 10 units at \$15.00 each.
  3. The employee changes the price back to \$1,500.
  4. During an inventory audit, management notices the loss, but the database only shows the current price with no log of who executed the temporary price override.
- **Potential Consequences**:
  - *Technical*: Insider threats operate undetected; loss of forensic accountability.
  - *Business*: Severe inventory theft, collusion fraud, inability to satisfy regulatory audit requirements.
- **Recommended Mitigation**: Enforce individualized administrative accounts (no shared admin logins); record all administrative mutations in an append-only audit log capturing the administrator's User ID, timestamp, original value, modified value, and IP address; generate immediate alerts in the SIEM for price reductions exceeding 20%.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-04** (DF-05) and logged across **TB-05** (DF-14b, DF-14c).

---

### Category 4: Information Disclosure (I)

#### Threat THREAT-I-01: Insecure Direct Object Reference (IDOR) Leaking Customer PII & Order History
- **STRIDE Category**: Information Disclosure (Violates Confidentiality; caused by Broken Access Control)
- **Classification Reasoning**: Exposing private data to unauthorized users who query sequential or predictable database object identifiers.
- **Affected Component / Asset**: `P3: Order Service` & `DS1: Database` / `AST-01: Customer PII` & `AST-05: Order Records`
- **Threat Description**: When a customer views their receipt via `GET /api/orders/1045`, if the application backend does not verify that the logged-in user owns order `#1045`, an attacker can iterate through sequential numbers (`/api/orders/1046`, `/1047`, etc.) to view every other customer's full name, shipping address, phone number, and purchase details.
- **Example Attack Scenario**:
  1. Attacker registers an account (User ID `501`) and places an order, receiving Order ID `#2001`.
  2. The browser calls `GET /api/orders/2001`.
  3. The attacker writes a simple script iterating from `1` to `5000`: `curl -H "Cookie: session=attacker_token" https://secureshop.example/api/orders/{id}`.
  4. The server returns the order details for all 5,000 customers because the SQL query executes `SELECT * FROM orders WHERE id = :id` without checking `AND user_id = :current_user_id`.
- **Potential Consequences**:
  - *Technical*: Mass exfiltration of customer database records and PII.
  - *Business*: Catastrophic privacy breach, mandatory public disclosure under GDPR/CCPA, massive regulatory fines, class-action lawsuits.
- **Recommended Mitigation**: Implement strict server-side authorization checks on all resource queries: verify that `order.user_id === session.user_id` before returning data; replace sequential integer IDs with cryptographically secure, non-sequential UUIDv4 identifiers (e.g., `/api/orders/f47ac10b-58cc-4372-a567-0e02b2c3d479`); enforce automated unit and integration tests asserting that unauthorized users receive HTTP 403/404.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-02** / **DF-03** and retrieved from **DS1** across **TB-02** via **DF-11**.

---

#### Threat THREAT-I-02: Verbose Error Stack Traces Disclosing Database Schema & Internal Framework Versions
- **STRIDE Category**: Information Disclosure (Violates Confidentiality)
- **Classification Reasoning**: Accidental leakage of internal architectural and configuration secrets to external callers through unhandled system exceptions.
- **Affected Component / Asset**: `P1: Web Application Server` / `AST-08: System Configuration`
- **Threat Description**: When unexpected application errors occur (e.g., malformed JSON payloads, database timeout, or unhandled null pointers), production servers running in debug mode return full stack traces, database connection strings, raw SQL queries, or internal file paths to the user's browser.
- **Example Attack Scenario**:
  1. An attacker sends a malformed query parameter: `GET /api/products?id[]=foo`.
  2. The application framework crashes with an unhandled exception.
  3. The HTTP response displays: `PostgreSQL syntax error in line 44 of /var/www/secureshop/services/order.js: SELECT * FROM products WHERE id = Array. Database host: 10.0.2.15, User: db_user`.
  4. The attacker uses the disclosed framework version (Express 4.17.1), database engine (PostgreSQL 14.2), and internal IP structure to craft targeted zero-day exploits.
- **Potential Consequences**:
  - *Technical*: Reconnaissance data leakage, simplifying exploitation of internal vulnerabilities.
  - *Business*: Lowered barrier of entry for attackers; failure to meet compliance standards (OWASP ASVS).
- **Recommended Mitigation**: Disable debug mode and detailed error reporting in production environments; implement a centralized error-handling middleware that logs full stack traces internally to **DS3** while returning a generic, safe error message to the client (e.g., `{"error": "An internal error occurred. Reference ID: ERR-8841"}`); remove `X-Powered-By` and `Server` HTTP response headers.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-02** (Web App $\to$ Customer Browser).

---

#### Threat THREAT-I-03: Eavesdropping on Unencrypted HTTP Traffic or Missing HSTS
- **STRIDE Category**: Information Disclosure (Violates Confidentiality)
- **Classification Reasoning**: Interception of data in transit across public network channels due to lack of transport layer encryption.
- **Affected Component / Asset**: Edge Network (TB-01) / `AST-01: PII`, `AST-02: Credentials`, `AST-03: Session Cookies`
- **Threat Description**: If SecureShop allows plaintext HTTP connections, fails to automatically redirect HTTP to HTTPS, or omits HTTP Strict Transport Security (HSTS), an attacker positioned on the same local network (e.g., coffee shop public Wi-Fi) can perform an SSL stripping attack (using tools like `sslstrip`) to intercept credentials and session cookies in cleartext.
- **Example Attack Scenario**:
  1. A customer connects to a hotel Wi-Fi network and types `secureshop.example` into their browser.
  2. An attacker running ARP poisoning on the network intercepts the initial unencrypted HTTP request.
  3. The attacker proxies the connection to SecureShop over HTTPS but serves plaintext HTTP back to the customer.
  4. The customer logs in, and their username, password, and session cookie are captured in cleartext by the attacker.
- **Potential Consequences**:
  - *Technical*: Man-in-the-Middle (MitM) credential theft, session cookie interception.
  - *Business*: Massive loss of customer account confidentiality; non-compliance with PCI-DSS Requirement 4.
- **Recommended Mitigation**: Enforce TLS 1.3 across all routes; permanently redirect all HTTP port 80 traffic to HTTPS port 443; send the `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` header; disable legacy TLS 1.0/1.1 protocols and weak cipher suites.
- **Relevant Trust Boundary / Data Flow**: Traverses **TB-01** across data flows **DF-01**, **DF-02**, **DF-03**, **DF-05**.

---

### Category 5: Denial of Service (D)

#### Threat THREAT-D-01: Resource Exhaustion via Unbounded Wildcard Catalog Searches
- **STRIDE Category**: Denial of Service (Violates Availability)
- **Classification Reasoning**: Degrades system responsiveness and starves compute resources by inducing expensive database computations.
- **Affected Component / Asset**: `P3: Product & Order Service` & `DS1: Database` / `AST-07: Application Availability`
- **Threat Description**: If the product search endpoint (`/api/products/search?q=...`) performs unindexed regex searches or unbounded wildcard SQL queries (e.g., `WHERE description LIKE '%a%'`) without query length limits or database timeouts, an attacker can send concurrent search requests that consume 100% of the database CPU and memory.
- **Example Attack Scenario**:
  1. An attacker identifies that searching for single characters or regex operators causes a 3-second database delay.
  2. The attacker uses a lightweight script to spawn 50 concurrent threads requesting `GET /api/products/search?q=%25%25%25`.
  3. The PostgreSQL database exhausts its worker processes trying to scan millions of text rows.
  4. Legitimate shoppers attempting to browse products receive `504 Gateway Timeout` errors, halting all purchasing.
- **Potential Consequences**:
  - *Technical*: Database CPU saturation, memory exhaustion, connection starvation for other application services.
  - *Business*: Immediate disruption of sales, lost revenue during peak shopping events (e.g., Black Friday).
- **Recommended Mitigation**: Implement search query sanitization restricting search strings to 3–50 alphanumeric characters; use full-text search indexing (e.g., PostgreSQL tsvector or Elasticsearch) instead of leading wildcards; enforce strict query timeout limits (max 2 seconds) on database queries; enforce rate limiting on search endpoints (e.g., max 30 searches per minute per IP).
- **Relevant Trust Boundary / Data Flow**: Originates across **TB-01** (DF-01) and cascades through **TB-02** (DF-10, DF-11).

---

#### Threat THREAT-D-02: Automated Registration & Login Flood Exhausting Connection Pools
- **STRIDE Category**: Denial of Service (Violates Availability)
- **Classification Reasoning**: Deprives legitimate users of authentication services through deliberate compute and connection exhaustion.
- **Affected Component / Asset**: `P1: Web App` & `P2: Auth Service` / `AST-07: Availability`
- **Threat Description**: Cryptographic password hashing algorithms (Argon2id, bcrypt) are deliberately designed to be computationally expensive (CPU and memory intensive) to deter offline cracking. If registration and login endpoints lack rate limiting and concurrency limits, an attacker can flood the server with thousands of fake login attempts, overloading the server's CPU cores and exhausting database connection pools.
- **Example Attack Scenario**:
  1. An attacker fires 500 requests per second to `POST /api/auth/login` with random credentials.
  2. Each request triggers an Argon2id hashing computation on the backend CPU.
  3. Within 10 seconds, CPU utilization reaches 100%, and the database connection pool is completely saturated with waiting auth queries.
  4. Legitimate users are unable to log in, browse, or checkout, rendering the application unavailable.
- **Potential Consequences**:
  - *Technical*: Application thread deadlock, CPU starvation, complete authentication service outage.
  - *Business*: Downtime during critical trading hours, financial loss, public embarrassment.
- **Recommended Mitigation**: Deploy aggressive rate limiting at the edge reverse proxy / WAF (e.g., max 5 login attempts per minute per IP); implement proof-of-work or CAPTCHA challenges during sudden traffic spikes; queue hashing jobs with strict worker thread pool caps so background tasks cannot starve the main web event loop.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** (DF-01) and reaches **P2** via **DF-07**.

---

### Category 6: Elevation of Privilege (E)

#### Threat THREAT-E-01: Broken Access Control Permitting Regular Customers to Call Admin APIs
- **STRIDE Category**: Elevation of Privilege (Violates Authorization)
- **Classification Reasoning**: An unauthorized user with standard customer privileges successfully invokes high-privilege administrative functionality.
- **Affected Component / Asset**: `P1: Web App` & `P3: Product Service` / `AST-06: Catalog`, `AST-05: Orders`
- **Threat Description**: If administrative access controls rely solely on client-side UI hiding (e.g., hiding the "Admin Dashboard" link in the frontend) but fail to enforce server-side role validation middleware on backend administrative API endpoints (`/api/admin/*`), a standard customer can directly send HTTP requests to create, alter, or delete products and orders.
- **Example Attack Scenario**:
  1. A user creates a standard customer account and logs in, receiving a valid customer session token.
  2. The user inspects the frontend JavaScript bundle or uses guessing to discover the endpoint `POST /api/admin/products`.
  3. The user crafts a request to `POST /api/admin/products` with payload `{"name": "Free iPhone", "price": 0.00, "inventory": 100}` accompanied by their standard customer cookie.
  4. The backend endpoint executes the request because the controller does not check `if (session.user.role !== 'admin') return 403 Forbidden`.
  5. The unauthorized product is published, and the customer has effectively gained administrative superpowers.
- **Potential Consequences**:
  - *Technical*: Unauthorized administrative command execution, full integrity collapse of the product catalog.
  - *Business*: Complete operational compromise, unauthorized discounts, inventory sabotage.
- **Recommended Mitigation**: Enforce strict server-side Role-Based Access Control (RBAC) middleware on **every single** administrative route; adhere to a default-deny security policy; isolate administrative routes under a distinct URL namespace with independent session verification; log all unauthorized access attempts to **DS3** with immediate security alerts.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-04** (Admin Boundary) via **DF-01** / **DF-05** and processed by **P1** and **P3**.

---

#### Threat THREAT-E-02: JWT / Session Role Claim Manipulation Granting Superuser Rights
- **STRIDE Category**: Elevation of Privilege (Violates Authorization)
- **Classification Reasoning**: An attacker modifies their own authorization claims to elevate their security privileges within the system.
- **Affected Component / Asset**: `P2: Auth & Session Management Service` / `AST-02: Credentials` & `AST-03: Session Tokens`
- **Threat Description**: If SecureShop uses JSON Web Tokens (JWTs) for session state and fails to verify cryptographic signatures, accepts the `none` algorithm, or signs tokens using a weak or leaked secret key, an attacker can modify the payload from `{"user_id": 501, "role": "customer"}` to `{"user_id": 501, "role": "admin"}` and submit the altered token to execute administrative actions.
- **Example Attack Scenario**:
  1. Attacker registers as a regular customer and inspects their session JWT.
  2. Attacker modifies the payload header to `{"alg": "none"}` and the body to `{"sub": 501, "role": "admin"}`.
  3. If the server's JWT parsing library has the infamous `alg: none` vulnerability or does not strictly enforce the signature algorithm, it accepts the unverified token as valid.
  4. The attacker accesses all administrative endpoints as a verified superuser.
- **Potential Consequences**:
  - *Technical*: Complete authorization bypass, total takeover of administrative capabilities.
  - *Business*: Full system compromise, loss of customer data, unauthorized financial disbursements.
- **Recommended Mitigation**: Explicitly enforce strict asymmetric signing (e.g., RS256 or Ed25519) or strong symmetric keys (HS256 with at least 256 bits of entropy); reject any tokens specifying `alg: none`; store signing keys securely in a secrets manager; maintain an active token revocation blocklist in **DS2** for compromised or invalidated tokens.
- **Relevant Trust Boundary / Data Flow**: Crosses **TB-01** via data flow **DF-01** and processed by **P2** via **DF-07** and **DF-09**.

---

## 4. Beginner-Friendly Mentorship Section

### 4.1 Purpose of the Threat Analysis Document
The **STRIDE Threat Analysis Document** is the engine room of a threat modeling assessment. Its purpose is to systematically interrogate every component, connection, and data flow to uncover how an attacker could break the system.

### 4.2 Why It Matters
Without a structured methodology like STRIDE, security reviews rely purely on intuition or random guessing. STRIDE ensures that security engineers do not just look for "cool hacks" (like SQL injection) but also thoroughly examine subtle, devastating design flaws like missing audit trails (Repudiation), parameter tampering in checkout workflows (Tampering), or excessive search queries crashing the server (Denial of Service).

### 4.3 How It Was Created
1. Analyzed the Level 1 Data Flow Diagram created in `docs/02-system-architecture.md`.
2. Applied the STRIDE mnemonic to each of the 9 components and 14 data flows.
3. Formulated realistic, step-by-step attack scenarios based on real-world e-commerce breach patterns.
4. Mapped each threat to concrete technical mitigations and documented the violated security property.

### 4.4 How to Review This Document
- Confirm that every one of the six STRIDE categories contains at least two distinct, realistic threats.
- Verify that every threat has a unique identifier (`THREAT-S-01`, etc.) that links directly to the Risk Register in `docs/04-risk-register.md`.
- Ensure that the attack scenarios describe practical, real-world attack chains rather than vague, abstract statements.

---

### 4.5 Cybersecurity Reviewer Questions & Suggested Answers

**Q1: How do you prevent confusion between Tampering and Elevation of Privilege when an attacker tampers with their user ID?**
> *Answer:* Look at the *ultimate objective* and the *violated security property*. If an attacker modifies an order price or product quantity, they are altering business data integrity (Tampering). If an attacker changes their role from `customer` to `admin` or tampers with a parameter to access someone else's privileged administrative portal, the primary violation is unauthorized authorization expansion (Elevation of Privilege). We classify based on the primary security control failure.

**Q2: Why is unauthenticated webhook spoofing (THREAT-S-03) rated as Critical?**
> *Answer:* Webhooks represent an automated, server-to-server bridge across Trust Boundary TB-03. Because SecureShop offloads payment card processing to a third party, the payment confirmation webhook is the *sole authoritative trigger* for the warehouse to ship goods. If an attacker can spoof this notification without paying, they can order unlimited merchandise for free, causing catastrophic financial losses.

**Q3: Why is Repudiation considered a core threat category in e-commerce?**
> *Answer:* E-commerce systems handle financial transactions and contractual purchases. If the system fails to record immutable audit trails with customer attribution, the business cannot defend itself against credit card chargebacks ("friendly fraud"), regulatory compliance audits, or malicious insider actions. Non-repudiation provides the legal and forensic certainty required to operate commercially.
