# Trustworthy Cybersecurity References & Technical Standards

This document catalogs the authoritative, industry-standard cybersecurity resources, guidelines, and frameworks that ground the **SecureShop Threat Model and Hardening Assessment**. Each resource includes an official reference link and a specific explanation of how it directly informed the architecture, threat modeling, risk evaluation, and defensive controls of this project.

---

## 1. Threat Modeling Methodologies & Frameworks

### 1.1 OWASP Threat Modeling Cheat Sheet
- **Authoritative Organization**: Open Web Application Security Project (OWASP)
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
- **Assessment Application**:
  - Provided the foundational four-question framework: *What are we building? What can go wrong? What are we going to do about it? Did we do a good enough job?*
  - Guided the creation of the Level 1 Data Flow Diagram (DFD) and the systematic decomposition of SecureShop into external entities, processes, data stores, and data flows.

### 1.2 Microsoft STRIDE Threat Modeling Documentation
- **Authoritative Organization**: Microsoft Security Engineering
- **Official Resource Link**: [https://learn.microsoft.com/en-us/previous-versions/commerce-server/ee823878(v=cs.20)](https://learn.microsoft.com/en-us/previous-versions/commerce-server/ee823878(v=cs.20))
- **Additional Guide**: [https://owasp.org/www-community/Threat_Modeling_Process](https://owasp.org/www-community/Threat_Modeling_Process)
- **Assessment Application**:
  - Supplied the core categorization taxonomy: **S**poofing, **T**ampering, **R**epudiation, **I**nformation Disclosure, **D**enial of Service, and **E**levation of Privilege.
  - Dictated the STRIDE-per-element analysis used in `docs/03-threat-analysis.md` to identify the 15 granular threat scenarios against SecureShop's components.

---

## 2. Risk Assessment Standards

### 2.1 NIST Special Publication 800-30 Revision 1
- **Title**: *Guide for Conducting Risk Assessments*
- **Authoritative Organization**: National Institute of Standards and Technology (NIST), U.S. Department of Commerce
- **Official Resource Link**: [https://csrc.nist.gov/publications/detail/sp/800-30/rev-1/final](https://csrc.nist.gov/publications/detail/sp/800-30/rev-1/final)
- **Assessment Application**:
  - Established the semi-quantitative risk calculation methodology ($\text{Risk Score} = \text{Likelihood} \times \text{Impact}$).
  - Provided the conceptual foundation for defining Likelihood (1–5) based on threat capability and attack surface, and Impact (1–5) based on organizational harm.
  - Grounded the distinction between initial inherent risk and post-hardening residual risk in `docs/04-risk-register.md`.

### 2.2 NIST Special Publication 800-53 Revision 5
- **Title**: *Security and Privacy Controls for Information Systems and Organizations*
- **Authoritative Organization**: National Institute of Standards and Technology (NIST)
- **Official Resource Link**: [https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- **Assessment Application**:
  - Guided control engineering in `docs/05-hardening-checklist.md` across Access Control (AC), Audit and Accountability (AU), Identification and Authentication (IA), and System and Communications Protection (SC).

---

## 3. Web Application Security Standards & Hardening

### 3.1 OWASP Top 10 Web Application Security Risks (2021)
- **Authoritative Organization**: OWASP Foundation
- **Official Resource Link**: [https://owasp.org/Top10/](https://owasp.org/Top10/)
- **Assessment Application**:
  - Directly informed threat selection and prioritization:
    - **A01:2021 - Broken Access Control**: Mapped to `THREAT-E-01` (Admin API access) and `THREAT-I-01` (IDOR order access).
    - **A02:2021 - Cryptographic Failures**: Mapped to `THREAT-I-03` (Transport encryption) and `SEC-CTRL-01` (Argon2id hashing).
    - **A03:2021 - Injection**: Mapped to `THREAT-T-02` (SQL injection via search).
    - **A04:2021 - Insecure Design**: Addressed via the threat modeling process itself and client-side cart tampering (`THREAT-T-01`).
    - **A05:2021 - Security Misconfiguration**: Addressed by verbose error handling suppression (`THREAT-I-02`) and HTTP headers (`SEC-CTRL-12`).
    - **A07:2021 - Identification and Authentication Failures**: Addressed by brute force defense (`THREAT-S-01`) and session security (`THREAT-S-02`).
    - **A09:2021 - Security Logging and Monitoring Failures**: Addressed by repudiation threats (`THREAT-R-01`, `THREAT-R-02`) and audit logging controls (`SEC-CTRL-15`).

### 3.2 OWASP Application Security Verification Standard (ASVS v4.0.3)
- **Authoritative Organization**: OWASP Foundation
- **Official Resource Link**: [https://owasp.org/www-project-application-security-verification-standard/](https://owasp.org/www-project-application-security-verification-standard/)
- **Assessment Application**:
  - Provided the benchmark for defining precise **Verification Methods** for each hardening control in `docs/05-hardening-checklist.md`.
  - Informed session management standards (Chapter V3), access control verifications (Chapter V4), and stored data protection (Chapter V8).

---

## 4. Specialized Technical Cheat Sheets (OWASP Cheat Sheet Series)

### 4.1 Authentication & Password Storage
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- **Official Password Storage Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- **Assessment Application**:
  - Dictated the selection of Argon2id with memory-hard parameters ($m=65536, t=3, p=4$) and bcrypt work factor $\ge 12$ in `SEC-CTRL-01`.
  - Defined modern password length recommendations ($\ge 12$ characters) aligned with NIST SP 800-63B.

### 4.2 Session Management
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- **Assessment Application**:
  - Dictated cookie hardening requirements (`HttpOnly`, `Secure`, `SameSite=Strict`), 128-bit CSPRNG token entropy, session rotation upon privilege elevation, and 30-minute idle session timeouts in `SEC-CTRL-04`.

### 4.3 Authorization & Access Control
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- **Official IDOR Prevention Link**: [https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_References_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_References_Prevention_Cheat_Sheet.html)
- **Assessment Application**:
  - Provided the dual-defense approach for IDOR in `SEC-CTRL-03`: server-side user identity binding combined with non-sequential UUIDv4 object identifiers.

### 4.4 SQL Injection Defense
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- **Assessment Application**:
  - Established the universal parameterized query and prepared statement requirement in `SEC-CTRL-10`, rejecting raw string interpolation.

### 4.5 REST Security & Error Handling
- **Official Resource Link**: [https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)
- **Assessment Application**:
  - Guided the design of centralized error-handling middleware returning generic user responses with opaque tracking IDs in `SEC-CTRL-14`.

---

## 5. Payment Industry & Compliance Standards

### 5.1 PCI-DSS v4.0 (Payment Card Industry Data Security Standard)
- **Title**: *PCI Data Security Standard: Requirements and Testing Procedures v4.0*
- **Authoritative Organization**: PCI Security Standards Council
- **Official Resource Link**: [https://www.pcisecuritystandards.org/document_library/](https://www.pcisecuritystandards.org/document_library/)
- **Assessment Application**:
  - Governed the architectural decision to adopt the **Self-Assessment Questionnaire (SAQ) A** model.
  - Confirmed that delegating card capture entirely to a certified third-party gateway hosted iframe or redirect eliminates raw cardholder data from SecureShop's internal databases, radically decreasing the application's attack surface and compliance burden.
