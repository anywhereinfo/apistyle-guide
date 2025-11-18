# The Canonical Data Model (CDM): The Foundation for API Standardization

> Confluence Page ID: 675693986024, Version: 4

In our pursuit of a robust and agile enterprise architecture, we face a fundamental challenge: managing the immense complexity of interrelated services and configurations. The common pitfall is to chase a single, perfect, all-encompassing data model, which is unrealistic and ultimately stifles innovation.

This document outlines our strategy for the Canonical Data Model (CDM). It is not a proposal for a rigid, monolithic system, but rather a **principle-based framework for creating a system of models**. This approach allows us to build a stable foundation while embracing flexibility and isolating complexity. Our strategy is founded on three core architectural principles:

1. **Bounded Contexts:** We will manage complexity by breaking down our enterprise into logical domains, each with its own specific and consistent model.
2. **The Immutable Core + Optional Extensions:** We will define a stable, non-negotiable core for each data entity and provide flexibility through well-defined, optional extensions.
3. **Standardized & Extensible by Design:** All models will be defined using a common, machine-readable format that is inherently designed to be extended over time without breaking its foundation.

Adopting this approach is mandatory for achieving the goals of our Strategic Interface Layer (SIL) and Partner Interface Layer (PIL). It is the key to delivering standardization, agility, and true decoupling.

Example Bounded Contexts
------------------------

Defining clear bounded contexts is crucial for managing complexity across Lumen's vast product and network portfolio. Here are some logical domains:

#### **1. Product Catalog & Quoting Context**

* **Responsibility:** Manages the commercial definitions of Lumen's products. This context is concerned with what can be sold, where it can be sold, and at what price.
* **Example Data:** Product SKUs, pricing rules, service level agreements (SLAs), and serviceability data (which addresses are on-net). It defines a "service" from a sales perspective.

---

#### **2. Order Management & Orchestration Context**

* **Responsibility:** Governs the end-to-end lifecycle of a customer order from submission to completion. It tracks the status of an order as it moves through various fulfillment systems.
* **Example Data:** Order ID, customer details, list of services on the order, and high-level order statuses (e.g., `In Progress`, `Completed`, `Cancelled`).

---

#### **3. Customer & Site Management Context**

* **Responsibility:** Acts as the master source for all customer account information and their physical locations (sites).
* **Example Data:** Customer legal name, account numbers, contact information, and site addresses (CLLI codes, latitude/longitude, site contacts).

---

#### **4. Service Provisioning & Activation Context**

* **Responsibility:** This is a large context often broken down further. It's concerned with the technical configuration of a service on the network.

  + **Access/UNI Sub-Context:** Manages the customer-facing interface. Data includes port speed, encapsulation type and VLAN IDs. This is the physical or virtual handoff to the customer.
  + **Metro & Transport Sub-Context:** Manages the underlying network paths within and between cities. Data includes MPLS LSPs, segment routing paths, and optical circuit details.
  + **Edge & Peering Sub-Context:** Manages how customer traffic gets to the internet or other VPN sites. Data includes BGP sessions, route-maps, and VRF/GRT definitions.
  + **Service Layer Sub-Context:** Defines the end-to-end service construct itself. For an L3VPN, this context would orchestrate elements from the Access and Edge contexts to build the complete virtual private network.

---

#### **5. Network & Service Assurance Context**

* **Responsibility:** Monitors the health of the network and services post-activation. It manages fault detection, performance monitoring, and trouble ticketing.
* **Example Data:** Circuit IDs, alarm statuses, performance metrics (latency, jitter), and trouble ticket numbers. A "service" in this context is something to be monitored.

---

#### **6. Billing & Invoicing Context**

* **Responsibility:** Manages the financial aspects of a customer's service. It's concerned with monthly recurring charges (MRCs), usage-based charges, and generating invoices.
* **Example Data:** Billing account numbers, charge codes, and invoice details.

What is the Canonical Data Model (CDM)?
---------------------------------------

The Canonical Data Model (CDM) is the single, unified, and consistent data structure for our key business entities (e.g., Customer, Order, Service, Port) across the enterprise. It acts as the universal language for all our APIs.

In the context of our new architecture—the Strategic Interface Layer (SIL) and the Partner Interface Layer (PIL)—the CDM is not just beneficial, it is **absolutely mandatory** for achieving our goals of standardization, agility, and decoupling.

Why the CDM is Critical in the PIL and SIL Context
--------------------------------------------------

The CDM's primary role is to enforce the principle of **decoupling**—shielding consuming applications from the technical complexity and volatility of backend systems.

| Strategic Interface Layer (SIL) | Partner Interface Layer (PIL) |
| --- | --- |
| **Mission:** Provide clients (internal and external) with a stable, predictable, and consistent view of our business services. | **Mission:** Protect our internal services from the complexity and constant change of integrating with external third-party vendors. |
| **How the CDM Helps:** The SIL uses the CDM as its public contract. All data, regardless of its source system, is translated into the standard CDM format before being exposed through the SIL. | **How the CDM Helps:** The PIL uses the CDM as its internal standard. It translates vendor-specific data models into our internal CDM, ensuring our core services only ever have to understand one data language. |

Benefits for the Strategic Interface Layer (SIL)
------------------------------------------------

| Benefit | Why it Matters |
| --- | --- |
| **Consistent Client Contract** | The CDM ensures that all clients (mobile, web, partners) receive the same field names, formats, and structures for an entity like 'Customer', regardless of which backend system is supplying the data. This eliminates ambiguity and reduces integration costs for consumers. |
| **Simplified Orchestration** | When the SIL orchestrates a complex business process, data retrieved from one system (e.g., Billing) is immediately recognizable and usable when passed to another system (e.g., Provisioning). The CDM removes the need for multiple, complex data translations within the SIL itself. |
| **Accelerated Migration** | We can replace a legacy backend system with a new strategic one without changing the client-facing API contract. The old system was translated to the CDM, the new system will natively speak the CDM. This gives us immense technical and business flexibility. |

Benefits for the Partner Interface Layer (PIL)
----------------------------------------------

| Benefit | Why it Matters |
| --- | --- |
| **Internal Decoupling** | Internal services only ever see and use our Canonical Model for a concept like a 'Port'. They are entirely shielded from the specific, often messy, data formats (XML, JSON, SOAP) required by partners like AT&T or Verizon. |
| **Plug-and-Play Partner Integration** | Partner Adaptors become simple two-way translators: **Our CDM ⇄ Partner Format**. This means integrating a new partner (e.g., Starlink) is fast, requiring only a new adaptor, with **zero code change** to the PIL or our internal services. |
| **Vendor Neutrality** | Business rules for partner selection (managed by the PIL) can be based on standardized CDM attributes (e.g., `portType`, `locationID`), not vendor-specific identifiers. This makes switching partners a business decision, not a major technical project. |

Governance and Ownership
------------------------

This is a partnership between Product and Architecture.

| Area | Product Team Responsibility | Architecture Team Responsibility |
| --- | --- | --- |
| **Definition** | Define the **business fields, naming conventions, data types, and business rules** for their domain's model. They are the ultimate authority on what a "Customer" or "Order" means to the business. | Define the **technical format** (e.g., JSON Schema, OpenAPI), **security policies** (e.g., Field-Level Authorization), and overall **versioning strategy**. They provide the guardrails and tools. |
| **Version Control** | **Drive the roadmap** for the next version of the model, communicating changes clearly to all consumers and stakeholders. They manage the "product backlog" for the data model. | **Enforce Semantic Versioning** (e.g., v1.0.0), ensuring no breaking changes are introduced without proper notification and deprecation plans. They act as the release managers for the technical standard. |
| **Enforcement** | Ensure all **new strategic backends** created by the domain team natively expose the CDM. Advocate for the CDM in all new development. | Mandate that all **Adaptors** (both SIL and PIL) perform the necessary translation to the CDM before data is passed to core services. They ensure compliance at the architectural level. |

The CDM is a Product, Not a Project
-----------------------------------

To succeed, we must treat our Canonical Data Models as long-lived, strategic assets.

### A. The CDM as an Internal Product

A project has a start and an end date. A product has a lifecycle, a roadmap, and dedicated owners who seek to maximize its value for its customers.

* **Your Consumers are Customers:** The "customers" of your domain's CDM are all the other development teams (internal and external) who will use it. Your goal is to make their job easier by providing a clear, stable, and well-documented data contract.
* **Create a Roadmap:** What new attributes will be needed in the next 6-12 months? Are there upcoming business initiatives that will require changes to the `Order` or `Service` entity? Plan for this evolution and communicate it transparently.
* **Prioritize and Justify Changes:** Every change to the CDM has a ripple effect. Changes must be justified by clear business value, not just technical preference. Use a backlog to manage and prioritize requested changes from consuming teams.

### B. The Strategic Framework: Domain-Driven Design (DDD)

The principle of "Context Boundaries" mentioned earlier comes directly from Domain-Driven Design (DDD), a crucial framework for avoiding common CDM pitfalls. DDD helps us manage complexity by acknowledging that a single model cannot effectively serve the entire business.

* **Bounded Contexts are Your Friend:** The business is composed of different "Bounded Contexts" (e.g., Sales, Support, Billing), each with its own specific language and model. A "Customer" in the Sales context (with attributes like `leadScore`) is different from a "Customer" in the Billing context (with attributes like `paymentTerms`).
* **The CDM is the Shared Kernel:** Instead of a universal "God Model," the CDM should represent the **Shared Kernel**—the small, stable subset of the model that different contexts must agree on. The `customerID` and `legalName` are canonical and shared. The Sales team's `leadScore` is not; it belongs only within their Bounded Context.
* **Anti-Corruption Layers:** When contexts interact, DDD uses an "Anti-Corruption Layer" to translate between models. Our SIL and PIL adaptors are perfect examples of this. They protect our core services from the specific, and often messy, data models of other systems.