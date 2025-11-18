# API Standard: GET Method

> Confluence Page ID: 675848454259, Version: 4

none

### **Purpose**

The `GET` method retrieves a resource or a collection of resources.  
It **MUST NOT** change server state and **MUST** be safe and idempotent as defined in [RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110).

This page defines how to design, document, and implement `GET` operations across Lumen APIs to ensure predictable, cache-friendly, and standards-compliant behavior.

---

### **Semantics**

| Property | Requirement |
| --- | --- |
| **Safe** | ✅ YES, never modifies data or triggers side effects. |
| **Idempotent** | ✅ YES, repeating the same GET yields the same result (unless data has changed). |
| **Request Body** | ❌ NOT ALLOWED. |
| **Response Body** | ✅ REQUIRED for `200 OK`, MUST represent the requested resource(s). |
| **Caching** | ✅ Strongly RECOMMENDED, use `ETag` and `Cache-Control`. |

---

### **URI Design**

**Examples**

* Single resource: `/v1/customers/{customer_id}`
* Collection: `/v1/customers`
* Filtered collection: `/v1/customers?status=active&country=US`

**MUST**

* Use plural nouns for collections.
* Avoid action verbs (`/getCustomer`) — the HTTP verb already defines intent.
* Preserve canonical noun hierarchy (**no verbs, no RPC style**).

---

### **Singleton Resources**

**Definition**  
A *singleton resource* represents a unique conceptual instance of a type that exists once per context (e.g., current user profile, organization settings, service status).  
Singletons do not require an identifier or belong to a collection.

**Examples**

```
GET /v1/profile                → current user profile
GET /v1/organization/settings  → organization configuration
GET /v1/configuration          → global configuration
GET /v1/status                 → system status or health
```

**Design Guidance**

| Rule | Description |
| --- | --- |
| **MUST** | Treat singletons as first-class nouns (`/profile`, `/settings`, `/status`). |
| **MUST NOT** | Nest singleton resources under collections **if they represent contextual or global concepts** (e.g., avoid `/users/{id}/profile` for the authenticated user’s profile). Such resources have a single instance determined by the caller’s identity or environment — not by path variables — and therefore **belong at the root level** (e.g., `/profile`). |
| **MAY** | Nest singleton resources **only when they are scoped to a parent resource** (e.g., `/organizations/{orgId}/settings` — one settings resource per organization). |
| **MUST** | Support `GET` for retrieval and `PATCH` for updates (if mutable). Never use `POST` or `DELETE`. |
| **MUST** | Return `200 OK` with a body when available; `404 Not Found` if temporarily unavailable. |
| **SHOULD** | Include `ETag` and support `If-None-Match` for cache validation. |
| **MUST NOT** | Return `204` for existing singletons — always return a representation unless explicitly empty. |

**Example**

```
GET /v1/profile
If-None-Match: "v12"

HTTP/1.1 200 OK
ETag: "v13"
Cache-Control: private, max-age=60
{
  "id": "u123",
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

```
GET /v1/profile
If-None-Match: "v13"

HTTP/1.1 304 Not Modified
ETag: "v13"
```

**Governance Note**  
Singletons must be documented as distinct resources (`/profile`, `/settings`, `/status`) and enforced through Spectral rules to verify `ETag` and `If-None-Match` support.

---

### **Response Structure**

Every successful `GET` MUST return a JSON object.

#### **Single Resource**

```
{
  "id": "c123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "created_at": "2025-10-20T12:34:56Z"
}
```

#### **Collection**

```
{
  "data": [
    { "id": "c123", "name": "Jane Doe" },
    { "id": "c124", "name": "John Smith" }
  ],
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 245
  }
}
```

**MUST**

* Wrap collections in a `data` array to allow metadata (`pagination`, `links`) at the top level.
* Use consistent pagination keys (`limit`, `offset`, `total`).

---

### **Headers and Conditional Requests**

#### **ETag and Cache Validation**

| Header | Usage | Example |
| --- | --- | --- |
| `ETag` | Server-generated version tag for the resource | `ETag: "v7a2"` |
| `If-None-Match` | Client cache validation header | `If-None-Match: "v7a2"` |
| Response on match | `304 Not Modified` (no body) |  |

**MUST**

* Return `ETag` on all cacheable responses.
* Support `If-None-Match` on all `GET` endpoints.
* Return `304 Not Modified` if ETag matches.

**SHOULD**

* Add `Cache-Control` (e.g., `private, max-age=60`) to define freshness.
* Include `Last-Modified` when available for weak validation.

---

### **Query Parameters and Filtering**

| Principle | Guidance |
| --- | --- |
| **Predictability** | Parameter names must be consistent across APIs (`status`, `sort`, `limit`, `offset`). |
| **Multiple Filters** | Use `&` separated query pairs. |
| **Sorting** | `?sort=+name` ascending, `?sort=-created_at` descending. |
| **Pagination** | Refer to pagination standards. |
| **Empty Results** | Return `200 OK` with an empty `data` array, not `404`. |
| **Invalid Filters** | Return `400 Bad Request` with Problem Details. |

---

### **Status Codes**

| Code | Meaning | Applies To | Usage Notes |
| --- | --- | --- | --- |
| **200 OK** | Successful retrieval | ✅ Single / ✅ Collection | Always return for successful GETs; include response body. |
| **304 Not Modified** | Resource unchanged since client cache | ✅ Single / ✅ Collection | Used with `If-None-Match`, no body. |
| **400 Bad Request** | Invalid query or parameter | ✅ Single / ✅ Collection | Syntactic or invalid filter errors. |
| **401 Unauthorized** | Authentication required or invalid | ✅ Both |  |
| **403 Forbidden** | Authorized but lacks permission | ✅ Both |  |
| **404 Not Found** | Resource not found | ✅ Single | Never use for empty collections. |
| **405 Method Not Allowed** | HTTP verb not supported | ✅ Both |  |
| **422 Unprocessable Content** | Valid syntax but invalid semantic input | ✅ Both |  |
| **500 Internal Server Error** | Unexpected server failure | ✅ Both | No stack traces exposed. |

---

### **Example Request / Response Set**

**Request**

```
GET /v1/customers/123
If-None-Match: "v5"
Accept: application/json
```

**Response (Resource Updated)**

```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v6"
Cache-Control: private, max-age=60
{
 "id": "123",
 "name": "Jane Doe",
 "email": "jane@example.com"
}
```

**Response (Not Modified)**

```
HTTP/1.1 304 Not Modified
ETag: "v6"
```

---

### **OAS Authoring Requirements**

```
paths:
  /v1/customers/{id}:
    get:
      summary: Retrieve customer by ID
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: header
          name: If-None-Match
          schema: { type: string }
      responses:
        '200':
          description: Successful response
          headers:
            ETag:
              description: Entity tag for versioning
              schema: { type: string }
        '304':
          description: Not Modified
        '400': { description: Invalid request }
        '404': { description: Not found }
```

**MUST**

* Include `ETag` in `200` response headers.
* Include `If-None-Match` parameter.
* Restrict status codes to the approved matrix.
* Return `application/json`.

---

### **Governance Checklist**

| Check | Validation |
| --- | --- |
| ✅ `GET` defined as safe and idempotent | No side effects |
| ✅ `ETag` header returned | Required for cacheable responses |
| ✅ `If-None-Match` supported | Enables `304 Not Modified` |
| ✅ No request body allowed | Specification enforces it |
| ✅ Proper status codes used | 200/304/400/401/403/404/405/422/500 only |
| ✅ OAS validated via Spectral rules | Governance automation ready |

---

### **Developer Summary**

* Use `GET` exclusively for read-only operations.
* Never require or process a body in `GET`.
* Always include `ETag` and handle `If-None-Match` for conditional caching.
* Return `200 OK` for success, `304` for cache hits, `404` only for missing single resources.
* Treat collection queries as successful even when empty.
* Keep response structure predictable (`data`, `pagination`).
* Follow the approved status-code matrix and Problem Details format.