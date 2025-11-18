# Proposed "Buy" Reference Architecture for Partner Integration at Lumen: The PIL Model

> Confluence Page ID: 675690971354, Version: 3

This document outlines the proposed **"Buy" Reference Architecture**—the strategy for integrating external partner capabilities (e.g., off-net ports, connectivity from AT&T, Verizon, Starlink) into Lumen's digital platform. This architecture is designed for **security, governance, and decoupling** to ensure that Lumen's internal services are protected from the complexity and potential volatility of third-party APIs.

The Need for Standardization
----------------------------

Just as the Strategic Interface Layer (SIL) standardizes Lumen's *selling* capabilities, a robust "Buy" architecture is needed to standardize *partner sourcing*. When buying capabilities, the goal is to abstract the partner's technology and data model from your internal systems. Without it, internal Lumen services would be tightly coupled to specific partner technologies, leading to:

* **High Maintenance Cost:** Any change to a partner's API would require updating every Lumen service that calls it.
* **Lack of Agility:** Adding a new partner (e.g., Starlink) would require extensive code changes across the organization.
* **Security Risk:** Scattered management of external partner credentials and security policies.

The Core Principle: A Playbook, Not a Single Solution
-----------------------------------------------------

This Reference Architecture is not a single, monolithic system. It is a strategic **pattern** that will be implemented for each distinct "Buy" side business domain (e.g., an "OffNet Connectivity PIL," a "Cloud Marketplace PIL").

The Reference Architecture is prescriptive about the **principles and components** (the "what" and "why") but flexible about the **implementation specifics** (the "how"). It provides strong guardrails while empowering delivery teams to make the right architectural choices for their specific business problem.

Structural Components (The "What")
----------------------------------

This defines the non-negotiable building blocks that **must** be used in every implementation to ensure structural consistency across the enterprise.

| Component | Primary Role | Rationale |
| --- | --- | --- |
| **Partner Interface Layer (PIL)** | The internal-facing facade for a specific business domain. | Centralizes domain-specific business logic (e.g., partner selection) and exposes a single, clean API for internal consumers. |
| **Partner Adaptors** | A dedicated microservice for each individual partner. | Ensures technical isolation. If a partner's API changes, only its single adaptor is updated, protecting all other components. |
| **External Partner Gateway** | The single, governed exit point for all outbound traffic to partners. | Critical for governance. Enforces uniform security, rate limiting, and centralized logging, preventing unmanaged service-to-internet sprawl. |
| **Webhook Ingestion Layer** | The single, governed entry point for all inbound asynchronous events from partners. | Provides a secure and resilient "front door" for partner webhooks, authenticating and queuing events before they are processed. |

Interaction Principles (The Rules of Engagement)
------------------------------------------------

This defines how the components **must** interact to ensure decoupling and maintain clear architectural boundaries.

* **Internal Consumers:** Internal services **must** only ever call the PIL's interface. They are forbidden from calling a Partner Adaptor directly.
* **The PIL:** The PIL **must** expose a clean, domain-specific API that hides the underlying complexity of partner orchestration and selection.
* **Partner Adaptors:** An Adaptor's responsibility is limited to the technical concerns of a single partner (authentication, data transformation). It **must not** contain business logic that spans multiple partners.

Governance & Non-Functional Requirements (The Standards)
--------------------------------------------------------

This ensures that every implementation is secure, consistent, and observable, regardless of the specific use case.

* **Security:** All endpoints exposed by the PIL and the Webhook Ingestion Layer **must** be secured using the enterprise standard (e.g., OAuth 2.0). All outbound traffic from the External Partner Gateway **must** use appropriate transport-layer security (e.g., mTLS).
* **Observability:** All transactions **must** include a correlation ID that is passed through every layer (PIL, Adaptor, Gateway) for end-to-end tracing.
* **Logging:** A standardized logging format **must** be used by all components to enable centralized analysis and alerting.

A Decision Framework for Extensions (The "How")
-----------------------------------------------

The Reference Architecture provides guided flexibility. Any team building an implementation **must** consciously make and document their decisions based on this framework.

The most critical decision is the state management strategy. The design **must** explicitly document whether it is using a "Stateful" or "Stateless" pattern and provide a clear justification.

* **Question** 1: What **is the Source of Truth?**

  + If the **partner** is the absolute source of truth for the resource (like a cloud resource), a **Stateless** pattern is required.
  + If **Lumen** is the source of truth for the *transaction* itself (like an OffNet order), a **Stateful** pattern is required.
* **Question 2: What is the Lifecycle of the Transaction?**

  + Is it a short-lived, real-time request/response? -> **Stateless**.
  + Is it a long-running, asynchronous process that could take days or weeks? -> **Stateful**.
* **Question 3: How are Out-of-Band Changes Handled?**

  + The design **must** address how it will handle the "tricky case" where a resource's state is changed on the partner's side without a webhook (e.g., via their portal). A Stateful pattern, for example, must include a state reconciliation mechanism

The Secure Outbound Flow
------------------------

All requests for partner services must follow a stringent, controlled path to ensure security and maintainability.

![buy_arch.png](images/675690971354-0-buy_arch.png)

**Internal Lumen Service → PIL → Adaptor → External Partner Gateway → 3rd Party Partner API**

| **Step** | **Component** | **Action** |
| --- | --- | --- |
| **Internal Call** | **Lumen Service** | Makes a standardized request to the PIL (e.g., "Find Port in Dallas"). |
| **Business Logic** | **PIL** | Applies business rules to select the best partner (e.g., "Verizon is cheapest here") and forwards the request to the corresponding Adaptor. |
| **Transformation/Vendor specific AuthN** | **Adaptor** | Transforms the standardized PIL request into the exact format and data model required by the chosen partner's API. Also responsible for obtaining/caching the OAuth token for a specific external vendor. |
| **Routing** | **External Partner Gateway** | Manages mTLS/TLS, applies rate limits, and centrally logs the external transaction. |
| **External Execution** | **3rd Party Partner** | Receives and executes the request. |

Key Benefits
------------

This layered "Buy" architecture solves the core enterprise challenges of partner integration:

* **True Decoupling:** Lumen's internal systems only ever talk to the **PIL**'s standardized contract. They are completely ignorant of partner technology, meaning replacing 3rd party vendor with a new provider has zero impact on core Lumen code.
* **Centralized Security:** By forcing all outbound calls through the **External Partner Gateway**, Lumen ensures all traffic is uniformly secured and adheres to external partner agreements and rate limits.
* **Agile Onboarding:** Integrating a new partner is a focused project—it requires only building a new **Partner Adaptor**. The existing PIL, internal services, and Gateway remain untouched.
* **Resilience:** The **PIL** can implement failover logic. If the vendor Adaptor reports an API error, the PIL can automatically retry if the error is recoverable.

Inbound Flow - Handling Asynchronous Partner Events (Webhooks)
--------------------------------------------------------------

For long-running processes like service provisioning, partners provide asynchronous status updates via webhooks. The Reference Architecture mandates a secure, resilient, and decoupled pattern for ingesting and processing these events.

The flow is the mirror image of the outbound request, ensuring that internal systems are shielded from the complexity and proprietary formats of partner events.

1. **Ingestion:** A partner system sends a `POST` request to a single, secure endpoint exposed by the **Webhook Ingestion Layer**. This endpoint is specific to the partner (e.g., `https://webhooks.lumen.com/v1/att`). The payload is in the partner's native format.
2. **Authentication & Queuing:** The Ingestion Layer's only jobs are to authenticate the request (e.g., via an API key or HMAC signature) and immediately place the raw, untranslated event onto a dedicated **Partner Events Message Bus**. This ensures resilience and durability.
3. **Adaptor Translation:** The appropriate **Partner Adaptor** (e.g., the `AT&T Adaptor`) is subscribed to its topic on the message bus. It consumes the raw event and performs the critical translation, converting the partner's proprietary status codes and data fields into the official **Lumen Canonical Model**.
4. **Canonical Event Publication:** The Adaptor publishes a new, clean, **canonical event** (e.g., `lumen.order.status.updated`) onto a separate **Internal Canonical Events Bus**.
5. **Internal Consumption:** The appropriate **PIL** (or other internal service) is subscribed to the internal bus. It receives the clean, canonical event and can reliably take action, such as updating an order's state in its database.

![webhookflow.png](images/675690971354-1-webhookflow.png)

**Deployment Mandate: The Colocation Principle for the PIL**
------------------------------------------------------------

### 1. Context and Rationale

The "Buy" Side Reference Architecture is designed as a layered, distributed system that relies on high-chattiness, low-latency communication between its internal components, particularly between the **Partner Interface Layer (PIL)** and its **Partner Adaptors**. This is especially critical during the "fan-out" process for multi-partner quoting.

However, Lumen's current enterprise constraint of a single, centralized API Gateway in GCP creates a "network hairpin" problem, introducing prohibitive latency (~130ms+ per hop) for any service-to-service communication that crosses cloud boundaries. While long-term solutions like a Federated Gateway or a Service Mesh are the strategic goal, they are not immediately available.

Therefore, to ensure the performance and viability of all near-term PIL implementations, the **Colocation Principle is adopted as the mandatory, pragmatic interim deployment pattern.**

### 2. The Colocation Principle: Mandated Rules

Given the immediate need for a workable solution, the following rules are mandated for all PIL implementations until a federated gateway or service mesh is available.

* **PIL and Adapters as a Single Deployment Unit:** The PIL and the Partner Adapters it needs to orchestrate for a specific domain **must** be deployed in the same cloud, same VPC, and ideally the same cluster. They should be treated as a single, cohesive deployment unit to guarantee low-latency communication during quote fan-outs and other internal calls.
* **Intra-Cloud Orchestration Only:** The PIL is strictly forbidden from performing real-time, synchronous orchestration that calls Partner Adapters across different cloud environments (e.g., a PIL in AWS cannot call an adapter in GCP). All real-time orchestration **must** be contained within a single cloud boundary.
* **Separate Logical Deployments Required:** While they must be colocated, the PIL and its Partner Adapters **must** remain logically separate deployments (e.g., distinct microservices or containers). Bundling adapters as a library (e.g., a JAR file) inside the PIL is considered an anti-pattern, as it creates a monolith and prevents independent deployment, scaling, and maintenance.
* **Handling Multi-Cloud Data Requirements:** For any use case requiring a unified view of data from partners whose adaptors reside in different clouds, the primary principle is to **colocate all adaptors required for real-time orchestration**. Since Partner Adaptors are location-agnostic, they should be deployed in the same cloud environment as the PIL that consumes them.