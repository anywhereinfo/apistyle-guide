# API Security Standard: Identity Derivation, Multi-Tenancy & Zero Trust

> Confluence Page ID: 675893968900, Version: 1

**1. Purpose**
==============

This standard defines how APIs must derive user identity and tenant context within a **Strict Zero Trust Architecture (ZTNA)**.

Zero Trust assumes the internal network is hostile, therefore:

* **Identity must always come from a cryptographically verified token**, and
* **Tenant context must always be explicitly provided and validated**, not inferred.

This prevents spoofing, privilege escalation, cross-tenant leakage, and ambiguous authorization behavior across APIs and services.

---

**2. Core Identity Principle**
==============================

### **Identity MUST come from the verified authentication token.**

Backends must NOT accept identity hints from:

* Request bodies
* Custom or forwarded headers
* Query parameters
* Emails supplied by the caller

The token is the **single source of truth** for actor identity.

Actor identity (`who is performing the action`) is derived from token claims such as:

* `sub` (subject identifier — REQUIRED)
* `email` (optional contact)
* `scp` / `scope`
* `roles` / `groups` (optional)

Notification recipients (e.g., `notification_email`, `manager_email`) may appear in the request body, but they **never serve as acting-user identity**.

---

**3. Allowed vs Forbidden Identity Scenarios**
==============================================

| Scenario | Allowed? | Reason |
| --- | --- | --- |
| **A. “Notify Me”** – Acting user wants an email | **FORBIDDEN** | System must derive acting user's email from token. Caller cannot override identity. |
| **B. “Notify Boss / Another Party”** | **ALLOWED** | Additional recipients are not identity. Body fields allowed when clearly named. |
| **C. “Act on My Behalf (Self)”** | **FORBIDDEN** | Actor must come from token’s `sub`. No identity in request payload. |
| **D. “Admin Acting on Behalf of End User”** | **SPECIAL CASE** | Requires delegated authorization (RFC 8693 `act` claim). Backend validates both admin & represented user. |
| **E. “Partner Acting on Behalf of Customer”** | **SPECIAL CASE** | Allowed only with delegated access tokens or JWT assertions. Backend validates partner → customer linkage via scopes & delegation rules. |
| **F. “SaaS / Automation App Acting on Behalf of User”** | **SPECIAL CASE** | Requires delegated user-consent tokens. Backend validates app identity, user identity, and granted scopes. |

---

**4. Gateway Requirements (Defense in Depth)**
==============================================

Gateways MUST:

### **1. Validate**

* Perform preliminary signature + expiry checks
* Reject invalid tokens early

### **2. Propagate**

Forward the **original** `Authorization: Bearer <token>` header unchanged.

### **3. Sanitize**

Strip all caller-provided identity headers:

* `X-User`
* `X-Email`
* `X-Authenticated-User`
* `X-Acting-User`

Gateways may add diagnostic headers, but they are **non-authoritative**.

---

**5. Backend Requirements (True Zero Trust)**
=============================================

Backends MUST independently validate:

### **1. Token authenticity**

Signature, issuer, audience, expiry, and required claims.

### **2. Actor identity**

Derived only from the token (`sub`, optional claims).

### **3. Delegated identity (acting on behalf of)**

If token contains delegated identity (e.g., `act`, RFC 8693):

* Validate both the acting identity and represented identity
* Validate permitted delegated scopes
* Enforce tenant and resource boundaries

### **4. Tenant boundaries**

The backend must verify the caller is authorized to act *in the tenant they specify*.

### **5. No identity in payload**

Identity hints in request bodies must be ignored and should not be present in compliant APIs.

---

**6. Multi-Tenant Identity & Authorization**
============================================

Multi-tenant architectures allow users, partners, and systems to interact with multiple organizations or accounts.  
Zero Trust requires **explicit tenant selection** and **verifiable tenant authorization** for every request.

Identity = Who  
Tenant = Where / On behalf of whom

Both must be provided independently.

---

**6.1 Identity vs Tenant: Critical Distinction**
================================================

### **Identity (actor) — provided by token**

* `sub`
* `email`
* `scp`
* `roles` / `groups`
* `partner_id` (optional)

### **Tenant (context) — provided by the request**

* `tenant_id`
* `org_id`
* `account_id`

**Identity tells us who is calling.**  
**Tenant tells us where they are acting.**

They MUST NOT be collapsed into each other.

---

**6.2 Required Token Claims for Multi-Tenant Context**
======================================================

To enforce tenant-specific authorization, tokens MUST include:

| Claim | Purpose |
| --- | --- |
| `sub` | Who the actor is |
| `scp` **/** `scope` | What actions are allowed |
| `tenant_membership` **(custom)** | List of tenants the user has access to (recommended) |
| `act` **(delegated identity, RFC 8693)** | For acting on behalf of another user/tenant |
| `partner_id` **(custom)** | Identifies partner organization (if applicable) |
| `azp` | Authorized party for SaaS/on-behalf-of flows |

---

**6.3 Explicit Tenant Selection (Mandatory)**
=============================================

The API MUST require the caller to explicitly specify the tenant they are acting in, e.g.:

```
/tenants/{tenant_id}/orders
POST /orders?tenant_id=12345
```

### **Why must the tenant be provided explicitly?**

1. **Users often belong to multiple tenants** (consultants, partners, admins).
2. **Tokens do not represent tenant intent**, only identity.
3. **Inference creates ambiguity** and breaks Zero Trust.
4. **Explicitness prevents cross-tenant privilege escalation.**
5. **Auditing demands clear, deliberate tenant selection.**
6. **Identity ≠ Tenant**, and must not imply tenant context.
7. **Delegated access requires explicit “on behalf of” tenant resolution.**

**Explicit tenant selection is the only way to guarantee correct authorization in multi-tenant APIs.**

---

**6.4 Tenant Cannot Be Inferred**
=================================

The tenant MUST NOT be inferred from:

* “default tenant” settings
* email domain
* token issuer or IdP tenant
* resource IDs
* internal lookup
* user profile metadata

### **Why inference is banned**

* Creates unintended privilege escalation
* Causes operations on the wrong tenant
* Breaks federated identity expectations
* Violates Zero Trust: **Assume nothing. Validate everything.**
* Inconsistent behavior across APIs
* Impossible to audit reliably

---

**6.5 Tenant Membership Validation**
====================================

Once tenant is explicitly provided, backend MUST verify:

1. Caller is mapped to that tenant
2. Caller has appropriate scope for that tenant
3. Delegation or admin privileges (if cross-tenant)
4. Access is logged for audit
5. If any check fails → **403 Forbidden**

---

**6.6 Cross-Tenant Access**
===========================

Cross-tenant access is **never permitted by default**.  
It is only allowed via **delegated authorization** or **elevated admin scopes**.

### **Pattern 1 — Delegated Access (“Acting on behalf of Customer”)**

Used for:

* Partners
* Resellers
* SaaS integrations
* Bots / automation systems

Required:

* Token with `act` **claim** (RFC 8693)
* Delegated scopes granting partner → customer access
* Optional constraints like `partner_id`, `customer_id`

Backend validates:

* Partner identity
* Customer identity
* Delegation chain
* Allowed scopes
* Tenant boundaries

---

### **Pattern 2 — Admin Scopes (Internal elevated access)**

Used by:

* Support teams
* NOC / operations
* Internal engineering tools

Required:

* Token containing admin-level scopes (e.g., `tenant.admin`, `customer.manage`)
* Optional delegated identity (`act`) if impersonation occurs

Backend validates:

* Admin’s identity
* Admin privileges
* Tenant scope
* Operation type matches admin rights

---

**6.7 Multi-Tenant Scenarios**
==============================

| Scenario | Allowed? | Reason |
| --- | --- | --- |
| **MT-1: User acts in tenant they belong to** | ✔️ ALLOWED | Tenant is explicit; membership & scopes validated. |
| **MT-2: User attempts to act in tenant they do NOT belong to** | ❌ FORBIDDEN | Tenant isolation violation; return `403`. |
| **MT-3: Internal Admin acts across tenants** | ⚠️ SPECIAL CASE | Allowed only with admin scopes/delegation. |
| **MT-4: Partner acts on behalf of Customer** | ⚠️ SPECIAL CASE | Must use delegated tokens (RFC 8693) and authorized scopes. |
| **MT-5: SaaS app acts on behalf of end-user** | ⚠️ SPECIAL CASE | Requires delegated user-consent token and validated authorization chain. |

---

**7. PII & Email Handling**
===========================

### **Actor Email**

* Always derived from token
* Never supplied in payload
* If wrong → user updates profile in IdP

### **Notification Emails**

Allowed only for contacts that are **not the actor**, e.g.:

* `manager_email`
* `approver_email`
* `notification_email`

---

**8. Governance as Code (Spectral Rule)**
=========================================

```
rules:
  no-acting-user-in-body:
    description: "Acting user identity must be derived from the authentication token, not from the request body."
    severity: error
    given: "$.paths[*][*][?(@.security)]..requestBody.content.*.schema.properties"
    then:
      field: "@key"
      function: pattern
      functionOptions:
        notMatch: "^(user_email|actor_email|acting_user|customer_email|user_id)$"
```

---

**9. Migration Plan**
=====================

| Phase | Action |
| --- | --- |
| 1. **Discover** | Identify APIs containing acting user identity in payload. |
| 2. **Backend Update** | Switch identity derivation to token. |
| 3. **Contract Update** | Remove identity fields; publish new versions. |
| 4. **Deprecate** | Follow COM/API deprecation lifecycle. |
| 5. **Hard Gate** | Gateway blocks requests that include forbidden identity fields. |

---

10. Final Summary
=================

**Identity = Token**  
**Tenant = Explicit**  
**Delegation = RFC 8693 or Admin Scopes**  
**Notification ≠ Identity**  
**Headers = Untrusted**  
**Zero Trust = Verify at Every Layer**