# API Standard: Lumen Problem Details — Minimal Profile (LPDP-Mini v1.0)

> Confluence Page ID: 675749954024, Version: 2

none

Lumen Problem Details — Minimal Profile (LPDP-Mini v1.0)
========================================================

### Purpose

Provide a **lightweight, RFC 9457-compliant** error envelope that’s consistent across all Lumen APIs.  
It focuses on three essentials — `code`, `message`, and optional `meta` — while letting teams extend `meta` per API as needed.

---

Profile Summary
---------------

| **Design Goal** | **Description** |
| --- | --- |
| **Simplicity** | Practical & viable RFC 9457 extension that developers can implement correctly. |
| **Stability** | `code` is machine-readable and stable across languages and messages. |
| **Extensibility** | `meta` can be open, omitted, or refined per API. |
| **Security** | No PII, credentials, or stack traces in error payloads. |
| **Compliance** | Fully aligns with [RFC 9457 – Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457). |

---

Core Structure (Org-wide Standard)
----------------------------------

### Media Type

All error responses **MUST** use:  
`Content-Type: application/problem+json`

### JSON Schema (OpenAPI fragment)

components:
schemas:
LpdpProblem:
type: object
additionalProperties: false
description: Minimal Problem-Details envelope (RFC 9457-compliant)
properties:
title:
type: string
description: Short, human-readable summary.
detail:
type: string
description: Optional longer explanation.
errors:
type: array
minItems: 1
items: { $ref: '#/components/schemas/LpdpError' }
required: [ title, errors ]
LpdpError:
type: object
additionalProperties: false
description: Individual error item.
properties:
code:
type: string
description: Stable, machine-readable identifier (string form preferred).
message:
type: string
description: Human-readable explanation of the issue.
meta:
type: object
description: >
Optional unregulated extension for machine-readable context.
Lumen does not prescribe its structure. Teams may define custom
schemas if needed.
additionalProperties: true
required: [ code, message ]

---

RFC 9457 Alignment
------------------

| RFC 9457 Member | LPDP-Mini Treatment | Rationale |
| --- | --- | --- |
| `type` | Optional / constant URI | RFC 9457 §3.1.1 allows omission. |
| `title` | Required | Short human summary. |
| `status` | Omitted | HTTP status line authoritative. |
| `detail` | Optional | Narrative text. |
| `instance` | Optional | Rarely used. |
| `errors` | Extension (§3.2) | Array of per-error details. |
| `code`,`message`,`meta` | Extensions | Fully compliant. |

---

`code` — Role and Representation
--------------------------------

### What `code` is

A **stable, machine-readable key** identifying the error category; independent of message wording or localization.

### Naming Convention

`<domain>.<category>.<reason>`  
Examples: `auth.missing`, `resource.not_found`, `dependency.failed`.

### Why string codes are standard

| Aspect | String (`auth.missing`) | Numeric (`1043`) |
| --- | --- | --- |
| Readability | Human-friendly in logs | Opaque |
| Namespacing | Simple (`auth.*`) | None |
| Client branching | Safe (`if code==…`) | Ambiguous |
| Governance | Regex enforceable | Central registry needed |
| Size | Slightly larger | Smaller |
| Interop | Locale-agnostic | Requires catalog |
| HTTP overlap | Distinct from status codes | Confusing |

> **Recommendation:**  
> Lumen should use **string codes** exclusively.  
> Numeric identifiers, if required for backward compatibility, appear only as `meta.legacy_code`.

---

`meta` — Optional and Team-Defined
----------------------------------

### Guidance

* Optional, uncontracted unless a team defines its own schema.
* **Never** include secrets, internal identifiers, hostnames, stack traces, PII, or full payloads.
* Keep small (< 4 KB) and limited to relevant machine context.
* Teams may narrow `meta` locally if they need a structured contract.

### Example of Per-API Override

components:
schemas:
LpdpError\_MCGW\_V1:
allOf:
- $ref: '#/components/schemas/LpdpError'
- type: object
properties:
meta:
$ref: '#/components/schemas/McgwErrorMetaV1'
McgwErrorMetaV1:
type: object
additionalProperties: false
description: Meta structure specific to MCGW v1 APIs.
properties:
subsystem: { type: string }
operation: { type: string }
cause:
type: string
enum: [timeout, unavailable, error, authorization, unknown]
required: [ subsystem, cause ]

---

Canonical Examples and HTTP Status Mapping
------------------------------------------

| **Use Case** | **Status** | **Example Summary** |
| --- | --- | --- |
| **Malformed Request** | **400 Bad Request** | Invalid JSON or schema syntax. |
| **Unauthorized / Forbidden** | **401 / 403** | Missing credentials / insufficient rights. |
| **Validation Errors** | **422 Unprocessable Entity** | Field-level or semantic violations. |
| **Resource Not Found** | **404 Not Found** | Composite key absent. |
| **Ambiguous Match / Conflict** | **409 Conflict** | Multiple resource matches or conflicting state. |
| **Dependency Failure** | **424 Failed Dependency** (alt 502/503/504) | Downstream timeout / failure. |
| **Internal Failure / Unknown Cause** | **500 Internal Server Error** | Unexpected system condition. |

---

### (a) Validation Error — 422

{
"title": "Validation failed",
"errors": [
{ "code": "string.min", "message": "[name] must be at least 3 characters.", "meta": { "min": 3, "actual": 2 } },
{ "code": "cidr.invalid", "message": "[ip\_address] must be a valid IPv4 CIDR.", "meta": { "expected": "IPv4 CIDR, e.g., 192.168.1.0/24" } }
]
}

### (b) Dependency Failure — 503

{
"title": "Backend dependency failed",
"errors": [
{
"code": "dependency.failed",
"message": "VRF service did not respond.",
"meta": { "subsystem": "VRFService", "operation": "lookupVrf", "resource": "vrf", "cause": "timeout" }
}
]
}

### (c) Composite Key Not Found — 404

{
"title": "VRF not found",
"errors": [
{ "code": "resource.not\_found", "message": "No VRF matched the provided name and customer.", "meta": { "resource": "vrf" } }
]
}

### (d) Ambiguous Resource — 409

{
"title": "VRF selection is ambiguous",
"errors": [
{ "code": "resource.ambiguous", "message": "More than one VRF matches the provided criteria.", "meta": { "resource": "vrf", "candidates": 3 } }
]
}

### (e) Unknown Failure — 500

{
"title": "Unable to fetch VRF information",
"errors": [
{ "code": "resource.lookup\_failed", "message": "VRF information could not be retrieved.", "meta": { "subsystem": "VRFService", "cause": "unknown" } }
]
}

### (f) Auth Failures — 401 / 403

{
"title": "Unauthorized",
"errors": [ { "code": "auth.missing", "message": "Authorization header is required." } ]
}
{
"title": "Forbidden",
"errors": [ { "code": "auth.forbidden", "message": "Caller not permitted to perform this operation." } ]
}

### (g) Malformed Request — 400

{
"title": "Malformed request",
"errors": [ { "code": "request.invalid\_json", "message": "Request body is not valid JSON." } ]
}

---

Governance Rules
----------------

1. All 4xx / 5xx responses → `application/problem+json`.
2. Each error → `code` and `message`.
3. `code` → **string** form only; numeric legacy values in `meta.legacy_code`.
4. `meta` → optional; must exclude sensitive or internal data.
5. Per-API teams may narrow `meta` via local schemas without breaking the global profile.

---

### Summary

| **Field** | **Required** | **Purpose** |
| --- | --- | --- |
| `title` | ✅ | Human-readable summary |
| `errors` | ✅ | Array of issues |
| `code` | ✅ | String identifier (machine key) |
| `message` | ✅ | Human explanation |
| `meta` | ⚙️ | Optional contextual data |
| `detail`, `instance`, `type` | ⚙️ | Optional per RFC |
| `status`, `trace_id` | ❌ | Omitted |
| Rate-limit info | — | Conveyed via headers only |