# Proposed Roadmap

> Confluence Page ID: 675752411161, Version: 2

none

**Thematic Roadmap**
--------------------

Our strategy will progress through three distinct phases:

* **Q4 2025: Foundation & Piloting:** Formalize all core standards and validate them with a pilot project.
* **Q1 2026: Expansion & Enablement:** Operationalize our governance process and expand modeling to the next strategic domain.
* **Q2 2026: Operationalization & Automation:** Scale governance through automation and drive adoption with a world-class Developer Experience (DX).

---

### **Q4 2025 — Foundation & Piloting**

**Goal:** Formalize our core API standards, processes, and reference architectures while designing the first canonical models for a selected pilot project.

#### **Foundational Playbook & Governance**

**Objective:** Draft, socialize, and ratify all v1.0 foundational governance documents.  
**Deliverables:**

* API Style Guide v1.0
* Draft Domains/SubDomains and their corresponding URIs
* API Security Standard v1.0
* “Sell” Side Reference Architecture v1.0
* “Buy” Side Reference Architecture v1.0
* API Lifecycle and Governance Process v1.0

  **Activities:**
* Draft all initial governance and reference documents.
* Socialize drafts with Architecture, Security, and Engineering.
* Incorporate feedback and ratify through the API Governance.

#### **Canonical Data Modeling (Pilot)**

**Objective:** Validate the canonical modeling approach through a real-world pilot project.  
**Deliverables:**

* Ratified Decision Document on Modeling Strategy (Build vs. MEF/TMF).
* Selected Pilot Project (e.g., LMCC / MCGW) with defined Bounded Contexts.
* Immutable Core and Optional Extensions for pilot domain.
* Ratified Canonical Models v1.0 published to the Schema Registry.  
  **Activities:**
* Convene working group and perform rapid gap analysis.
* Conduct domain workshops with subject matter experts.
* Draft and review OpenAPI/JSON Schema specifications.

#### **Governance Automation & Observability**

**Deliverables:**

* Implement automated governance checks using Spectral / Redocly CLI.
* Integrate validation pipeline within pilot CI/CD workflow.
* Implement W3C trace headers in gateway

---

### **Q1 2026 — Expansion & Enablement**

**Goal:** Operationalize the governance process, expand canonical modeling to the next domain, and initiate structured enablement across teams.

#### **Foundational Playbook & Governance**

**Deliverables:**

* Operationalized governance process with first formal API design reviews.
* Enablement and training materials delivered to product teams.

#### **Partner Data-Sharing & Consent Architecture**

**Objective:** Define and enforce the shared-customer model, including consent-based visibility and partner projections.  
**Cross-Dependencies:** Security, Data Governance, API Governance.  
**Deliverables:**

* Defined Partner Data-Sharing Model.
* Consent Artifact Schema and Enforcement Framework.
* Partner “Subset Visibility” API Design Pattern.

#### **Canonical Data Modeling (Expansion)**

**Deliverables:**

* Selected next business domain for modeling (e.g., Order Management).
* v1.0 Canonical Models ratified and published.

#### **Tactical Engagement & Analysis**

**Deliverables:**

* Completed API portfolio analysis for the target domain.
* Prioritized backlog of alignment and remediation tasks delivered to project teams.

5. **Platform & Enablement (New)**

**Deliverables:**

* API Observability Standard v1.0 drafted.
* Defined Core Metrics (latency, error rate, throughput).
* Published SLO Template Library for API services.
* Assess if Swaggerhub can satisfy needs of internal API Portal

---

### **Q2 2026 — Operationalization & Automation**

**Goal:** Scale governance through automation, strengthen reliability with observability standards, and improve adoption through a world-class Developer Experience (DX).

**Focus Areas:**

* Expand automated governance checks across enterprise CI/CD pipelines.
* Launch internal API Portal
* Establish SLO Governance Dashboards for operational visibility.
* Begin cross-domain canonical alignment and schema versioning.