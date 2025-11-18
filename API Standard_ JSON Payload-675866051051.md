# API Standard: JSON Payload

> Confluence Page ID: 675866051051, Version: 1

none

### **Overview**

All Lumen APIs **must use JSON** as the canonical message format for request and response bodies.  
From a governance standpoint, this ensures predictable interoperability across platforms, easier validation using OpenAPI 3.0.3, and forward-compatible payload design.

To achieve this, all JSON payloads must follow the structural, naming, and encoding rules defined in [RFC 7159 (JSON)](https://datatracker.ietf.org/doc/html/rfc7159) and [RFC 7493 (I-JSON)](https://datatracker.ietf.org/doc/html/rfc7493).  
The goal is to enforce **clarity, safety, and uniformity** across all APIs regardless of team or domain.

---

### **Governance Principles**

| **Governance Objective** | **Policy Requirement** | **Rationale** |
| --- | --- | --- |
| *Standardized Format* | All request and response bodies **MUST** be valid JSON objects. | Guarantees structured, parsable data with clear schema evolution paths. |
| *I-JSON Compliance* | JSON payloads **MUST** conform to the I-JSON profile (RFC 7493). | Ensures consistent behavior across clients, SDKs, and backend systems. |
| *Encoding* | All JSON content **MUST** use UTF-8 encoding. | Avoids encoding ambiguity and corruption across gateways. |
| *Uniqueness of Keys* | Each JSON object **MUST** contain **unique property names**. | Prevents parser ambiguity and security issues (duplicate key override). |
| *Content Type* | Media type **MUST** be declared as `application/json`. | Enables correct content negotiation and API discovery. |
| *Extensibility* | Payloads **MUST** support additive evolution (new fields tolerated). | Allows forward-compatible releases without breaking clients. |

---

### **Structural Rules**

#### **Nested Structures**

**SHOULD** prefer nested, well-scoped objects over flattened key sets.  
This improves logical grouping, maintainability, and future extensibility.

✅ **Recommended**

```
{
  "customer_id": "C123",
  "billing_address": {
    "line1": "100 Main St",
    "city": "Denver",
    "postal_code": "80202"
  }
}
```

❌ **Not Recommended**

```
{
  "customer_id": "C123",
  "billing_address_line1": "100 Main St",
  "billing_address_city": "Denver",
  "billing_address_postal_code": "80202"
}
```

---

#### **Field Naming Convention**

Field names **MUST** follow **lower snake\_case**, matching regex:

```
^[a-z][a-z0-9_]*$
```

Rules:

* First character must be lowercase.
* Use underscores to separate words.
* No hyphens, camelCase, or PascalCase.
* ASCII-only.

✅ **Recommended** → `customer_name`, `invoice_number`, `created_at`  
❌ **Not Recommended** → `salesOrderId`, `CustomerName`, `sales-order-id`

---

#### **Array Naming Convention**

Array properties **MUST** use plural names to signal multiplicity.  
Empty arrays must be represented as `[]`, *not* `null`.

✅ **Recommended**

```
{
  "line_items": []
}
```

❌ **Not Recommended**

```
{
  "line_items": null
}
```

---

### **I-JSON Compliance Checklist**

| **Category** | **Requirement** | **Description** |
| --- | --- | --- |
| **Encoding** | UTF-8 | Required for all JSON payloads. |
| **Duplicate Keys** | Forbidden | Parsers must reject payloads where a key appears more than once in the same object. |
| **Numbers** | Finite, safe range only | No `NaN`, `Infinity`, or numbers > 2⁵³ − 1. |
| **Strings** | Valid Unicode scalars only | No control or invalid Unicode characters. |
| **Top-Level Type** | Object only | Arrays, primitives, or raw strings as top-level payloads are not permitted. |
| **Date/Time** | RFC 3339 UTC format | Example: `"2025-11-11T18:30:00Z"` |

---

### **Example — Valid Payload**

```
{
  "order_id": "ORD_9876",
  "customer": {
    "customer_id": "C12345",
    "name": "Acme Corp"
  },
  "line_items": [
    {
      "item_id": "I567",
      "quantity": 3
    }
  ],
  "total_amount": 1500,
  "currency": "USD",
  "created_at": "2025-11-11T18:30:00Z"
}
```

---

### **Example — Invalid Payloads**

**Duplicate Keys**

```
{
  "name": "Acme",
  "name": "Acme Inc"
}
```

**Null Array**

```
{
  "line_items": null
}
```

**Unsafe Number**

```
{
  "amount": 99999999999999999
}
```

---

### **Governance Enforcement**

* **Spectral Linting Rules:**  
  Validate UTF-8 compliance, key uniqueness, and snake\_case fields.  
  Disallow null arrays and duplicate keys.
* **Gateway Validation:**  
  Reject invalid or non-I-JSON payloads at request-time.
* **Schema Linting in CI:**  
  Automated pipelines must validate sample payloads and OpenAPI schemas for I-JSON compliance.
* **Documentation Consistency:**  
  All examples in the Developer Portal must use valid, lint-clean JSON payloads.

---

### **Summary**

JSON payload hygiene directly determines API quality and interoperability.  
Lumen APIs **must** adhere to I-JSON constraints, snake\_case naming, predictable array behavior, and consistent UTF-8 encoding.  
This ensures that payloads are **machine-safe, forward-compatible, and consistent** across every domain and consumer.