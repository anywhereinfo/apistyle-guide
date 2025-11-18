# Proposed Sell Reference Architecture: : A Strategic Interface Layer Architecture

> Confluence Page ID: 675690086637, Version: 4

The Challenge
-------------

Lumen, a company with a history of frequent mergers and acquisitions, faces a common architectural dilemma: a fragmented backend landscape. Decades of integrating disparate systems have resulted in a lack of standardization, multiple APIs serving the same concepts with inconsistent data models, and a significant impediment to agility and unified customer experiences. The core challenge is to standardize Lumen's digital platform without requiring an immediate, disruptive "big bang" overhaul of all legacy systems

The Proposed Solution: A Strategic Interface Layer (SIL)
--------------------------------------------------------

To address these challenges, we propose an architecture centered around a  **Strategic Interface Layer (SIL)**. This layer acts as a crucial abstraction and orchestration point, designed to provide immediate standardization for clients while enabling a phased, long-term migration of legacy backends.

The architecture revolves around two key principles:

* **Client Abstraction:** Shielding client applications from the inherent complexity and inconsistencies of Lumen's diverse backend systems.
* **Phased Migration:** Allowing legacy backends to be refactored or replaced incrementally, without impacting the client-facing APIs.

![sellsideref.png](images/675690086637-0-sellsideref.png)

Key Components:
---------------

1. **Lumen Clients:** Front-end applications (web, mobile, partner integrations) that consume Lumen's digital services. Their primary need is a stable, consistent, and well-documented API.
2. **API Gateway :** The external entry point for all client requests. After enforcing standard gateway functions (security, rate limiting, logging), this component will route all request to SIL Layer
3. **Strategic Interface Layer (SIL):** The is “the face of Lumen APIs”. It is key interface, based off a canonical, well governed API model exposed to clients. This is the orchestration and standardization layer. Its responsibilities include:

   * **Standardized Contract:** Exposing a unified, domain-driven API contract for core Lumen concepts (e.g., Customer, Order, Service).
   * **Lean Orchestration:** For requests that involve multiple legacy backends or complex workflows not yet encapsulated by a single strategic backend.
   * **Data Model Transformation:** Ensuring that data returned to clients conforms to the strategic data model, regardless of the underlying backend's format.
   * **Backward Compatibility:** Managing API versions to ensure existing clients are not broken during backend transitions.
4. **Backend Adapter Layer:** A sub-component of the SIL (or a set of distinct services called by the SIL) responsible for translating the standardized SIL requests into the specific API calls and data formats required by individual legacy backends. Each adapter is purpose-built for a particular legacy system.
5. **Legacy Backends:** The existing, unstandardized systems resulting from Lumen's M&A history. These systems will eventually be deprecated, refactored, or replaced.

Operational Flow
----------------

* A **Lumen Client** makes a request to the **API Gateway**.
* The **API Gateway / Smart Router** routes the request to the **Strategic Interface Layer (SIL)**.
* The **SIL** then performs any necessary orchestration, calls the appropriate **Backend Adapter** for legacy systems.
* Responses are transformed by the SIL (if originating from legacy systems) to adhere to the strategic data model before being sent back to the client via the Gateway.

Interaction Principles (The Rules of Engagement)
------------------------------------------------

This defines how the components **must** interact to ensure decoupling and maintain clear architectural boundaries.

* **External Consumers:** All external clients **must** only ever call the APIs exposed at the API Gateway. They are forbidden from calling any internal service directly.
* **API Gateway:** The Gateway after performing its function, routes the requests to SIL
* **Decoupling:** The public-facing API contract **must** remain stable, even when a backend system is migrated from the "Legacy Path" to the "Modern Path." This transition must be completely transparent to the external consumer.

Governance & Non-Functional Requirements (The Standards)
--------------------------------------------------------

This ensures that every implementation is secure, consistent, and observable.

* **Security:** All endpoints exposed at the API Gateway **must** be secured using the enterprise standard (e.g., OAuth 2.0).
* **Observability:** All transactions **must** include a correlation ID that is passed through every layer for end-to-end tracing.
* **API Standards:** All public-facing APIs **must** adhere to the official Lumen API Style Guide.

The Colocation Principle: Context
---------------------------------

The "Sell" Side Reference Architecture is designed as a layered, distributed system. Effective performance of this architecture relies on low-latency communication between its internal components, particularly between the Strategic Interface Layer (SIL) and its Backend Adapters.

However, Lumen's current enterprise constraint of a single, centralized API Gateway in GCP creates a "network hairpin" problem, introducing prohibitive latency (~130ms+ per hop) for any service-to-service communication that crosses cloud boundaries. While long-term solutions like a Federated Gateway or a Service Mesh are the strategic goal, they are not immediately available.

Therefore, to ensure the performance and viability of all near-term SIL implementations, the **Colocation Principle is adopted as the mandatory, pragmatic interim deployment pattern**

The Colocation Principle: Mandated Rules
----------------------------------------

Given the immediate need for a workable solution, the following rules are mandated for all SIL implementations until a federated gateway or service mesh is available.

* **SIL and Adapters as a Single Deployment Unit:** The SIL and its corresponding Backend Adapters **must** be deployed in the same cloud, same VPC, and ideally the same cluster. They should be treated as a single, cohesive deployment unit to guarantee low-latency communication.
* **Intra-Cloud Orchestration Only:** The SIL is strictly forbidden from performing real-time, synchronous orchestration that calls services or adapters across different cloud environments (e.g., a SIL in AWS cannot call an adapter in GCP). All real-time orchestration must be contained within a single cloud boundary.
* **Separate Logical Deployments Required:** While they must be colocated, the SIL and its Backend Adapters **must** remain logically separate deployments (e.g., distinct microservices or containers). Bundling adapters as a library (e.g., a JAR file) inside the SIL is considered an anti-pattern, as it creates a monolith and prevents independent deployment, scaling, and maintenance.
* **Multi-Cloud Aggregation via Asynchronous Data:** For any use case that requires a unified view of data from multiple clouds, the approved pattern is **asynchronous data aggregation**, not real-time orchestration. The SIL **must** perform a single, low-latency query against a pre-aggregated data store (e.g., a Data Hub or Lakehouse) rather than performing a live, cross-cloud "fan-out" to multiple services.