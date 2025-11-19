# Principles of Cloud-First API Design

> Confluence Page ID: 675768664368, Version: 2

These principles are the strategic solution to the very problems our customers and internal teams face. They provide the "North Star" that justifies our tactical API Style Guide.

How to Use These Principles: A Guide for Stakeholders
-----------------------------------------------------

This section translates the high-level principles into actionable guidance for different roles.

### For Product Managers

**Your Role: Define the "What" and the "Why"** You are the voice of the customer and the business. These principles empower you to define products, not just features.

* **Define Workflow Resources, Not Verbs:** When writing user stories, define the outcome the customer wants, not the steps they take. Your requirement should be "A customer can change bandwidth with a single API call" (**Principle 1 & 2**). This forces the creation of a simple orchestration façade.
* **Make Monetization a Requirement:** Every new API feature **MUST** have defined billing and entitlement logic. You must specify the `usageType` and `meteringUnit` as part of the feature definition (**Principle 5**).
* **Prioritize a "No-Ticket" Experience:** Your product vision **MUST** be geared toward 100% API-driven self-service. Any requirement for manual intervention or a support ticket is a design failure (**Principle 6**).
* **Own the API Metrics:** You are responsible for the API's success KPIs. You **MUST** define and track metrics for adoption, latency, and error rates to drive your roadmap and prove business value (**Principle 13**).

### For Developers & Engineers

**Your Role: Implement the "How"** You build the reliable, consistent, and scalable platform. These principles are your blueprint for implementation.

* **Implement the Asynchronous Pattern:** For any operation that takes more than 2 seconds (e.g., provisioning), you **MUST** implement the async request-reply pattern. Return a `202 Accepted` and a `Location` header pointing to a job/operation resource (**Principle 3**). Do not build long-polling, synchronous endpoints.
* **Enforce Standards in Your Pipeline:** You **MUST** integrate the provided Spectral linting rules into your CI pipeline. A build that violates the API Style Guide (naming, structure, etc.) **MUST** fail (**Principle 11**).
* **Use the Standard Patterns:** Do not invent your own error models, pagination, or authentication flows. You **MUST** use the platform-standard `problem+json` for errors, `Link` headers for pagination, and OAuth 2.0 for security (**Principle 12 & 8**).
* **Abstract Internal Complexity:** When building a façade, you are responsible for hiding the "identifier hell." Your implementation **MUST** handle the mapping between the single canonical API ID and all necessary internal system IDs.

### For Architects & Platform Leadership

**Your Role: Enforce the Platform Vision** You are the guardians of platform cohesion, long-term strategy, and governance.

* **Own the Canonical Taxonomy:** You are responsible for defining and enforcing the uniform naming and versioning strategy (`/{product}/v{version}/{resource}`). You **MUST** reject designs that create URI conflicts or inconsistent paths (**Principle 4 & 7**).
* **Champion Automation:** You **MUST** provide and maintain the automated governance tools (e.g., the master Spectral ruleset) that enable developers to comply with the standards at scale (**Principle 11**).
* **Design for the Ecosystem:** Your architectural reviews **MUST** ensure that new APIs are designed for federation and extensibility. A design that only works for a single use case and cannot support partners is a strategic failure (**Principle 10**).
* **Maintain Platform Trust:** You are responsible for enforcing policies on backwards compatibility and graceful evolution. You **MUST** ensure that teams follow proper deprecation procedures to avoid breaking client integrations (**Principle 15**).

1. Resource-Oriented Interface (The "How")
------------------------------------------

This is our core design philosophy. It means we model **all** interactions—from simple data management to complex, multi-step workflows—as "resources" (nouns) that a client can interact with.

This principle **explicitly forbids** exposing internal verbs or process steps (RPC-style).

It's crucial to understand that "resource-oriented" is **not** just for simple CRUD (Create, Read, Update, Delete). It is the *interface style* we use for two distinct patterns:

1. **CRUD-Style Resources:** For simple, state-based entities (e.g., `/ports`, `/users`).
2. **Workflow-Style Resources:** For complex, outcome-based actions (e.g., `/orders`, `/connections`).

**Why:** A consistent, resource-based interface enables self-service, idempotency, and alignment with all modern cloud provider conventions (AWS/GCP/Azure).

---

2. Desired-State & Outcome-Based (The "What")
---------------------------------------------

This is the *message* you send to a resource. It's the **opposite of an imperative command**.

Instead of a step-by-step *process*, the client sends a *declarative payload* describing the **desired end state**. Our orchestration façade is then responsible for making the "actual state" match that "desired state."

### Example: The Core Distinction

This is the most important concept. We **abstract** our internal process into a single, simple, resource-based API call.

#### ❌ Anti-Pattern: The RPC/Verb Process

The client must know our internal steps. **(FORBIDDEN)**

```
POST /api/checkPrice
POST /api/createQuote
POST /api/submitOrder
```

#### ✅ The "Workflow Resource" (Our Standard)

The client interacts with **one resource** (`/v1/connections`) and provides a **desired-state** body. Our façade hides the *entire* price/quote/order process.

```
POST /v1/connections
{
  "bandwidth": "10G",
  "src_port": "port-abc",
  "dst_port": "port-xyz"
}
```

---

**3. Asynchronous and Operation-Tracked**
-----------------------------------------

**What it means:** Long-running provisioning and teardown tasks (like the `POST /v1/connections` example above) **MUST** return an operation resource immediately.

**Example:**

1. `POST /fabric/v1/connections` → `202 Accepted`
2. **Response Header:** `Location: /fabric/v1/operations/12345`

**Why:** Mirrors GCP/Azure async operation patterns; avoids client timeouts and supports distributed job orchestration.

---

4. Product-Scoped Versioning
----------------------------

**What it means:** Versioning sits within the product namespace, never at the cross-domain layer.

**Example:**

* `/fabric/v1/telemetry/events`
* `/mcgw/v1/connections`

**Why:** Each product can evolve independently; prevents version collisions across domains.

---

5. Entitlement-Aware and Monetizable
------------------------------------

**What it means:** Every resource request validates the caller’s entitlements and records usage events for billing.

**Example Metadata:**

```
{
  "entitlementId": "ENT-123",
  "usageType": "bandwidth-hours",
  "meteringUnit": "GB"
}
```

**Why:** Supports SaaS-like pay-per-use and quota enforcement similar to cloud hyperscalers.

---

6. Self-Service and Declarative Lifecycle
-----------------------------------------

**What it means:** Customers and partners can onboard, provision, and manage services fully via API or IaC templates—no ticketing.

**Why:** Enables elasticity, automation, and integration into DevOps pipelines.

---

**7. Uniform Taxonomy and Namespacing**
---------------------------------------

**What it means:** Product namespaces and resource naming follow a canonical structure: `/{product}/v{version}/{resource}/{subresource}`

**Why:** Drives predictability across APIs, allowing governance automation (Spectral rules, gateway routing) and consistent developer experience.

---

8. Strong Identity and Policy Integration
-----------------------------------------

**What it means:** OAuth 2.0/OIDC as baseline; JWT claims carry entitlements, scopes, and tenant context.

**Example Scopes:** `fabric:read`, `mcgw:modify`, `wholesale:metrics`

**Why:** Enables fine-grained authorization and uniform policy enforcement across products.

---

9. Observable and Telemetry-Driven
----------------------------------

**What it means:** APIs emit standardized metrics and events: latency, success rate, error codes, SLO violations, and usage.

**Example:**

```
POST /telemetry/events
{
  "api": "/fabric/v1/ports",
  "latencyMs": 245,
  "status": 200
}
```

**Why:** Provides the foundation for automated SLA tracking, billing accuracy, and health dashboards.

---

10. Extensible Through Federation and Partner APIs
--------------------------------------------------

**What it means:** Partner and wholesale APIs follow the same design principles, enabling multi-tenant federation.

**Example:**

* `/wholesale/fabric/v1/connections`
* `/partner/mcgw/v1/orders`

**Why:** Supports ecosystem integration (hyperscalers, value-added resellers) without duplicating backend logic.

---

11. Automation and Governance-Ready
-----------------------------------

**What it means:** Design patterns are machine-verifiable. URI structure, version placement, naming, and required metadata are enforced by Spectral linting and CI pipelines.

**Why:** Guarantees compliance at scale, reduces manual reviews, and allows self-serve publication to catalogs (e.g., Backstage).

---

12. Developer Experience Consistency
------------------------------------

**What it means:** Consistent error models, pagination, and async patterns across all APIs; SDKs and documentation generated from OAS.

**Example Error:**

```
{
  "errorCode": "FABRIC-404",
  "message": "Port not found",
  "correlationId": "abc-123"
}
```

**Why:** Developers can reuse tooling and templates across products, just like AWS SDK parity across services.

---

13. Outcome Metrics and Continuous Improvement
----------------------------------------------

**What it means:** APIs publish KPIs on adoption, latency, and SLA compliance to a governance dashboard.

**Why:** Drives data-driven evolution and productization—every API is treated as a measurable SaaS asset.

---

14. Compliance, Security, and Data Residency by Design
------------------------------------------------------

**What it means:** Security and compliance are embedded through consistent header policies, trace IDs, and encryption standards, not added after design.

**Why:** Supports multi-jurisdictional and regulated workloads (finance, healthcare).

---

15. Backwards Compatibility and Graceful Evolution
--------------------------------------------------

**What it means:** APIs evolve additively; breaking changes require a new major version. Deprecated endpoints remain discoverable with `Deprecation` headers and clear timelines.

**Why:** Ensures clients can safely migrate; matches cloud provider deprecation best practices.