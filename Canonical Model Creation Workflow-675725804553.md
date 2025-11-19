# Canonical Model Creation Workflow

> Confluence Page ID: 675725804553, Version: 3

**Introduction**
----------------

This document provides the official blueprint and workflow for creating an API Canonical Model at Lumen. Its purpose is to guide architects and teams through a strategic, collaborative process that ensures every canonical model is a consistent, reusable, and valuable enterprise asset.

Following this workflow is essential for implementing our core architectural principles of decoupling, consistency, and agility. It transforms the creation of a model from a siloed technical task into a strategic design process that aligns with our business domains.

As illustrated in the diagram below, this workflow produces reusable **Domain Models** (the foundation), which are consumed by product-facing **Facade APIs** (the experience layer), and are all enforced by an automated **Governance Layer** (the contract).

![canonicalmodelusage.png](images/675725804553-0-canonicalmodelusage.png)

**Guiding Principles**
----------------------

Before starting the workflow, internalize these core principles that govern our approach:

* **Model the Domain, Not the System:** The model must represent the business reality (a "Customer," a "Service"), not the database tables or technical quirks of a legacy backend system.
* **Embrace Bounded Contexts:** Do not attempt to create a single model for the entire enterprise. Acknowledge that a "Port" in the Access domain has different attributes than a "Port" in the Assurance domain. Design models that are specific and unambiguous within their logical boundary.
* **Design for Stability and Extension:** Identify the stable, **Immutable Core** of a model and protect it. Allow for flexibility and future growth through well-defined **Optional Extensions**.
* **Treat the Model as a Product:** A canonical model is a long-lived asset with other development teams as its customers. It requires a product mindset, including dedicated ownership, a roadmap, and a clear versioning strategy.

---

**The Workflow**
----------------

### **Phase 1: Discovery and Scoping 🗺️**

In this phase, you define the problem space and establish the boundaries for your model.

1. **Define the Business Capability:** Clearly articulate the business process the model will support. Start with the "why."

   * *Example: "Provision a Last Mile Cloud Connect service" or "Retrieve a customer's billing history."*
2. **Establish the Bounded Context:** Work with product and enterprise architects to draw the conceptual boundary around the domain. Determine what entities and attributes are inside this context and what belongs to others. This is the most critical step for managing complexity.
3. **Identify Domain Experts and Stakeholders:** Identify the key people who have authoritative knowledge of this domain. This includes Product Managers, business stakeholders, and senior engineers from the relevant teams. Modeling cannot happen in isolation.

### **Phase 2: Collaborative Model Design 🧱**

This is the core workshop phase where the model takes shape through collaboration.

1. **Host Design Workshops:** Bring the identified domain experts together. Use a whiteboard or digital collaboration tool to visually map out the entities and their relationships within the bounded context.
2. **Identify Core Entities:** Within the bounded context, identify the key "nouns."

   * *Example: For a Network domain, this might be* `Port`*,* `Circuit`*,* `LogicalDevice`*, and* `BgpSession`*.*
3. **Define the Immutable Core:** For each entity, define its essential, non-negotiable attributes that are stable and unlikely to change.

   * *Example: For a* `Port` *model, this would be its* `portID`*,* `siteID`*, and* `physicalInterfaceType`*.*
4. **Define Optional Extensions and Relationships:** Flesh out the model with attributes that provide flexibility or may not always be present. Define the relationships between entities.

   * *Example: A* `Circuit` *is composed of two* `Ports`*.*

### **Phase 3: Formalization and Review**

This phase turns the conceptual model into a formal, ratified asset.

1. **Draft the Formal Specification:** Translate the workshop outputs into a formal, machine-readable contract using OpenAPI Specification (v3.x) and standalone JSON Schema files. These schemas will form the **Domain OAS** (e.g., Network, Connectivity) shown in Figure 1.
2. **Conduct Architectural Review:** Submit the draft specification for a formal review with the API Governance Guild. The model is checked for adherence to enterprise standards using automated tools like **Spectral** alongside peer review.
3. **Ratify and Version the Model:** Once approved, the model is formally ratified. It is assigned a version number (e.g., v1.0.0) following Semantic Versioning (SemVer) and becomes the official standard for that entity within its bounded context.

### **Phase 4: Publication, Implementation, and Governance 🚀**

The final phase involves publishing the model and managing its lifecycle within our architecture.

1. **Publish to the Domain OAS Registry:** The ratified model specification is published to the official Lumen Schema Registry (a dedicated Git repository). This registry contains the versioned **Domain OAS** files (the green boxes in Figure 1) and is the single source of truth for all teams.
2. **Implement via Product API Facades:** The canonical models are consumed by the various **Product APIs** (e.g., MCGW, VPN), which act as facades. As shown in the middle layer of Figure 1, these APIs use `$ref` to reference the canonical schemas from the Domain OAS registry and can add their own local, product-specific extensions.
3. **Enforce via the Governance Layer:** The ratified model is a living asset. The **API Governance Layer** (the top box in Figure 1) uses tools like Spectral and the Redocly Registry to automatically validate the Product APIs. This ensures they are correctly implementing the canonical models, enforces consistent naming, and detects any "schema drift," protecting the integrity of our API ecosystem. Any proposed changes to a canonical model must follow this governance process before a new version is released.