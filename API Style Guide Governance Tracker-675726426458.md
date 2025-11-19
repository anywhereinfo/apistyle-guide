# API Style Guide Governance Tracker

> Confluence Page ID: 675726426458, Version: 19

1. Overview
-----------

This page tracks the progress of the API Working Group in reviewing, consolidating, and approving the official Lumen API Style Guide. The goal is to create a single, unified set of standards that will be used for all API development and governance.

### Status Legend

* 🟩 **Approved:** The standard has been reviewed and approved by the working group.
* 🟧 **Pending:** The standard is under review, has open issues, or is awaiting a formal decision.
* ⬜ **Draft:** The standard exists but has not yet been formally reviewed by the group.

2. Master Governance Tracker
----------------------------

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| Subject Area | Source | Status | Open Issues / Questions | Notes / Rationale |
| **Principles of Cloud-First API Design** | Confluence | 🟩 **Approved** |  |  |
| **OAS Specification Best Practices** | Confluence | 🟩 **Approved** | 3.1.x (limited support). Current support 3.0.x | Updated support for 3.0.x |
| **API Versioning Strategy** | Confluence | 🟩 **Approved** | `Deprecation`: date format to follow RFC. `Modify status code` | Updated |
| **API Pagination Standards** | Confluence | 🟩 **Approved** | `next/prev` is not possible at this time | Removed |
| **API Standardized Error Responses** | Confluence | 🟩 **Approved** | `traceparent`, `9457` (RFC 9457 Problem Details) | Created RFC 9457 Problem Detail profile for Lumen |
| **API URI Standards and Design Patterns** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Idempotency** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Partial Resource Retrieval** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Delete Method** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Caching & Concurrency** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: HTTP Headers** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Date & Time Naming** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Standardized Data Types** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: Rest & Resource Design** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: JSON Payload** | Confluence | 🟩 **Approved** |  | extracted date standards seperately from json payload |
| **API Standard: GET Method** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: POST Method** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: PUT Method** | Confluence | 🟩 **Approved** |  |  |
| **API Standard: PATCH Method** | Confluence | 🟩 **Approved** |  |  |
| **webhook** | (New Item) | ⬜ **Draft** | `jacob(webhooks - undermetristic), casey (sre)` | Lower priority, not required for base API Styleguide |
| **API Governance Process** | Confluence | ⬜ **Draft** |  | Lower priority, not required for base API Styleguide |
| **API Standard: Long-Runing Operations (LRO)** | Confluence | 🟩 **Approved** |  | Lower priority, not required for base API Styleguide |
| **API Standard: Bulk Data Transfer** | Confluence | 🟩 **Approved** |  | Lower priority, not required for base API Styleguide |
| **API Standard: Batch Operations** | Confluence | 🟩 **Approved** |  |  |

Export to Sheets