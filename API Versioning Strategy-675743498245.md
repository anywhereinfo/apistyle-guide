# API Versioning Strategy

> Confluence Page ID: 675743498245, Version: 5

This document outlines the proposed official strategy for versioning all public-facing REST APIs. A stable and predictable versioning strategy is critical for building trust with API consumers and ensuring a reliable developer experience.

Versioning Principles
---------------------

Our strategy uses both **Major** and **Minor** versions, but they are applied in different places to serve distinct purposes.

### Major Versions: The Public Contract 📜

The major version represents the public contract with the client and **MUST** only change when a **breaking change** is introduced.

* **Implementation**: The major version **MUST** be included in the URI path and prefixed with a "v" (e.g., `v1`, `v2`).

  ```
  https://api.lumen.com/inventory/v1/connections
  ```
* **Client Impact**: A client's integration is tied to this major version. We guarantee that no breaking changes will be made within a given major version.

### Minor Versions: The Diagnostic Tool 🩺

The minor version is an internal reference used to track non-breaking, additive changes. It serves as a crucial tool for communication and diagnostics.

* **Implementation**: The server **SHOULD** include the semantic version (major.minor) in an HTTP response header.

  ```
  API-Version: 1.2
  ```
* **Client Impact**: This has **zero impact** on the client's code. They are not required to send or parse this header. Its purpose is to provide a precise reference point for debugging, especially in distributed systems where deployment lag could lead to different nodes running different code versions.

Defining Breaking vs. Non-Breaking Changes
------------------------------------------

To ensure clarity, the following examples define what constitutes a breaking change versus a non-breaking change. When in doubt, a change **SHOULD** be treated as breaking.

### Breaking Changes (Requiring a New Major Version)

Breaking changes are modifications that can potentially break an existing client integration.

* **Request Changes:**

  + Removing or renaming a parameter.
  + Adding a new *required* parameter.
  + Making a previously optional parameter *required*.
  + Changing the data type of a parameter.
  + Adding a new validation rule to an existing parameter (e.g., restricting the length of a string).
  + Changing authentication or authorization requirements.
* **Response Changes:**

  + Removing or renaming a field in the response body.
  + Making a previously *required* response field optional.
  + Changing the data type of a response field (e.g., from an integer to a money object).
  + Changing the structure of the JSON payload (e.g., nesting an existing field).
  + Removing a value from an `enum`.
  + Any change to the primary HTTP Status Code returned by an existing API endpoint.
  + Adding a new value to enum

### Non-Breaking Changes (Minor Version Increments)

Non-breaking (additive) changes are modifications that should not break an existing integration. These changes will be available in all supported API versions.

* **Request Changes:**

  + Adding a new *optional* parameter.
  + Adding a new *optional* request header.
  + Adding an optional field in the request body
* **Response Changes:**

  + Adding a new field to the response body.
  + Adding a new endpoint or operation.
  + Adding a new response header.

API Lifecycle Management
------------------------

### Announcing Major Changes (Pre-Release Communication)

To provide our consumers with adequate time to plan and budget for upcoming migrations, breaking changes **SHOULD** be announced at least **6 months** *before* a new major version is released.

* This announcement **MUST** be made through the official API changelog, documentation, and, where possible, direct email communication to affected consumers.
* The announcement **MUST** detail the upcoming breaking changes, the reasons for them, and provide a preliminary migration guide.

### Active Version Support

All API products **MUST** support at least two active major versions during a migration period.

* When a new REST API version with breaking changes is released, the previous API version **MUST** be supported for at least **24 more months**.
* This overlap period allows clients to migrate at their own pace without service disruption.

### **Deprecating Endpoints and Features**

Deprecating an endpoint or feature requires a phased approach. The removal of any API element is a **breaking change** and can only happen in a new major version.

1. **Announce (Phase 1) 📣: The endpoint remains fully functional.**

   * In the OpenAPI specification, the operation **MUST** be marked with `deprecated: true`.
   * The server **SHOULD** return a `Sunset` header (RFC 8594) containing the exact date and time when the endpoint will be fully removed.
   * The server **SHOULD** also return a `Deprecation` header containing the date when the deprecation was announced. The format required by the RFCs is a standard **HTTP-date**, as defined in [RFC 7231](https://www.google.com/search?q=https://datatracker.ietf.org/doc/html/rfc7231%23section-7.1.1.1)
   * The documentation **MUST** be updated to reflect the deprecation, the removal timeline, and any new preferred endpoints.

   **Example HTTP Headers**

   ```
   HTTP/1.1 200 OK
   Deprecation: Wed, 22 Oct 2025 13:30:00 GMT
   Sunset: Fri, 22 Oct 2027 13:30:00 GMT
   ```
2. **Guide (Phase 2)** 🚀: Provide a clear path forward for consumers.

   * If a replacement exists, the server **SHOULD** return a `Link` header pointing to the new endpoint (e.g., `Link: <.../v1/new-endpoint>; rel="alternate"`).
   * If the feature is being removed entirely, the documentation **MUST** state this clearly.
3. **Remove (Phase 3)** ❌: After the support window, the endpoint is removed.

   * The deprecated endpoint **MUST** be completely removed in the next major version release (e.g., `/v2`).

### **Sunsetting and Decommissioning a Major Version**

After the mandated 24-month support window for a previous version ends, that version will be decommissioned.

* **Communication**: Consumers still using the old version will be notified via multiple channels about the upcoming shutdown.
* **Brownouts**: A "brownout" period may be implemented, where the old version is temporarily disabled for short periods to alert remaining users.
* **Final Response**: Once decommissioned, the old version's endpoints **SHOULD** return an HTTP `410 Gone` status code to indicate that the resource is intentionally and permanently unavailable.

**Handling Experimental (Beta) Features**
-----------------------------------------

To gather feedback on new features, a beta version may be released. Beta features are not bound by the standard 24-month support policy and can be changed or removed without the standard deprecation process.

* **Implementation**: Beta features **SHOULD** be identified clearly, either through a beta-specific URI (e.g., `/v1-beta/new-feature`) or a version header (`API-Version: 2.1.0-beta`).

**Updating Shared Data Models**
-------------------------------

Making a breaking change to a common data model (e.g., the `Address` object) across multiple APIs **MUST** be handled with the **"Expand and Contract"** pattern to avoid a disruptive, simultaneous release of new major versions for all dependent APIs.

1. **Expand (Non-Breaking)**: In the current major version, add new fields as *optional* and mark the old fields as *deprecated*. The server logic is updated to handle both old and new fields.
2. **Transition (Monitor)**: Allow consumers to migrate to the new fields over the 24-month support window. Monitor the usage of the deprecated fields.
3. **Contract (Breaking)**: In the next major version, remove the deprecated fields entirely. The new fields can now be made *required*