# Lumen's Audience-Driven API Strategy

> Confluence Page ID: 675727377167, Version: 3

**Executive Summary**
---------------------

To win in the digital marketplace, Lumen cannot offer a single, one-size-fits-all API. Our success depends on providing a tailored, world-class experience for each of our key customer segments: **Carriers**, large **Enterprises**, and **Direct/Digital-Native** customers.

This document outlines our proposed strategy for designing and exposing APIs for each audience. Our approach is to leverage industry standards like MEF and TM Forum for interoperability and enterprise completeness, while offering a simplified, proprietary API for customers who prioritize speed and developer experience.

This strategy will be implemented through our architectural cornerstones: the **Strategic Interface Layer (SIL)** and the **Partner Interface Layer (PIL)**, which will present the correct, purpose-built "API face" to each consumer.

**The Core Principle: A Multi-Faceted API Platform**
----------------------------------------------------

Our architecture is designed to present a specific, purpose-built API "face" to each audience from a common set of underlying capabilities. This allows us to meet the unique needs of each segment without creating duplicative backend logic.

**API Strategy by Customer Segment**
------------------------------------

### **Carrier-to-Carrier: The Interoperability Standard**

* **Recommendation:** Strictly adhere to **MEF LSO** `Sonata` **APIs**.
* **Rationale:** This is the non-negotiable global standard for inter-carrier automation. When another carrier wants to connect with Lumen to buy or sell services, they will expect a MEF-compliant API. This reduces friction, eliminates the need for custom integrations, and positions Lumen as a modern, easy-to-integrate partner.
* **Architectural Implementation:** The `Partner Interface Layer (PIL)` is responsible for exposing and consuming MEF LSO `Sonata` APIs.

### **Enterprise Customers: The Complete Digital Experience**

* **Recommendation:** A combination of **MEF LSO** `Cantata` **APIs** and **TM Forum Open APIs**.
* **Rationale:** Large enterprise customers require a complete, end-to-end digital experience that covers both technical and business operations.

  + **MEF LSO** `Cantata` provides the standardized, on-demand interface for their **technical teams** to programmatically order, configure, and manage network services.
  + **TM Forum Open APIs** provide the standard interface for their **business operations** to handle commercial processes like service qualification, quoting, trouble ticketing, and billing.
* **Architectural Implementation:** The `Strategic Interface Layer (SIL)` will expose a combination of MEF and TMF APIs to enterprise customers.

### **Direct Customers & Developers: The Simplicity Standard**

* **Recommendation:** A simplified, proprietary **"Lumen Native API"**.
* **Rationale:** For direct customers, developers, and DevOps teams who value speed and simplicity above all, a more opinionated and streamlined proprietary API can provide a superior developer experience (similar to the native APIs of PacketFabric or Megaport). This API should be designed for maximum ease of use and the fastest possible time-to-first-call.
* **Architectural Implementation:** This "Lumen Native API" will be a simplified facade built on the `Strategic Interface Layer (SIL)`. A single, easy-to-use call to our native API will trigger a series of orchestrated calls to the underlying standard MEF and TM Forum APIs, hiding the complexity from the end-user.

**Summary & Implementation**
----------------------------

|  |  |  |  |
| --- | --- | --- | --- |
| Audience | Recommended API Standard | Architectural Layer | Key Purpose |
| **Carrier Partners** | **MEF LSO** `Sonata` + **TM Forum Open APIs** | Partner Interface Layer (PIL) | Interoperability & Wholesale |
| **Large Enterprise Customers** | **MEF LSO** `Cantata` **+ TM Forum Open APIs** | Strategic Interface Layer (SIL) | End-to-End Digital Experience |
| **Direct / Digital-Native Customers** | **Simplified Proprietary API** | Strategic Interface Layer (SIL) | Ease of Use & Developer Experience |

The **API Governance** will oversee the creation, versioning, and documentation of these APIs. All public-facing API specifications will be published to the Lumen Developer Portal, with clear guides indicating which API is best suited for each use case.