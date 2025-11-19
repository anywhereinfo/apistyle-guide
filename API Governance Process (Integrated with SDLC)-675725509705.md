# API Governance Process (Integrated with SDLC)

> Confluence Page ID: 675725509705, Version: 4

1. Overview
-----------

This document defines the standard lifecycle and governance process for designing, building, deploying, and deprecating APIs at Lumen. It ensures APIs are consistent, secure, aligned with strategic goals (per the Principles of Cloud-First API Design), and adhere to the official Lumen API Style Guide. This process integrates governance checkpoints directly into the standard Software Development Lifecycle (SDLC).

2. Process Stages
-----------------

---

### Stage 1: Planning Phase (e.g., PI Planning)

* **Trigger:** Product Owner (PO) or Solution Architect identifies the need for a new API or significant changes (including deprecation) to an existing one.
* **Action:** Notify the API Governance Working Group (via designated intake form/meeting).
* **Required Info:**

  + Clear **Business Purpose** & Use Case.
  + High-level functional requirements.
  + Anticipated consumers (internal/external/partner/wholesale).
  + If deprecating: Reason, proposed timeline, and replacement strategy (if any).
* **Governance Review (Initial Checkpoint):**

  + Strategic Alignment: Does this align with Cloud-First Principles?
  + Domain Placement: Correct Business Function or Product Domain?
  + Reuse Analysis: Does this capability exist? New API or extend existing?
  + Scope & Pattern: High-level agreement on API Pattern (CRUD, Façade, Async).
  + Deprecation Impact: Initial assessment of deprecation plan and user impact.
* **Benefits:** Prevents duplication, ensures strategic alignment early, guides teams to correct domain/pattern, anticipates governance workload.

---

### **Stage 2: Design Phase**

* **Trigger:** Development team is ready to start API design.
* **Action (Repo Request):**

  + **New API:** Request new Git repo from API Governance (provide domain, name, purpose, team). Governance provisions repo per standards.
  + **Existing API:** Identify existing repo. Discuss versioning strategy (new major version `v(n+1)` for breaking changes, minor update to `v(n)` for non-breaking additions) with governance, based on the approved API Versioning Standard.
* **Action (OAS Development):**

  + Development team designs the API in a feature branch, creating/updating `openapi.yaml`.
  + **Mandatory:** Use the central Spectral linter ruleset locally during development. Only lint-error-free OAS files should be submitted.
  + **Performance NFRs:** Define and document expected latency, TPS, and payload limits alongside the OAS.
  + **Documentation:** Ensure all operations, parameters, schemas, and examples are clearly documented within the OAS (`summary`, `description`).
* **Governance Support:** Provide design consultation, clarification on standards, examples.

---

### Stage 3: Review Phase (Pull Request)

* **Trigger:** Development team completes OAS design/update.
* **Action:** Submit a Pull Request (PR) against the main branch. Tag API Governance Working Group.
* **Automated Checks (CI/Jenkins):**

  + PR **MUST** trigger automated checks.
  + **Job Steps:** Run official Spectral linter ruleset; (Optional) Run breaking change detection.
  + **Outcome:** Failed checks **MUST** block the PR. Dev team must fix and resubmit.
* **Security Review (Post-Automation, Pre-Governance):**

  + **Trigger:** Automated checks pass.
  + **Action:** The designated Security team reviews the OAS PR, focusing on authentication, authorization scopes, data sensitivity, potential vulnerabilities (input validation, etc.), and adherence to security standards.
  + **Outcome:** Security provides approval or required remediation comments on the PR.
* **Manual Governance Review (Post-Security Approval):**

  + **Trigger:** Security review is approved.
  + **Focus:**

    - Semantics: Correct domain modeling? Clear resource names?
    - Design Patterns: Correct Cloud API Pattern? REST principles followed?
    - Consistency: Adherence to URI Standard, Error Handling (LPDP-Mini), Headers, etc.? Documentation quality within OAS?
    - Versioning: Correct application of the API Versioning Standard?
    - Performance NFRs: Documented and realistic?
  + **Outcome:**

    1. **Approved:** Governance approves the PR.
    2. **Rejected with Comments:** Governance rejects PR with actionable feedback.
    3. **Exception Required:** Governance identifies a necessary deviation from standards.
* **Exception Handling:**

  + If an exception is required, the Governance Working Group documents the justification and submits it to the **API Governance Approval Board**.
  + The Board reviews, tracks, and formally approves or denies the exception request. Approved exceptions are documented alongside the standard.
* **Final Approval & Merge:** Once all reviews (Automated, Security, Governance) pass and any exceptions are approved, the PR is merged. This merge signifies the OAS is **officially approved**.

---

### Stage 4: Implementation & Testing Phase

* **Trigger:** OAS is approved and merged.
* **Action (Development):** Implement API backend based on the approved OAS contract.
* **Action (Performance Testing):**

  + Performance tests are run against the implemented API.
  + **Goal:** Validate against documented NFRs.
  + **Feedback Loop:** Failures require remediation. Significant changes impacting the OAS contract **MUST** notify API Governance.

---

### Stage 5: Deployment Phase

* **Trigger:** API implementation is complete and tested.
* **Action (Gateway Team):**

  + Gateway team is notified (via automated process or ticket) that an approved API version is ready.
  + Gateway team pulls the **approved OAS** (from the main branch or designated tag in the repo).
  + Configure Apigee proxy based on OAS, applying policies.
  + Expose API on the gateway.

---

### Stage 6: Publication Phase

* **Trigger:** API is deployed and ready for consumers.
* **Action (Dev Portal Team / Automation):**

  + The **approved OAS** from the repository is ingested into the Developer Portal.
  + API reference documentation is automatically generated and published.

---

### Stage 7: Deprecation Phase

* **Trigger:** Decision made during Planning Phase (Stage 1) to deprecate an API, operation, field, etc.
* **Phase 1: Announce 📣**

  + Endpoint remains fully functional.
  + OAS **MUST** be updated: mark relevant element(s) with `deprecated: true`.
  + API responses **SHOULD** include `Deprecation` and `Sunset` headers (per RFC 8594) indicating announcement date and removal date.
  + Developer Portal documentation **MUST** be updated with deprecation notice, timeline, and migration path (if any).
  + Communicate proactively to known consumers.
* **Phase 2: Sunset 🌇**

  + On the announced `Sunset` date:
  + Remove the deprecated element(s) from the API implementation.
  + The gateway **SHOULD** return appropriate errors (e.g., `404 Not Found` or `410 Gone` for removed endpoints; potentially `400 Bad Request` if a deprecated field is used incorrectly).
  + Update OAS to remove the element(s).
  + Update Developer Portal documentation.

---

3. RACI Matrix
--------------

|  |  |  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Activity** | **Product Owner** | **Sol. Architect** | **Dev Team** | **API Governance WG** | **API Gov. Board** | **Security Team** | **Gateway Team** | **Dev Portal Team** |
| 1. **Planning: Identify Need & Notify Gov.** | **R**, A | **R** | I | C | I | I | I | I |
| 1. **Planning: Initial Governance Review** | C | C | I | **A**, R | I | I | I | I |
| 2. **Design: Request Repo / Versioning** | I | C | **R**, A | C | I | I | I | I |
| 2. **Design: Develop OAS & NFRs** | C | C | **A**, R | C | I | C | I | I |
| 2. **Design: Use Linter Locally** | I | I | **A**, R | I | I | I | I | I |
| 3. **Review: Submit PR** | I | I | **R**, A | I | I | I | I | I |
| 3. **Review: Run Automated Checks (CI)** | I | I | R | **A** (Ruleset) | I | I | I | I |
| 3. **Review: Security Review** | I | C | R | I | I | **A**, R | I | I |
| 3. **Review: Manual Governance Review** | C | C | R | **A**, R | C (Exceptions) | C | I | I |
| 3. **Review: Request/Approve Exception** | I | I | I | R | **A** | I | I | I |
| 3. **Review: Merge Approved PR** | I | I | **A**, R | I | I | I | I | I |
| 4. **Implement: Build API** | I | C | **A**, R | I | I | C | I | I |
| 4. **Implement: Performance Testing** | I | C | R | I | I | I | I | I |
| 5. **Deploy: Notify Gateway Team** | I | I | **R** | I | I | I | I | I |
| 5. **Deploy: Configure & Deploy Proxy** | I | I | I | C | I | C | **A**, R | I |
| 6. **Publish: Update Developer Portal** | I | I | I | I | I | I | C | **A**, R |
| 7. **Deprecate: Announce (OAS, Headers)** | C | C | **A**, R | C | I | I | C | R |
| 7. **Deprecate: Sunset (Remove Code)** | I | C | **A**, R | I | I | I | I | I |
| 7. **Deprecate: Update Gateway/Portal** | I | I | I | C | I | I | R | R |

**Legend:** **R**=Responsible, **A**=Accountable, **C**=Consulted, **I**=Informed