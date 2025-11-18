# API Standard: PATCH Method

> Confluence Page ID: 675851894814, Version: 3

none

**Purpose**
-----------

The `PATCH` method partially updates an existing resource at its canonical URI.  
Unlike `PUT`, it does not require the full representation, only the fields being changed.  
`PATCH` is **not inherently idempotent**, but can be made so when combined with `If-Match` and deterministic patch semantics.

This page defines Lumen’s governance requirements and best practices for designing, documenting, and implementing `PATCH` operations across APIs.

---

**Semantics**
-------------

| Property | Requirement |
| --- | --- |
| **Safe** | ❌ No, modifies server state. |
| **Idempotent** | ⚠ Not guaranteed, SHOULD use `If-Match` for concurrency control. |
| **Typical Use** | Partial modification of resource fields. |
| **Request Body** | ✅ Required, describes changes to apply. |
| **Response Body** | ✅ Required for `200 OK`, returns updated representation. |

---

**When to Use** `PATCH`
-----------------------

| Use Case | Example | Required Behavior |
| --- | --- | --- |
| **Modify selected fields** | `PATCH /v1/customers/{id}` | Updates only given attributes; returns `200 OK` with updated object. |
| **Change workflow state** | `PATCH /v1/orders/{id}` | Updates only `status` or `stage`. |
| **Partial configuration change** | `PATCH /v1/configurations/{id}` | Applies incremental changes. |
| **Partial update of singleton** | `PATCH /v1/organization/settings` | Updates subset of fields in a singleton resource. |

---

### ⚖️ **Governance Note — When PATCH Is Business-Critical**

> The `PATCH` method **MUST NOT** be used merely for developer convenience.  
> Teams **SHOULD use** `PATCH` **only when partial updates are business-critical** — meaning that sending or overwriting the entire resource representation (via `PUT`) would introduce data-loss, concurrency, or operational issues.

**PATCH is business-critical when:**

* Multiple systems or users own different parts of a resource (e.g., shared ownership of a profile).
* Partial mutation prevents overwriting external or read-only fields.
* Resource size or update frequency makes `PUT` inefficient.
* Incremental updates are required for workflows, IoT configuration, or multi-step state transitions.

**PATCH is** ***not*** **business-critical when:**

* The resource is small and easily replaced with `PUT`.
* The update semantics are simple (single owner, no concurrency risk).
* PATCH is used only to avoid sending redundant fields.

| Situation | Recommended Method | Rationale |
| --- | --- | --- |
| Toggle a single user preference or flag | **PATCH** | Partial update is isolated and critical to UX. |
| Update multiple independent fields in a shared resource | **PATCH** | Prevents data overwrite from concurrent systems. |
| Replace a complete entity such as customer, invoice, or address | **PUT** | Simple and predictable full-replacement semantics. |
| Minor API optimization (no concurrency risk) | **PUT** | Avoids unnecessary PATCH complexity. |

---

**Singleton Resources**
-----------------------

### **Definition**

A singleton resource (e.g., `/configuration`, `/profile`, `/settings`, `/status`) represents a unique, addressable entity that may be mutable.

### **Guidance**

| Rule | Description |
| --- | --- |
| **MAY use PATCH** | For partial updates to existing or pre-created singletons. |
| **MAY use PUT** | When replacing the entire representation. |
| **MUST include If-Match** | To prevent overwriting concurrent changes. |
| **MUST return 200 OK** | With updated representation on success. |
| **MUST return 412 Precondition Failed** | If `If-Match` does not match current ETag. |
| **SHOULD include ETag** | Every GET/PATCH response must include new ETag. |
| **MUST NOT** | Use PATCH to create a singleton — creation uses PUT. |

**Example**

PATCH /v1/organization/settings
If-Match: "v1"
Content-Type: application/json
{
"auto\_approve": false
}
HTTP/1.1 200 OK
ETag: "v2"
Content-Type: application/json
{
"auto\_approve": false,
"timezone": "America/Chicago"
}

---

**Patch Semantics**
-------------------

### **JSON Merge Patch (RFC 7396) — Preferred**

Simplified document-based merge:  
present keys → overwrite,  
`null` → delete,  
absent → no change.

PATCH /v1/customers/123
Content-Type: application/merge-patch+json
{ "status": "inactive" }

---

### **JSON Patch (RFC 6902) — Optional**

Explicit, ordered operations for atomic control.

PATCH /v1/customers/123
Content-Type: application/json-patch+json
[
{ "op": "replace", "path": "/status", "value": "inactive" },
{ "op": "remove", "path": "/temporary\_flag" }
]

---

### ⚖️ **Governance Callout — PATCH Format Standards**

> **Lumen Default Standard:**  
> All Lumen APIs **MUST** implement `PATCH` using **JSON Merge Patch (RFC 7396)** with  
> `Content-Type: application/merge-patch+json`.

**Rationale**

* Aligns with standard REST partial-update semantics.
* Easy for developers; payload mirrors resource shape.
* Fully idempotent when combined with `ETag + If-Match`.

**Optional Alternative (Advanced Use Only)**  
`application/json-patch+json` (**RFC 6902**) **MAY** be supported for:

* Explicit `add` / `remove` / `replace` operations.
* Ordered multi-field or audited mutations.
* Complex structured documents.

**Governance Enforcement**

| Rule | Description |
| --- | --- |
| **MUST** | Default `Content-Type` = `application/merge-patch+json`. |
| **MAY** | Support `application/json-patch+json` only after design review. |
| **MUST NOT** | Implicitly accept both without declaring them in OAS. |
| **MUST** | Validate PATCH media type via Spectral rule. |

**Spectral Example**

rules:
patch-must-use-merge-patch:
description: "PATCH operations must default to application/merge-patch+json"
given: "$.paths[\*].patch.requestBody.content"
then:
field: "application/merge-patch+json"
function: truthy

**Summary**

| Aspect | Governance Standard |
| --- | --- |
| Default RFC | **RFC 7396 — JSON Merge Patch** |
| Optional RFC | RFC 6902 — JSON Patch |
| Default Content-Type | `application/merge-patch+json` |
| Optional Content-Type | `application/json-patch+json` (advanced) |
| Idempotency | `ETag + If-Match` required |
| Documentation | OAS MUST explicitly list supported types |

---

**Conditional Requests & Concurrency**
--------------------------------------

| Header | Usage | Example |
| --- | --- | --- |
| `ETag` | Returned with GET/PATCH responses. | `ETag: "v10"` |
| `If-Match` | Required for optimistic concurrency. | `If-Match: "v10"` |

**Rules**

* **MUST** return new ETag after successful PATCH.
* **MUST** reject updates without `If-Match` when concurrent writes possible.
* **MUST** return `412 Precondition Failed` on mismatch.
* **MUST NOT** overwrite concurrent updates.

---

**Idempotency**
---------------

`PATCH` is not guaranteed idempotent but SHOULD be made so where possible.

| Technique | Description |
| --- | --- |
| **ETag + If-Match** | Ensures update applies only to current version. |
| **Idempotency-Key** | Optional for safe retry behavior. |
| **Deterministic Patch** | Same payload → same final state. |

---

**Example**
-----------

PATCH /v1/customers/123
If-Match: "v4"
Content-Type: application/merge-patch+json
{
"email": "jane.doe@example.com",
"status": "inactive"
}
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v5"
{
"id": "123",
"name": "Jane Doe",
"email": "jane.doe@example.com",
"status": "inactive"
}

---

**Status Codes**
----------------

| Code | Meaning | Applies To | Notes |
| --- | --- | --- | --- |
| **200 OK** | Resource partially updated | ✅ Existing | MUST return updated representation. |
| **204 No Content** | Update succeeded (no body) | ⚠ Optional | Only if representation unchanged. |
| **400 Bad Request** | Invalid JSON or schema | ✅ Both | Syntax errors. |
| **401 Unauthorized** | Authentication required | ✅ Both |  |
| **403 Forbidden** | Authenticated but not permitted | ✅ Both |  |
| **404 Not Found** | Resource missing | ✅ Both |  |
| **405 Method Not Allowed** | Wrong verb | ✅ Both |  |
| **409 Conflict** | Version/state conflict | ✅ Both | Patch cannot apply. |
| **412 Precondition Failed** | `If-Match` mismatch | ✅ Both | Stale ETag. |
| **422 Unprocessable Content** | Semantic validation failure | ✅ Both | Violates business rule. |
| **500 Internal Server Error** | Unexpected failure | ✅ Both | No internal details. |

---

**OAS Authoring Requirements**
------------------------------

paths:
/v1/customers/{id}:
patch:
summary: Partially update customer
parameters:
- in: path
name: id
required: true
schema: { type: string }
- in: header
name: If-Match
schema: { type: string }
requestBody:
required: true
content:
application/merge-patch+json:
schema: { $ref: '#/components/schemas/CustomerPatch' }
responses:
'200':
description: Successful partial update
headers:
ETag:
description: Entity tag for versioning
schema: { type: string }
'412': { description: Precondition failed (stale ETag) }
'400': { description: Invalid request }
'404': { description: Not found }

---

**Governance Checklist**
------------------------

| Check | Validation |
| --- | --- |
| ✅ Partial update only | PATCH does not require full resource. |
| ✅ Concurrency control | `ETag` + `If-Match` required. |
| ✅ Media type governed | Default RFC 7396 only. |
| ✅ Status codes approved | 200/204/400/401/403/404/405/409/412/422/500. |
| ✅ Error model compliant | RFC 7807 Problem Details. |
| ✅ OAS validated via Spectral | Governance automation ready. |

---

**Developer Summary**
---------------------

* Use `PATCH` for **partial updates** only.
* Default to **JSON Merge Patch (RFC 7396)**.
* Use `If-Match` and `ETag` for safe concurrency.
* Return `200 OK` with updated representation or `204 No Content` if unchanged.
* For **singleton resources**, use PATCH for partial changes, PUT for full replacement.
* Follow approved status codes and RFC 7807 error structure.
* Declare supported patch type(s) explicitly in OAS.
* All `PATCH` endpoints must pass Spectral linting for governance compliance.