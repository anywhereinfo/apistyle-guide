# API Governance Metadata Model

> Confluence Page ID: 675829907489, Version: 6

1. Introduction
---------------

This document defines the metadata model for API governance. Its purpose is to capture the essential "delta" information—such as ownership, business context, and governance status—that is not already present in the OpenAPI Specification (OAS) file.

This model is a two-part system designed to work with an API's source of truth (the OAS) to enable discovery, automated governance, and lifecycle management.

1. **API-Level Metadata (**`api-metadata.yaml`**):** A separate metadata file that lives alongside the OAS file in the API's repository. This file describes the API as a whole.
2. **Endpoint-Level Metadata (OAS** `x-` **tags):** Custom extension properties placed *inside* the OAS file on specific operations to enable fine-grained, automated policy enforcement.

This structure makes the metadata discoverable by a cataloging tool (like Backstage) while keeping the machine-readable, automated governance rules co-located with the endpoint contracts they apply to.

2. API-Level Metadata (The `api-metadata.yaml` File)
----------------------------------------------------

This metadata **must** be stored in a dedicated YAML file (`api-metadata.yaml`) in the API's root directory.

* `schemaVersion` (String, Required): The version of this metadata schema, which enables CI validation and controlled evolution.

  + **Value:** `v1.0`

### 2.1. Ownership & Team

Identifies who is responsible for the API's development, maintenance, and business alignment. This is critical for accountability and for consumers to know who to contact.

* `apiName` (String, Required): The human-readable, canonical name of the API (e.g., "User Profile API", "Payment Orchestration Service").
* `ownershipModel` (String, Required): For the current governance scope, this value is always `Lumen-Owned.`This ensures a consistent ownership declaration and establishes a foundation for future expansion (e.g., `Externally-Owned`, `Joint-Owned`) when the internal API repository is introduced.
* `assetId`*(string, required)*: **Lumen Asset ID (SYSGEN)** that maps the API to its registered owner application in the Asset Management system (e.g., `SYSGEN-45123`). Used for authority, audit, and MAL PoC linkage.
* `businessOwner` (String, Required): The primary stakeholder or Product Manager responsible for the API's business function and roadmap (e.g., an email, user ID, or group alias).
* `technicalOwner` (String, Required): The Engineering Lead or Architect responsible for the API's technical design, implementation, and operational health.
* `developmentTeam` (String, Required): The name or alias of the team that builds and maintains the API (e.g., "identity-platform-team").
* `supportContact` (String, Required): The official support channel for consumers (e.g., a group email `api-support@example.com`, a Slack channel `#slack-team-identity`).

### 2.2. Business Context

Provides the "why" and "where" for the API within the larger enterprise architecture.

* `consumerAudience` (String, Required): The intended consumer channel, which dictates security and exposure.

  + **Values:** `internal-ui` (for internal UIs/frontends), `internal-service` (for other internal systems/automation), `external-partner`, `external-public`
* `apiLayer` (String, Required): The architectural classification of the API, based on its primary role.

  + **Values:** `Experience`, `Canonical`, `Process`, `System`
* `system` (String, Optional): A key/name that links this API to a larger "System" or "Domain" in the software catalog (e.g., `billing-platform`, `crm`, `customer-identity-domain`).

### 2.3. Governance & Lifecycle

Tracks the API's maturity and the level of governance applied.

* `lifecycleStage` (String, Required): The current stage of the API's life.

  + **Values:** `Design`, `In-Development`, `Active`, `Deprecated`, `Retired`
* `governanceLevel` (String, Required): The level of review and adherence to standards applied to this API. This informs consumer trust and risk.

  + **Values:** `Fully Governed`, `Partially Governed`, `Lift and Shift`, `Exception`
* `governanceProfile` **(String, Required):** The specific ruleset or "linter profile" that automation should apply to this API. This allows developers to self-select the strictness of the review based on the API's origin.

  + **Values:** `legacy` (Minimal checks for older APIs), `lift-n-shift` (Basic structural checks), `full-governance` (Strict adherence to all standards).
* `deprecationDate` (String, Optional): (If `lifecycleStage` is "Deprecated") The date (ISO 8601) when the API will no longer be supported.
* `sunsetDate` (String, Optional): (If `lifecycleStage` is "Deprecated") The date (ISO 8601) when the API will be shut down.

### 2.4. Technical & Design Details

Captures high-level technical attributes not defined in the OAS.

* `apiStyle` (String, Required): The architectural style of the API.

  + **Values:** `REST`, `RPC-style`, `GraphQL`, `gRPC`, `Async`
* `protocol` (String, Required): The primary protocol used.

  + **Values:** `HTTPS`, `WSS` (WebSockets), `AMQP` (Messaging), etc.
* `swaggerHubName`(String, Optional)**:** The unique, URL-friendly identifier (slug) for this API in SwaggerHub. This is used by CI/CD pipelines to sync the repository OAS with the SwaggerHub catalog.

  + **Example:** `customer-order-api` (Must match the SwaggerHub URL, not the display name).

### 2.5. Security & Compliance

Defines the overall risk and compliance posture for the API.

* `dataClassification` (String, Required): The **highest** sensitivity level of data handled by *any* endpoint in the API. This acts as a high-water mark for compliance.

  + **Examples:** `Public`, `Internal`, `Confidential`, `Restricted (PII, PCI, PHI)`
* `complianceRequirements` (List, Optional): A list of any compliance regulations that apply to the data handled by this API.

  + **Examples:** `["GDPR", "HIPAA", "PCI-DSS"]`

### 2.6. Review Audit Trail

An immutable log of the *latest* governance review. This section is updated automatically by the governance process or tooling upon a successful review.

* `reviewAuditTrail` (Object, Optional): A container for all review-related metadata.

  + `reviewId` (String, Required): A unique identifier for the latest governance review.
  + `reviewDate` (String, Required): The timestamp (ISO 8601) when the latest review was completed.
  + `reviewStatus` (String, Required): The outcome of the review.

    - **Values:** `Pending`, `Approved`, `Approved with Conditions`, `Rejected`
  + `securityReviewId` (String, Optional): A unique identifier that links this governance review to an internal risk, compliance, or security review record.

### 2.7. Schema Validation

To enable automated CI validation, editor autocompletion, and ensure conformity, all `catalog.yaml` files **must** validate against the official JSON Schema.

* **Schema Location:** `catalog-schema.json`

3. Endpoint-Level Metadata (Inside the OAS File)
------------------------------------------------

For fine-grained, automated governance (e.g., security policies, linting, gateway configuration), we must apply metadata at the **Operation (Endpoint) level**.

This metadata **must** be stored *inside* the OAS file using **OpenAPI Specification Extensions** (fields prefixed with `x-`).

### 3.1. How to Store Endpoint Metadata

This metadata lives directly on the operation object within the OAS file, making the spec a self-contained source of truth for automation.

**Example within the OAS (**`api.oas.yaml`**):**

```
paths:
  /users:
    post:
      summary: Create a new user
      operationId: createUser
      
      # --- Automated Governance Metadata Starts Here ---
      x-data-classification: "Restricted (PII)"
      x-operation-type: "lro" # (Long-Running Operation)
      x-financial-operation: false
      x-idempotent: false
      # --- Automated Governance Metadata Ends Here ---

      tags:
        - Users
      requestBody:
        description: User object to be created
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/User'
      responses:
        '202':
          description: Accepted. The user creation is in progress.
        '400':
          description: Invalid input
```

### 3.2. Key Endpoint-Level Metadata Fields

* `x-data-classification` (String, Required): The sensitivity of the data handled *by this specific endpoint*. This is the most critical field for automated security and can be more granular than the API-level high-water mark.

  + **Examples:** `Restricted (PII)`, `Confidential`, `Internal`, `Public`
* `x-operation-type` (String, Optional): Defines the behavior of the endpoint, allowing the API gateway to apply different policies (e.g., timeouts, retry logic).

  + **Examples:** `lro` (Long-Running Operation), `bulk-data`, `standard-sync`
* `x-financial-operation` (Boolean, Optional): Flags if this operation initiates a financial transaction, which can trigger stricter audit logging policies. Defaults to `false`.
* `x-idempotent` (Boolean, Optional): Explicitly declares if a `POST` or `PATCH` operation is safe to retry. This is crucial for resilient consumer design.

### 3.3. Addressing the "Messy Presentation" Concern

A common concern is that `x-` tags will clutter the rendered documentation (e.g., Swagger UI). This is a solvable presentation problem.

1. **Priority:** The primary consumer of `x-` tags is **automation** (linters, security scanners, API gateways). It is essential for these tags to be machine-readable.
2. **Solution:** The documentation tools should be configured to *interpret* these tags rather than just displaying them. Your developer portal can be customized to:

   * **Hide** governance tags from the default view.
   * **Render them as badges:** `x-data-classification: "Restricted (PII)"` could be rendered as a highly visible red "PII Data" badge.
   * **Render them as icons:** `x-operation-type: "lro"` could show a small "hourglass" icon.

4.0 API-Metadata JSON Schema
----------------------------

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "API Governance Catalog Metadata",
  "description": "Schema for the catalog.yaml file, which provides governance and discovery metadata for an API.",
  "type": "object",
  "properties": {
    "schemaVersion": {
      "description": "The version of this metadata schema.",
      "type": "string",
      "enum": ["v1.0"]
    },
    "apiName": {
      "description": "The human-readable, canonical name of the API.",
      "type": "string",
      "minLength": 1
    },
    "ownershipModel": {
      "description": "Defines the ownership classification of the API. For governance scope, always 'Lumen-Owned'.",
      "type": "string",
      "enum": ["Lumen-Owned"]
    },
    "assetId": {
      "type": "string",
      "minLength": 1,
      "description": "Lumen Asset ID (SYSGEN) linking to Asset Management",
      "examples": ["SYSGEN-45123"]
    },
    "businessOwner": {
      "description": "The primary stakeholder or Product Manager (email, ID, or alias).",
      "type": "string",
      "minLength": 1
    },
    "technicalOwner": {
      "description": "The Engineering Lead or Architect (email, ID, or alias).",
      "type": "string",
      "minLength": 1
    },
    "developmentTeam": {
      "description": "The name or alias of the team that builds and maintains the API.",
      "type": "string",
      "minLength": 1
    },
    "supportContact": {
      "description": "The official support channel for consumers (email or Slack channel).",
      "type": "string",
      "minLength": 1
    },
    "consumerAudience": {
      "description": "The intended consumer channel, which dictates security and exposure.",
      "type": "string",
      "enum": [
        "internal-ui",
        "internal-service",
        "external-partner",
        "external-public"
      ]
    },
    "apiLayer": {
      "description": "The architectural classification of the API, based on its primary role.",
      "type": "string",
      "enum": [
        "Experience",
        "Canonical",
        "Process",
        "System"
      ]
    },
    "system": {
      "description": "A key/name that links this API to a larger System or Domain in the software catalog.",
      "type": "string"
    },
    "lifecycleStage": {
      "description": "The current stage of the API's life.",
      "type": "string",
      "enum": [
        "Design",
        "In-Development",
        "Active",
        "Deprecated",
        "Retired"
      ]
    },
    "governanceProfile": {
      "description": "The specific ruleset or 'linter profile' that automation should apply to this API.",
      "type": "string",
      "enum": [
        "legacy",
        "lift-n-shift",
        "full-governance"
      ]
    },
    "governanceLevel": {
      "description": "The level of review and adherence to standards applied to this API.",
      "type": "string",
      "enum": [
        "Fully Governed",
        "Partially Governed",
        "Lift and Shift",
        "Exception"
      ]
    },
    "deprecationDate": {
      "description": "The date (ISO 8601) when the API will no longer be supported.",
      "type": ["string", "null"],
      "format": "date-time"
    },
    "sunsetDate": {
      "description": "The date (ISO 8601) when the API will be shut down.",
      "type": ["string", "null"],
      "format": "date-time"
    },
    "swaggerHubName": {
      "description": "The unique, URL-friendly identifier (slug) for this API in SwaggerHub. Used for CI/CD sync.",
      "type": "string"
    },
    "apiStyle": {
      "description": "The architectural style of the API.",
      "type": "string",
      "enum": [
        "REST",
        "RPC-style",
        "GraphQL",
        "gRPC",
        "Async"
      ]
    },
    "protocol": {
      "description": "The primary protocol used.",
      "type": "string",
      "enum": [
        "HTTPS",
        "WSS",
        "AMQP",
        "MQTT"
      ]
    },
    "dataClassification": {
      "description": "The highest sensitivity level of data handled by any endpoint in the API.",
      "type": "string",
      "enum": [
        "Public",
        "Internal",
        "Confidential",
        "Restricted (PII, PCI, PHI)"
      ]
    },
    "complianceRequirements": {
      "description": "A list of any compliance regulations that apply to this API.",
      "type": "array",
      "items": {
        "type": "string"
      },
      "uniqueItems": true
    },
    "reviewAuditTrail": {
      "description": "An immutable log of the latest governance review.",
      "type": "object",
      "properties": {
        "reviewId": {
          "description": "A unique identifier for the latest governance review.",
          "type": "string",
          "minLength": 1
        },
        "reviewDate": {
          "description": "The timestamp (ISO 8601) when the latest review was completed.",
          "type": "string",
          "format": "date-time"
        },
        "reviewStatus": {
          "description": "The outcome of the review.",
          "type": "string",
          "enum": [
            "Pending",
            "Approved",
            "Approved with Conditions",
            "Rejected"
          ]
        },
        "securityReviewId": {
          "description": "A unique identifier that links to an internal risk or security review record.",
          "type": "string"
        }
      },
      "required": [
        "reviewId",
        "reviewDate",
        "reviewStatus"
      ],
      "additionalProperties": false
    }
  },
  "required": [
    "schemaVersion",
    "apiName",
    "ownershipModel",
    "assetId",
    "businessOwner",
    "technicalOwner",
    "developmentTeam",
    "supportContact",
    "consumerAudience",
    "apiLayer",
    "lifecycleStage",
    "governanceProfile",
    "governanceLevel",
    "apiStyle",
    "protocol",
    "dataClassification"
  ],
  "additionalProperties": false
}
```