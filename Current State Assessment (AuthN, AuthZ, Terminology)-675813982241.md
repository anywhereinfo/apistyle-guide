# Current State Assessment (AuthN, AuthZ, Terminology)

> Confluence Page ID: 675813982241, Version: 2

**Story Summary**
-----------------

Document the *current state* of Lumen’s API security model — including authentication, authorization, customer onboarding, and terminology — across all domains and customer types.  
This baseline will identify current enforcement mechanisms, systems of record, and gaps before the introduction of standardized OAuth 2.1 and scoped access models.

---

**Problem Statement**
---------------------

Lumen’s API security landscape has evolved organically across multiple platforms and customer types (Partner, Enterprise, Federal, Internal).  
Today, teams use inconsistent terminology (e.g., “API Key” = `client_id + secret`), and authentication/authorization responsibilities are distributed across multiple systems (Gateway, Resource Server, Onboarding, etc).  
There is no unified documentation of *who authenticates*, *who authorizes*, or *what each term means*. This story establishes a comprehensive **current-state inventory** to support the design of future standardized security patterns.

---

**Scope**
---------

### **In Scope:**

* All externally (high prioirity) and internally (lower priority) exposed API domains (e.g., MCGW, Fabric, Billing, Provisioning, Customer360).
* Customer types: Partner, Enterprise, Internal, Federal.
* Systems involved in authentication, authorization, and onboarding.
* Terminology as used in current gateway policies, onboarding documentation, and partner contracts.

**Out of Scope:**

* Designing new OAuth 2.1 or OpenID Connect flows (Phase 2).
* Remediation or enforcement changes.

### **Deliverables**

1. **Confluence Documentation:**

   * A dedicated page per *Domain × Customer Type* using the “API Security Current-State Template”.
   * Completed sections for:

     + Authentication (AuthN)
     + Authorization (AuthZ)
     + Acting-on-Behalf-of / Partner–Customer Entitlement
     + Network & Transport Controls
     + Customer Onboarding
   * Attached **flow diagram** illustrating AuthN/AuthZ interactions and system responsibilities.
   * Evidence links to existing documentation.
2. **Terminology Glossary:**

   * Current-to-Standard mapping table (API Key ↔ client\_id/client\_secret, etc.).
   * Defined list of Lumen-specific terms and their formal equivalents (RFC 6749, 8705, 9068 alignment).
3. **System Inventory Table:**

   * Each system’s role (Authentication, Authorization, Onboarding, Audit).
   * Integration points between gateway, onboarding, resource servers, and PKI.
4. **Summary Analysis:**

   * Key observations and known gaps.
   * Diagram or table summarizing enforcement responsibility (Gateway vs Resource vs Onboarding).

### **Acceptance Criteria**

| # | Acceptance Criterion | Verification Method |
| --- | --- | --- |
| 1 | A Confluence page exists per active domain capturing all AuthN/AuthZ attributes using the standard template. | Review completed Confluence pages. |
| 2 | Each page includes a flow diagram showing systems and enforcement responsibilities. | Verify diagram presence and accuracy. |
| 3 | Terminology glossary page created and reviewed by API Governance & Security teams. | Cross-review against gateway configs and onboarding documentation. |
| 4 | System responsibility matrix completed for Gateway, Resource, Onboarding, IdP, and PKI. | Governance review. |
| 5 | Key gaps and inconsistencies documented and approved in governance log. | Review meeting minutes. |

### **Story Outcome**

When complete, Lumen will have a **governance-grade baseline** describing:

* Which systems perform authentication and authorization
* How customer onboarding issues credentials
* What terminology each team uses
* Where inconsistencies or gaps exist

This becomes the authoritative input to **Phase 2 – Security Standardization & Scopes Adoption**.

Details to Capture
------------------

### Overview

| Field | Description |
| --- | --- |
| *Domain / Product* | e.g. MCGW / Fabric / Billing |
| *Customer Type(s)* | Partner / Enterprise / Internal / Federal |
| *Exposure Channel* | Public-Internet / Partner-VPN / Lumen-Intranet / GovNet |
| *Business Owner* | Team or contact responsible |

### Authentication ( AuthN )

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *Entry Point* | Where authentication occurs | API-Gateway / Direct-Service |
| *AuthN Scheme* | Mechanism used | mTLS / API-Key / JWT-Bearer / Basic / None |
| *IdP / Issuer* | Token or cert issuer | Internal-IdP / Ping / Keycloak / Partner-Provided |
| *Credential Type / Store* | Key / Cert / JWT and where stored | Vault / GW-KV / DB |
| *Token Format* | If JWT | Opaque / JWT (JWS) / JWT (JWE) |
| *Token Audience (aud)* | Expected audience | Free text |
| *Token TTL (min)* | Duration | Integer |
| *TLS Version Min* | Transport security level | TLS 1.2 / TLS 1.3 |
| *mTLS Verification Mode* | Identity check method | SAN-DNS / SAN-URI / CN |
| *Secrets Rotation Policy* | Rotation cadence | ≤90d / ≤180d / Ad-hoc |
| *AuthN Owner System* | Which system enforces | API Gateway / IdP / Hybrid |

### Authorization ( AuthZ )

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *AuthZ Model* | Access-control type | None / RBAC (GW) / RBAC (Resource) / ABAC-Claims / Endpoint-Allowlist / Tenant-Partition / Backend-ACL |
| *Entitlement Source* | Where rules reside | GW-Policy / Client-Profile / Partner-Contract / Backend-ACL |
| *AuthZ Decision Point (PDP)* | Who decides | Gateway / Resource / External-PDP / Hybrid |
| *AuthZ Enforcement (PEP)* | Who enforces | Gateway / Resource / Both |
| *Gateway Enforcement* | Coarse-grained policies | JWT-Signature / Client-Cert / Rate-Limit / Header-Policy / Size-Limit |
| *Resource Enforcement* | Fine-grained policies | Object-Level-ACL / Tenant-Filter / Partner-Entitlement |
| *Entitlement Registry Source* | Data source validating Partner↔Customer | Partner\_Customer\_Link / CRM / Custom DB |

### Acting-On-Behalf-Of / Delegated Access

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *Acting-On-Behalf Model* | Delegation type | None / Partner-On-Behalf-Of-Customer / Internal-On-Behalf-Of-Partner |
| *Customer Context Assertion* | How customer ID conveyed | Header (X-Customer-Id) / JWT Claim (customer\_id) / Path Param |
| *Partner-Customer Validation Source* | Where entitlement checked | Entitlement DB / Gateway / Consent Service |
| *Consent Mechanism* | Customer consent model | None / Static Contract / Dynamic Consent / Open-Banking-Style |
| *Audit Trail Completeness* | Whether both IDs logged | Full / Partial / None |

### Network & Transport Controls

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *Network Path / Zone* | Boundary type | Internet / VPN / GovNet / Private-Link / Intranet |
| *IP / Geo Restrictions* | Network restrictions | None / IP-Allowlist / Geo-Block / VPN-Only / GovNet-Only |
| *FIPS Requirement* | Crypto compliance | None / FIPS-140-2 / FIPS-140-3 |
| *Forward Secrecy Required* | FS enabled | Yes / No / Unknown |

### Data Classification & Handling

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *Data Classification* | Sensitivity | None / PII / PCI / PHI / Mixed |

### Customer Onboarding Process

| Attribute | Description | Allowed Values / Examples |
| --- | --- | --- |
| *Onboarding System* | Source system | Portal / CRM / Manual Workflow |
| *Onboarding Mode* | How partner/customer is onboarded | Manual-Registration / Self-Service-Portal / Ticketed / Contract-Only |
| *Credential Issuance* | How credentials delivered | Portal-Download / Email / Automated API |
| *Entitlement Recording* | Where relationship stored | Partner\_Customer\_Link / CRM / Database |
| *Rotation / Revocation Process* | How keys or certs rotated | Manual / Automated / N/A |
| *Governance Owner* | Team managing onboarding | Security Ops / API Governance / Partner Mgmt |