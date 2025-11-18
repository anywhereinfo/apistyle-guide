# API Standard: PUT Method

> Confluence Page ID: 675851796520, Version: 3

none

**Purpose**
-----------

The `PUT` method **replaces the full representation** of a resource at its canonical URI.  
It is **idempotent** — identical requests produce the same final state.

This page defines Lumen’s standards for designing, documenting, and implementing `PUT` operations, including creation semantics, conditional updates, and approved response codes.

---

**Semantics**
-------------

| Property | Requirement |
| --- | --- |
| **Safe** | ❌ No, modifies server state. |
| **Idempotent** | ✅ Yes, repeating the same request yields the same result. |
| **Typical Use** | Full replacement of a resource. |
| **Request Body** | ✅ Required, represents the complete, updated resource. |
| **Response Body** | ✅ Required for `200 OK`, returns the latest representation. |

---

**When to Use** `PUT`
---------------------

| Use Case | Example | Required Behavior |
| --- | --- | --- |
| **Replace an existing resource completely** | `PUT /v1/customers/{id}` | Replaces the full customer object with the provided payload; returns `200 OK` and the updated resource. |
| **Create a resource at a client-known URI (rare)** | `PUT /v1/buckets/{id}` | MAY be used if the client controls ID generation. Return `201 Created` if the resource did not exist before. |
| **Update large static configurations or documents** | `PUT /v1/configurations/{id}` | Appropriate when sending an entire configuration file or template. |
| **Atomic overwrite of immutable document** | `PUT /v1/policies/{id}` | Old version replaced entirely, new ETag generated. |

---

**Singleton Resources**
-----------------------

### **Definition**

A **singleton resource** represents a unique instance that exists once per logical context (for example, `/configuration`, `/profile`, `/settings`, `/status`, `/organization/{id}/policy`).  
It is not part of a collection. A singleton **may or may not pre-exist** , if it does not, the client can create it by `PUT`ting a full representation at its canonical URI.

---

### **Guidance**

| Rule | Description |
| --- | --- |
| **MAY use PUT to create a singleton** | When the resource has a known, fixed URI and does not yet exist. `PUT` is idempotent, repeating the same request yields the same result. Return **201 Created** on first creation, **200 OK** on subsequent replacements. |
| **MUST** | Require `If-Match` on updates when the singleton already exists to prevent concurrent overwrites. |
| **MUST return 201 Created** | When the singleton did not exist previously and has now been created. |
| **MUST return 200 OK** | When replacing an existing singleton with a new representation. |
| **SHOULD include ETag** | Every GET and PUT response for a singleton must include an ETag for version control. |
| **MUST NOT** | Use PUT for actions or only for full replacement. Use PATCH for partial updates. |
| **MUST NOT** | Allow DELETE on singleton endpoints. |

---

### **Examples**

**Create (first-time PUT)**

```
PUT /v1/organization/settings
Content-Type: application/json

{
  "auto_approve": true,
  "timezone": "America/Chicago"
}

HTTP/1.1 201 Created
ETag: "v1"
Location: /v1/organization/settings
```

**Replace (subsequent update)**

```
PUT /v1/organization/settings
If-Match: "v1"
Content-Type: application/json

{
  "auto_approve": false,
  "timezone": "America/Chicago"
}

HTTP/1.1 200 OK
ETag: "v2"
```

**Conflict (stale ETag)**

```
HTTP/1.1 412 Precondition Failed
Content-Type: application/problem+json
{
  "type": "https://api.lumen.com/errors/conflict",
  "title": "Precondition Failed",
  "status": 412,
  "detail": "The resource has been modified by another client."
}
```

---

### **Governance Note**

* Singleton `PUT` operations follow the same idempotency, full-replacement, and ETag/If-Match rules as normal resources.
* Example Spectral validation:

```
rules:
  singleton-put-must-support-etag:
    given: "$.paths[?(@property.match(/^\\/v1\\/(configuration|profile|settings|status|organization\\/[^/]+\\/policy)/))].put"
    then:
      field: "responses['200'].headers.ETag"
      function: truthy
```

**Summary**

> Singleton resources can be created or replaced using PUT when they have a fixed URI and no existing representation.  
> Return 201 on first creation, 200 on subsequent updates.  
> Always include ETag and require If-Match for concurrency control.

---

**General Rules**
-----------------

| Rule | Description |
| --- | --- |
| **MUST** | Replace the **entire resource representation**; omitted fields are treated as deleted. |
| **SHOULD NOT** | Use PUT for partial updates — use PATCH instead. |
| **MUST** | Return the updated resource in the response body. |
| **MUST** | Preserve the resource identifier and canonical URI. |
| **MUST NOT** | Allow structural mutations (e.g., changing resource type or parent linkage). |
| **MUST** | Be **idempotent** — re-submitting the same payload must not create duplicates. |

---

**Conditional Requests & ETag**
-------------------------------

| Header | Usage | Example |
| --- | --- | --- |
| `ETag` | Returned with GET/PUT responses. | `ETag: "v12"` |
| `If-Match` | Required header for optimistic concurrency. | `If-Match: "v12"` |

**Rules**

* **MUST** return a new ETag on every successful modification.
* **SHOULD** reject updates without If-Match when concurrent edits are possible.
* **MUST** return 412 Precondition Failed if If-Match does not match.
* **MUST NOT** accept stale updates.

---

**Example**
-----------

```
PUT /v1/customers/c123
If-Match: "v4"
Content-Type: application/json

{
  "id": "c123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "status": "active"
}
```

```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v5"
Cache-Control: private, max-age=60

{
  "id": "c123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "status": "active"
}
```

---

**Status Codes**
----------------

| Code | Meaning | Applies To | Notes |
| --- | --- | --- | --- |
| **200 OK** | Resource successfully replaced | ✅ Existing resource | MUST include updated representation. |
| **201 Created** | Resource newly created at client-specified URI or singleton first creation | ✅ Creation | MUST include Location header. |
| **204 No Content** | Update succeeded, no body returned | ⚠ Optional | Only if representation unchanged. |
| **400 Bad Request** | Invalid or missing fields | ✅ Both | Schema/validation errors. |
| **401 Unauthorized** | Authentication required | ✅ Both |  |
| **403 Forbidden** | Authenticated but not allowed | ✅ Both |  |
| **404 Not Found** | Resource does not exist | ✅ Both |  |
| **405 Method Not Allowed** | Wrong HTTP verb | ✅ Both |  |
| **409 Conflict** | Version/state conflict | ✅ Both | For concurrent updates or integrity violations. |
| **412 Precondition Failed** | If-Match mismatch | ✅ Both | Stale ETag. |
| **422 Unprocessable Content** | Semantically invalid | ✅ Both | Valid JSON, violates business rule. |
| **500 Internal Server Error** | Unexpected server failure | ✅ Both | No internal details exposed. |

---

**Headers**
-----------

| Header | Usage |
| --- | --- |
| `If-Match` | Required for conditional updates. |
| `ETag` | Returned in every successful PUT. |
| `Cache-Control` | MUST specify `private` or `no-store`. |
| `Content-Type` | MUST be `application/json`. |
| `Accept` | MUST be `application/json` or compatible. |

---

**OAS Authoring Requirements**
------------------------------

```
paths:
  /v1/customers/{id}:
    put:
      summary: Replace a customer resource
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
          application/json:
            schema: { $ref: '#/components/schemas/Customer' }
      responses:
        '200':
          description: Successful replacement
          headers:
            ETag:
              description: Entity tag for versioning
              schema: { type: string }
        '201': { description: Created (first-time PUT) }
        '412': { description: Precondition failed (stale ETag) }
        '400': { description: Invalid request }
        '404': { description: Not found }
```

**MUST**

* Define `If-Match` and `ETag` headers.
* Restrict status codes to the approved matrix.
* Use `application/json`.

---

**Governance Checklist**
------------------------

| Check | Validation |
| --- | --- |
| ✅ Idempotency guaranteed | PUT deterministically replaces the same resource |
| ✅ Full resource in body | Required |
| ✅ Returns updated representation | Required |
| ✅ ETag / If-Match implemented | Required for concurrency control |
| ✅ Proper status codes used | 200/201/204/400/401/403/404/405/409/412/422/500 |
| ✅ Singleton creation allowed via PUT | 201 Created on first creation |
| ✅ OAS validated via Spectral | Governance automation ready |

---

**Developer Summary**
---------------------

* Use **PUT** for full resource replacement or idempotent creation at a known URI.
* Include the entire object; missing fields imply deletion.
* Support **ETag** and **If-Match** for concurrency safety.
* Return `201 Created` on first creation, `200 OK` on updates.
* Ensure idempotency — repeated identical requests yield identical state.
* Use **PATCH** for partial updates.
* Follow the approved status-code matrix and RFC 7807 error model.
* For **singleton resources**, PUT can create or replace them at a fixed URI; always include ETag and If-Match.

---

**Conditional Update Decision Tree**
------------------------------------

```
                   ┌────────────────────────────┐
                   │      Is this a change?      │
                   └──────────────┬──────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │                         │
               No → Use GET              Yes → Modify state
                                              │
                                  ┌───────────┴───────────┐
                                  │                       │
                      Is it a new resource?         Existing resource?
                                  │                       │
                    ┌─────────────┘                       └──────────────┐
                    │                                                  │
           Use POST (create)                             Replace or update?
                                                                   │
                                           ┌───────────────────────┴────────────────────────┐
                                           │                                              │
                                  Replace full representation?                Modify partial fields only?
                                           │                                              │
                                 ┌─────────┘                                              └──────────┐
                                 │                                                             │
                        ✅ Use PUT with If-Match                                 ✅ Use PATCH with If-Match
```