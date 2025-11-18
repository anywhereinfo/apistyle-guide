# API Standard: Delete Method

> Confluence Page ID: 675767681290, Version: 5

none

DELETE Method – Request and Response Standards
----------------------------------------------

### **Summary**

DELETE represents a request to remove the entire resource identified by the URI.  
It is *not* intended for partial removal, filtered deletion, or passing instruction payloads.

---

### **Request Rules**

| Rule | Requirement | Rationale |
| --- | --- | --- |
| **MUST NOT include a request body** | DELETE has undefined semantics for a body per RFC 9110 §9.3.5 | Ensures predictable behavior across proxies, SDKs, and gateways |
| **MUST target a single resource URI** | DELETE /{resource}/{id} | Full deletion semantics, not partial update |
| **MUST NOT support conditional filters or lists in the body** | Use PATCH or POST /{resource}:delete instead | Keeps REST semantics pure and idempotent |
| **MUST be idempotent** | Multiple identical DELETEs should have the same effect | Required for retry safety |

---

#### 🧾 **Example (Correct)**

```
DELETE /fabric/v1/connections/12345
```

**Response**

```
HTTP/1.1 204 No Content
```

---

#### 🚫 **Example (Incorrect)**

```
DELETE /mcgw/v1/routes
Content-Type: application/json

{
  "routeIds": ["r1","r2","r3"]
}
```

**Why This Is Non-Compliant:**  
This DELETE request attempts partial modification via a request body, which has no defined semantics under RFC 9110.  
Use PATCH or a POST action endpoint instead.

---

### **Response Rules**

| Status Code | Meaning | Use Case |
| --- | --- | --- |
| **204 No Content** | ✅ Successful deletion Or Resource does not exist | Resource deleted successfully Or Target resource already deleted or invalid |
| **400 Bad Request** | Invalid URI, malformed query parameter | Syntax or parameter validation failure |
| **401 Unauthorized** | Authentication required | Caller not authenticated |
| **403 Forbidden** | Insufficient permission | Caller lacks permission to delete |
| **405 Method Not Allowed** | DELETE not supported for this resource | Used when DELETE not implemented |
| **409 Conflict** | Discouraged. If 409 is possible during delete, model it as POST with cancel action. | e.g., “Resource locked” or “pending operation” |
| **422 Unprocessable Content** | Discouraged. If 422 is possible during delete, model it as POST with cancel action. | e.g., “Cannot delete active policy” — prefer PATCH or POST action |
| **500 Internal Server Error** | Platform/system failure | Server-side unexpected error |

---

### **Response Body Rules**

* **MUST NOT** include a response body for Delete.

### **Alternatives for Non-Standard Deletion Use Cases**

| Use Case | Recommended Design | Example |
| --- | --- | --- |
| Delete subset of items | PATCH with remove operation | `PATCH /mcgw/v1/routes { "remove": ["r1","r2"] }` |
| Batch or filtered deletion | POST to action endpoint | `POST /fabric/v1/connections:delete { "ids": ["c1","c2"] }` |
| Conditional deletion | POST to /{resource}:terminate with conditions | `POST /mcgw/v1/sessions:terminate { "force": true }` |

---

### **Governance Enforcement Examples**

**Rule 1 – No DELETE Request Body**

```
rules:
  lumen-no-delete-body:
    description: "DELETE requests must not define requestBody"
    given: "$.paths[*].delete"
    then:
      field: requestBody
      function: falsy
```

**Rule 2 – Valid DELETE Responses**

```
rules:
  lumen-delete-status-codes:
    description: "Allowed HTTP status codes for DELETE"
    given: "$.paths[*].delete.responses"
    then:
      function: lumen-validate-status-code
      functionOptions:
        validCodes: [204, 400, 401, 403, 404, 405, 500]
```

---

### **Summary Table**

| Category | Guideline |
| --- | --- |
| **Request** | DELETE must target a single resource and have no body |
| **Response** | Use only predefined codes (204, 400–500) |
| **Governance** | Enforced via Spectral rules for DELETE behavior and status codes |

---

### **Key Takeaways**

* DELETE is for **entire resource removal only** — not partial, conditional, or batch operations.
* No request body allowed — maintain idempotency and HTTP compliance.
* Use **PATCH** for partial deletion or **POST :action** for complex or conditional deletions.
* Responses should be predictable and minimal (typically `204 No Content`).
* All error responses should follow **RFC 7807** for consistency and automation.