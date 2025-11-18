# API Standard: POST Method

> Confluence Page ID: 675852386305, Version: 3

none

**Purpose**
-----------

The `POST` method creates new resources or performs non-CRUD actions on existing resources.  
It is **not inherently idempotent** and should be used when neither `PUT` nor `PATCH` semantics apply.

This page defines the required and recommended practices for all `POST` operations across Lumen APIs, ensuring consistency, predictability, and safe retries through **idempotency keys** and **standardized responses**.

---

**Semantics**
-------------

| Property | Requirement |
| --- | --- |
| **Safe** | ❌ No, may modify server state. |
| **Idempotent** | ⚠ No by default; **MUST** use `Idempotency-Key` if retries are expected. |
| **Typical Use** | Resource creation or triggering an action. |
| **Request Body** | ✅ Required, defines input parameters or resource representation. |
| **Response Body** | ✅ Required, returns the created resource or operation result. |

---

**When to Use** `POST`
----------------------

| Use Case | Example | Required Behavior |
| --- | --- | --- |
| **Create a new resource** | `POST /v1/orders` | Returns `201 Created` + `Location` header for the new resource. |
| **Create multiple resources (batch / composite operation)** | POST /v1/customers/batch | Performs creation of multiple items in one request. Returns **207 Multi-Status** if results are mixed, or **202 Accepted** if processed asynchronously via a job. Batch requests SHOULD complete within the synchronous timeout budget (≤ 30 s p95 / 60 s absolute) or follow the **LRO pattern**. |
| **Trigger an action on a resource** | `POST /v1/invoices/{id}/void` | Returns `200 OK` or `202 Accepted` depending on sync vs. async. |
| **Start an asynchronous or workflow operation** | `POST /v1/network/connections/provision` | Returns `202 Accepted` + operation URL for status tracking. |
| **Perform search or query with complex filters** (non-CRUD) | `POST /v1/orders/search` | Allowed when query parameters are too complex for GET; must be safe and side-effect free. |
| **Submit sensitive criteria or large query payloads (PII, account numbers, tokens, advanced filters)** | `POST /v1/customers/search` with body `{"email":"…","dob":"…"}` | Use `POST` with a **request body** instead of putting sensitive data in the URL. **MUST** be side-effect free. **MUST** set `Cache-Control: no-store`. **MUST NOT** log or echo sensitive fields. **MUST NOT** include PII or secrets in the path or query string. **SHOULD** return `413 Payload Too Large` for oversized bodies and document limits. **SHOULD** include `Vary: Authorization` for authenticated results. |

### **Non-Applicable Cases**

| Rule | Description |
| --- | --- |
| **MUST NOT** | Use `POST` for **singleton resources** (e.g., `/profile`, `/configuration`, `/status`). These resources are pre-defined by the system and not client-created. Use `GET` to retrieve and `PATCH` to update them. |

#### **Why the Sensitive-Data Rule Exists**

* URLs appear in logs, browser history, and referrers; request bodies do not.
* Shared or transparent caches may store GET responses; using POST + `Cache-Control: no-store` minimizes risk.
* Request bodies allow schema-driven validation and redaction (`x-lumen-sensitive: true` in OAS).

---

**Creation Semantics**
----------------------

### **Resource Creation**

* **MUST** create new resource(s) under the collection URI.  
  Example:  
  `POST /v1/customers` → creates a customer resource.
* **MUST** return:

  + `201 Created`
  + A `Location` header with the canonical URI of the new resource.
  + The newly created resource in the response body.
* **MUST NOT** use `POST` to **replace** an existing resource; use `PUT` instead.
* **MAY** return `202 Accepted` for **asynchronous creation**
* **MAY** return `207 Multi-Status` if a **batch request** partially succeeds.

**Example:**

```
POST /v1/customers
Content-Type: application/json
{
  "name": "Jane Doe",
  "email": "jane@example.com"
}

HTTP/1.1 201 Created
Location: /v1/customers/c123
Content-Type: application/json
{
  "id": "c123",
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

---

**Action Semantics**
--------------------

When performing an action on an existing resource that does not map to a CRUD operation:

| Rule | Description |
| --- | --- |
| **Pattern** | `POST /{resource}/{id}/{action}` (e.g., `/invoices/{id}/approve`). |
| **Response** | `200 OK` for synchronous result, `202 Accepted` for async. |
| **Body** | MAY contain parameters relevant to the action. |
| **Idempotency** | MUST support `Idempotency-Key` to handle retries safely. |

**Example:**

```
POST /v1/invoices/123/void
Idempotency-Key: 4af7e7d9-91a6-4823-a1c0-8d112c884cb5
HTTP/1.1 200 OK
Content-Type: application/json
{
  "id": "123",
  "status": "voided"
}
```

---

**Idempotency**
---------------

`POST` is **not idempotent by default** — but Lumen APIs must provide an idempotent experience when requests are retried.

| Header | Description |
| --- | --- |
| `Idempotency-Key` | A client-supplied opaque UUID identifying the request. |
| **Scope** | Applies to unsafe POSTs that could be retried. |
| **Server Behavior** | Deduplicate requests with the same key and identical payload; replay original result. |
| **Retention** | Store key and result for a minimum of 24 hours. |
| **Conflict Handling** | Return `409 Conflict` if the same key is reused with a different payload. |

**Example:**

```
POST /v1/payments
Idempotency-Key: 827dcf3e-44fb-4f07-94b3-6b47cf3b813d

→ 201 Created

# Retry (same key)
→ 201 Created (identical response)

# Retry (same key, different payload)
→ 409 Conflict
```

---

**Headers**
-----------

| Header | Usage |
| --- | --- |
| `Idempotency-Key` | Ensures retry-safe behavior. |
| `Location` | Points to newly created resource or operation resource. |
| `Retry-After` | Used when returning `202 Accepted` for async processing. |
| `Content-Type` | MUST be `application/json` or vendor subtype. |
| `Accept` | MUST be `application/json` (or compatible). |

---

**Status Codes**
----------------

| Code | Meaning | Applies To | Notes |
| --- | --- | --- | --- |
| **201 Created** | Resource successfully created | ✅ Resource creation | MUST include `Location` header and representation. |
| **202 Accepted** | Request accepted for asynchronous processing | ✅ LRO / async | MUST include operation URL in `Location`. |
| **207 Multi-Status** | Batch/composite request produced mixed results | ✅ Bulk / composite POSTs | Use when some items succeed and others fail, body MUST include per-item statuses and summary. |
| **200 OK** | Successful synchronous action | ✅ Resource actions | Used only for non-creation actions. |
| **400 Bad Request** | Invalid request body or parameter | ✅ Both | Syntax or schema violation. |
| **401 Unauthorized** | Authentication required | ✅ Both |  |
| **403 Forbidden** | Authenticated but lacks permission | ✅ Both |  |
| **404 Not Found** | Target resource or parent collection missing | ✅ Both |  |
| **405 Method Not Allowed** | Wrong HTTP method used | ✅ Both |  |
| **409 Conflict** | Resource already exists or idempotency key conflict | ✅ Creation / Action |  |
| **422 Unprocessable Content** | Semantic validation error | ✅ Both | Schema valid but business rule failure. |
| **500 Internal Server Error** | Unexpected server failure | ✅ Both | No internal details exposed. |

---

**OAS Authoring Requirements**
------------------------------

When documenting `POST` operations in OpenAPI:

```
paths:
  /v1/orders:
    post:
      summary: Create an order
      parameters:
        - in: header
          name: Idempotency-Key
          schema: { type: string, maxLength: 255 }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/OrderCreate' }
      responses:
        '201':
          description: Created
          headers:
            Location:
              schema: { type: string, format: uri }
        '202': { description: Accepted (async creation) }
        '207': { description: Partial success (batch results) }
        '400': { description: Invalid request }
        '409': { description: Conflict (duplicate or idempotency) }
        '422': { description: Validation failed }
```

**MUST**

* Include `Idempotency-Key` header definition.
* Include `Location` header for `201` or `202` responses.
* Constrain response codes to the approved matrix.
* Document LRO patterns (`202 + Location`) when applicable.
* Reference `207` for batch/composite POSTs if supported.

---

**Governance Validation Checklist**
-----------------------------------

| Check | Expectation |
| --- | --- |
| ✅ Correct verb usage | `POST` used only for create or action semantics |
| ✅ Location header present | For `201` and `202` responses |
| ✅ Idempotency supported | `Idempotency-Key` required for unsafe POSTs |
| ✅ Proper status codes | Matches approved matrix |
| ✅ Async workflows standardized | `202 + Location` pattern used |
| ✅ OAS validated via Spectral | Governance enforcement ready |

---

**Developer Summary**
---------------------

* Use `POST` to **create** new resources or **trigger** actions — never for updates.
* Always return `201 Created` with a `Location` header for creations.
* Use `Idempotency-Key` to protect clients from duplicate processing.
* For asynchronous operations, return `202 Accepted` and an **operation URI**.
* For partial batch success, return `207 Multi-Status` with `results[]` and `summary`.
* Adhere strictly to the **approved status-code matrix**.
* Document every `POST` in OpenAPI with consistent headers and response structure.