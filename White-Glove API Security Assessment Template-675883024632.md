# White-Glove API Security Assessment Template

> Confluence Page ID: 675883024632, Version: 1

**White-Glove API Security Assessment for <API/Product>**
=========================================================

**As an:** API Security Architect  
**I want to:** Perform a comprehensive API security assessment of the <API/Product> platform and its customer-facing APIs  
**So that:** We can identify gaps across authentication, authorization, token handling, runtime protections, entitlement enforcement, and auditability to align <API/Product> with Lumen’s API Security Standards and prepare for future enhancements.

---

**1. Platform Security (Internal Posture)**
===========================================

Security of the platform hosting and exposing the API.

### **Acceptance Criteria**

* **[Confirm]** Sensitive platform secrets (API keys, certificates, DSNs) are stored in enterprise-approved vaults and rotated per policy.
* **[Confirm]** All service-to-service communication uses TLS 1.2+ with certificate validation.
* **[Verify]** Administrative and operational actions are audit-logged and logs are tamper-evident.

---

**2. Customer API Authentication (AuthN)**
==========================================

Assessment of OAuth 2.0 implementation, token lifecycle, onboarding, and identity flows.

### **Acceptance Criteria**

* **[Review]** End-to-end customer onboarding:

  + Application registration
  + Customer/account association
  + Client ID/Secret issuance
  + Partner behavior when acting on behalf of customers
* **[Verify]** OAuth 2.0 Client Credentials is implemented correctly (or document if another flow is used).
* **[Confirm]** No refresh tokens are issued for client\_credentials flows (required by OAuth spec).
* **[Review]** Token lifecycle:

  + Access token TTL (short-lived)
  + Client secret rotation
  + JWKS rotation
* **[Verify]** Token validation behavior:

  + issuer (iss)
  + audience (aud)
  + algorithm (RS256/ES256)
  + exp, nbf, iat handling
  + replay protection (e.g., JTI, nonce, or opaque tokens)
* **[Identify]** Gaps caused by **not using OAuth scopes today**, including:

  + lack of standardized permission representation
  + entitlements not represented in token
  + authorization logic scattered across backend services
  + reduced auditability of enforcement decisions

---

**3. Customer API Authorization (AuthZ)**
=========================================

Understanding how access decisions are made and enforced today.

### **Acceptance Criteria**

* **[Identify]** The current entitlement model used by <API/Product>:

  + read vs write vs config permissions
  + customer/account-level authorization
  + per-resource vs per-subresource permissions
  + partner delegation rules
  + implicit entitlements implemented in service code
* **[Document]** Where each entitlement is enforced today:

  + API Gateway (coarse authorization)
  + Resource Server (business authorization)
  + Hybrid or duplicated enforcement
  + Places with **no enforcement**
* **[Identify Gaps]** in the existing authorization model:

  + no OAuth scopes
  + inconsistent entitlement enforcement
  + token does not carry required authorization context
  + duplicated or siloed authorization logic
  + missing data-boundary protections
* **[Review]** Data boundary enforcement:

  + Does gateway ensure requested ID/account == token claims?
  + Do backend services re-validate?
  + Are bypass paths possible?
* **[Identify]** Missing claims that would be required for secure future authorization  
  *(capture only—do NOT propose new claims).*

---

**4. Gateway vs Resource Server Enforcement Responsibilities**
==============================================================

Clear mapping of runtime security responsibilities.

### **Acceptance Criteria**

**API Gateway (Current-State Assessment)**
------------------------------------------

Catalog what the gateway enforces:

* OAuth 2.0 token validation
* Signature, issuer, audience checks
* Basic boundary checks (IDs, account numbers, customer identifiers)
* Request size limits
* Content-type validation
* Schema validation (if enabled)
* Rejection of malformed JSON/XML

**Identify missing responsibilities**, such as:

* lack of entitlement checks
* no operation-level authorization
* no scope-driven authorization
* absence of a centralized entitlement model

---

**Resource Server (Current-State Assessment)**
----------------------------------------------

Catalog business-level authorization enforced in services:

* Account/ownership validation
* Write/configuration permissions
* Delegation logic for partners
* Operation-specific entitlements
* Privileged action audit trails

**Identify risks:**

* inconsistent implementation across services
* missing secondary validation checks
* unclear trust boundary between gateway and services
* authorization bypass opportunities

---

**5. API Gateway Runtime Security Controls**
============================================

Threat detection, prevention, and real-time hardening.

### **Acceptance Criteria**

* **[Verify]** Per-application and per-customer rate limiting & quota enforcement.
* **[Confirm]** WAF protections (SQLi, SSRF, XSS, injection attacks).
* **[Verify]** Enforcement of:

  + request/response size limits
  + allowed methods
  + schema validation
  + content-type restrictions
  + rejection of malformed payloads
* **[Review]** mTLS posture (internal or external).
* **[Confirm]** Sensitive data handling:

  + PII redaction
  + masking rules
  + no secrets in logs
* **[Verify]** Protections against:

  + ID/account/customer enumeration
  + brute-force attempts
  + credential stuffing
  + replay attacks

---

**6. Observability, Logging & Security Monitoring**
===================================================

### **Acceptance Criteria**

* **[Verify]** API logs include:

  + trace-id
  + client-id
  + account/customer ID
  + principal-id
  + request path
  + auth outcome
* **[Confirm]** Logs flow into SIEM (Splunk/Sentinel).
* **[Verify]** Alerts exist for anomalies:

  + auth failure spikes
  + elevated 403/429
  + abnormal traffic per customer
* **[Review]** Log retention policies for compliance.
* **[Confirm]** PII redaction and masking rules are applied uniformly.

---

**7. Threat Modeling & Vulnerability Assessment**
=================================================

### **Acceptance Criteria**

* **[Deliverable]** Conduct a threat model (STRIDE/LINDDUN) for the API.
* **[Identify]** Abuse cases such as:

  + resource/account enumeration
  + unauthorized config/state changes
  + partner impersonation
  + misuse of presigned URLs (if applicable)
* **[Review]** Latest penetration test results and confirm remediation.
* **[Recommend]** A targeted API pentest if coverage is insufficient.

---

**8. White-Glove Assessment Deliverables**
==========================================

### **Acceptance Criteria**

* **[Deliverable]** **API Security Assessment Report** including:

  + Authentication findings
  + Authorization & entitlement findings
  + Token lifecycle/correctness issues
  + Gateway vs resource server enforcement mapping
  + Runtime security gaps
  + Observability/logging gaps
  + Identified risks
  + Prioritized remediation plan
* **[Deliverable]** **Entitlement Discovery Document**  
  A catalog of all fine-grained entitlements that exist today across services and where they are enforced.
* **[Action]** Conduct alignment and review sessions with Product Owner & engineering team.
* **[Action]** Update customer-facing or internal documentation (AuthN/AuthZ behavior, claims, onboarding) to reflect validated security posture.