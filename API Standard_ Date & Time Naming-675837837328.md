# API Standard: Date & Time Naming

> Confluence Page ID: 675837837328, Version: 2

none

### **1. Purpose & Scope**

This standard defines how **date and date-time fields** must be named, structured, and represented in API payloads.  
Consistent date naming and typing improves discoverability, reduces ambiguity, and enables automated validation through governance tools such as **Spectral**.

This convention applies to:

* All API resource representations (request and response payloads)
* Both system metadata (e.g., auditing fields) and business fields (e.g., expiration, activation)
* All date and date-time fields defined in OpenAPI specifications

---

### **2. Core Rules**

#### **2.1 Field Naming Suffixes**

| **Suffix** | **Meaning** | **OpenAPI Format** | **Example** |
| --- | --- | --- | --- |
| `_at` | Timestamp with time component (RFC 3339 `date-time`) | `format: date-time` | `created_at`, `updated_at`, `starts_at` |
| `_on` | Calendar date only (RFC 3339 `date`) | `format: date` | `effective_on`, `expires_on` |
| `*_from` / `*_until` | Time or date ranges | `format: date-time` or `date` | `valid_from`, `valid_until` |

> ⚠️ The OpenAPI schema **MUST** align the suffix to its proper format:  
> `_at → format: date-time`, `_on → format: date`.

---

### **2.2 State vs Schedule Semantics**

Use timestamp names that clearly express **intent** — whether the value represents a **state change** (what has happened) or a **scheduled event** (what will happen).

| **Category** | **Description** | **Examples** | **Verb Tense** | **Mutability** |
| --- | --- | --- | --- | --- |
| **State-based** | Reflect past events that changed system state. Used for audit, versioning, or ordering. | `created_at`, `updated_at`, `deleted_at`, `completed_at` | Past | Immutable |
| **Schedule-based** | Define future or ongoing timeframes within a business process. | `starts_at`, `ends_at`, `expires_at`, `effective_on`, `renews_on` | Present / Future | Mutable |
| **Range-based** | Define valid windows for temporal entities. | `valid_from`, `valid_until` | Mixed | Usually Mutable |

> 💬 **Clarification:**
>
> * *State fields* answer “**When did this happen?**”
> * *Schedule fields* answer “**When will or should this occur?**”

---

### **2.3 Examples**

#### ✅ **Recommended**

{
"created\_at": "2025-10-12T15:45:33Z",
"updated\_at": "2025-10-15T19:02:11Z",
"starts\_at": "2025-11-01T00:00:00Z",
"ends\_at": "2025-11-30T23:59:59Z",
"valid\_from": "2025-11-01T00:00:00Z",
"valid\_until": "2026-01-01T00:00:00Z"
}

#### ❌ **Not Recommended**

{
"created": "2025-10-12",
"modification\_date": "2025-10-15T19:02:11Z",
"start\_date": "2025-11-01",
"expire\_at": "2025-11-30"
}

**Issues:**

* Mixed and inconsistent suffixes (`_date`, `_at`)
* Some fields missing time component where expected
* Ambiguous intent (is `expire_at` a schedule or state?)

---

### **3. Governance & Enforcement (Spectral Rule)**

The following Spectral rule enforces this convention during API design-time validation.

#### **Rule ID:** `lumen-date-field-naming`

rules:
lumen-date-field-naming:
description: "Date/time fields must use '\_at' for RFC 3339 date-time and '\_on' for RFC 3339 date."
message: >-
"{{property}} field name does not follow Lumen Date/Time naming convention.
Use '\_at' for RFC 3339 date-time, '\_on' for RFC 3339 date.
For ranges, use \*\_from / \*\_until."
given: "$.components.schemas.\*.properties.\*"
severity: warn
then:
- function: pattern
field: "@key"
functionOptions:
match: ".\*(\_at|\_on|\_from|\_until)$"
- function: truthy
field: "format"
- function: enumeration
field: "format"
functionOptions:
values:
- date
- date-time

✅ **What it checks:**

* Ensures all date-related fields end with `_at`, `_on`, `_from`, or `_until`.
* Validates that `_at` fields use `format: date-time`.
* Validates that `_on` fields use `format: date`.
* Flags inconsistent or ambiguous names (`created`, `start_date`, `expire_at`).

---

### **4. Automation Notes**

* Integrate this rule into the **Spectral ruleset** used by the API linter in the CI pipeline.
* Apply it to all repositories that define OpenAPI 3.x specifications.
* Violations are classified as **warnings** (initial rollout) and will be escalated to **errors** once adoption reaches >80%.

---

### **5. Related Standards**

* [RFC 3339: Date and Time on the Internet](https://www.rfc-editor.org/rfc/rfc3339)