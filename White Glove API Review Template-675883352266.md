# White Glove API Review Template

> Confluence Page ID: 675883352266, Version: 7

The White Glove Approach: Definition and Expectations
=====================================================

The **White Glove Approach** signifies the **high-touch, proactive support and partnership** provided by the API Governance team to guide product teams through the adoption of foundational API standards, principles, and canonical models.

### Definition

The White Glove API Review is a **mandatory, deep-dive architectural evaluation** that ensures alignment with the platform's core standards. It is understood that due to real-world constraints, **technical debt** may exist. The primary objective is to make this debt **transparent, quantifiable, and formally managed** rather than blocking the API's release.

### Expectations

The expectation for any API undergoing a White Glove review is to achieve the **highest possible degree of conformance** while ensuring any non-conforming elements are explicitly accepted, prioritized, and scheduled for resolution.

|  |  |
| --- | --- |
| **Expectation Area** | **White Glove Standard (Focus)** |
| **Model Alignment** | Full usage of canonical resources and fragments; minimal exposure of non-canonical IDs. |
| **Style Conformance** | Adherence to all **mandatory** style guide rules; **Optional** deviations are documented and approved. |
| **Architectural Debt** | All identified technical debt **must be clearly outlined** in the **Summary of Gaps** . |
| **Remediation Plan** | A clear, committed **Remediation Roadmap** must be established and owned by the Product Team, detailing the plan, timeline, and resources to address all debt. |

> **Key Principle:** The White Glove Review serves as the **formal contract** for managing technical debt. An API may proceed with identified gaps, but only if the **Remediation Roadmap is agreed upon** and the **Final Architecture Rating**  is acceptable to the architectural stakeholders.

**Project Overview**
====================

| Field | Description |
| --- | --- |
| Project / Product Name |  |
| Domain / BU |  |
| Architect Reviewer(s) |  |
| Date of Review |  |
| Primary Stakeholders |  |
| API(s) Under Review |  |
| High-Level Business Capability |  |
| Deployment Model | (*internal / external / partner / wholesale*) |

---

**Canonical Model Alignment**
=============================

This section validates whether the API’s data model aligns with Lumen’s **canonical domain model**, including **resources, subresources, and reusable schema fragments**.

---

**Resource Inventory**
----------------------

| Resource | Type | Description | Canonical? | Notes |
| --- | --- | --- | --- | --- |

**Resource Type Options:**

* CRUD Resource
* Workflow Resource
* Configuration Resource
* Operational Resource
* Composite Resource (aggregates other resources)

---

**Reusable Schema Fragments Inventory**
---------------------------------------

List all reusable fragments defined or referenced by the API.

Examples of fragments:

* address
* money
* port-reference
* customer-reference
* name
* error-detail
* version-info
* metadata fragments
* pagination fragments
* resource\_links

| Fragment Name | Description | Canonical Source? | Reused Across APIs? | Notes |
| --- | --- | --- | --- | --- |

This ensures consistency AND reveals fragmentation (“address1/address2 vs line1/line2”).

---

**Canonical Alignment Evaluation**
----------------------------------

Evaluate the model holistically, not just the top-level resources.

### **Resource-Level Checks**

* Does each resource map to a canonical domain concept?
* Do resource names follow the uniform taxonomy?
* Are relationships represented as resources (nouns) vs verbs (RPC)?
* Are resource boundaries clear (no overlapping responsibilities)?

### **Reusable Fragment Checks (NEW)**

* Are reusable canonical fragments used instead of redefining schemas?
* Does the API introduce duplicate fragments that already exist in the platform?
* Are fragment names consistent with canonical naming conventions?
* Are fragments correctly versioned or tied to product namespace?

### **Identifier Checks**

* Are IDs opaque, stable, non-sequential, URL-safe?
* Does the resource use standardized `<resource>_id` naming?
* Are internal system IDs leaked to clients?

### **Composition & Structure Checks**

* Does the API follow canonical composition rules (nested vs reference)?
* Are subresources modeled consistently (/{resource}/{id}/{subresource})?
* Are reusable components appropriately referenced via `$ref`?
* Are optional and required fields consistent across services?

---

**Canonical Model Gaps & Recommendations**
------------------------------------------

Document discovered issues:

| Gap | Impact | Recommendation | Priority |
| --- | --- | --- | --- |

Examples:

* Duplicate address schema detected
* Non-canonical ID format used
* Workflow resource incorrectly modeled as CRUD
* Missing canonical fragments for links, pagination, metadata
* Internal BSS/OSS IDs exposed

---

**API Style Guide Conformance**
===============================

Evaluate the API against the official Lumen API Style Guide across the following areas:

### **Style Guide Areas**

* URI Design & Structure
* Resource Modeling & Taxonomy
* Naming Conventions
* Versioning Strategy
* Request & Response Schemas
* Identifiers & Resource IDs
* Error Model & Problem Details
* Pagination & Filtering
* Asynchronous Operations Pattern
* Header Standards (Required & Optional)
* Authentication & Authorization Model
* Idempotency
* HTTP Methods & Status Codes
* Content Type & Encoding Standards
* Security, PII, and Data Handling Requirements
* Rate Limiting & Throttling Metadata
* Observability (Logging, Tracing, Correlation IDs)
* Deprecation & Backwards Compatibility Rules
* Governance & Automation Integration
* API Version Lifecycle Management
* Bulk Operations & File Transfer Patterns (if applicable)
* Partner, Wholesale, and Federation-Specific Guidelines

### **Alignment Summary**

* Fully Aligned:
* Partially Aligned:
* Not Aligned:
* Notes & Gaps:

---

**Alignment with Cloud-First API Principles**
=============================================

For each principle, mark:  
✔ Fully Aligned ~ Partially ✘ Not Aligned

### **Resource-Oriented Interface**

### **Desired-State & Outcome-Based Design**

### **Asynchronous & Operation-Tracked**

### **Product-Scoped Versioning**

### **Entitlement-Aware & Monetizable**

### **Self-Service & Declarative Lifecycle**

### **Uniform Taxonomy & Namespacing**

### **Strong Identity & Policy Integration**

### **Observable & Telemetry-Driven**

### **Extensible via Federation & Partner APIs**

### **Automation & Governance-Ready**

### **Developer Experience Consistency**

### **Outcome Metrics & Continuous Improvement**

### **Backwards Compatibility & Graceful Evolution**

---

**API Architecture Pattern Evaluation**
=======================================

### **On-Demand Architecture**

(desired-state, async workflow resources)

### **Traditional Quote → Order Pattern**

(check for RPC leakage)

### **Resource-Driven CRUD**

### **Experience-Driven Façade APIs**

(check for ID mapping, complexity masking)

---

**Workflow & Async Operations Review**
======================================

* 202 Accepted usage
* Operation resource presence
* Location header correctness
* Status transitions
* Error handling
* Timeouts + retry model

---

**Developer Experience (DX) Assessment**
========================================

* Schema clarity
* Example payloads
* Error examples
* Documentation quality
* OAS completeness
* Try-it-out readiness
* Consistency across APIs

---

**Summary of Gaps**
===================

| Category | Gap | Severity | Recommendation | Owner |
| --- | --- | --- | --- | --- |

---

**Recommended Remediation Roadmap**
===================================

* Taxonomy updates
* Converting RPC to workflow resources
* Async conversion
* Canonical modeling changes
* Identity/claims alignment
* Observability improvements
* Security fixes
* Governance automation in CI

---

**Final Architecture Rating**
=============================

| Category | Score (1–5) | Notes |
| --- | --- | --- |
| Style Guide Conformance |  |  |
| Cloud-First Principle Alignment |  |  |
| API Consistency |  |  |
| Identity & Entitlement Model |  |  |
| Security Posture |  |  |
| Developer Experience |  |  |
| Operational Readiness |  |  |