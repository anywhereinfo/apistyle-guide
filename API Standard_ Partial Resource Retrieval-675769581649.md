# API Standard: Partial Resource Retrieval

> Confluence Page ID: 675769581649, Version: 1

none

Purpose
-------

This page extends the **Lumen API Style Guide (v1.0)** to define a standard for **partial resource retrieval** — enabling clients to request only the fields they need, reducing payload size and latency for high-volume NaaS APIs (e.g., telemetry, connections, interfaces).

Guiding Principle
-----------------

> “Always return full canonical resources **by default**, but allow clients to **opt-in** to partial responses using explicit field projection.”

Partial retrieval improves performance and bandwidth efficiency without changing the canonical model or breaking API contracts.

Standard Definition
-------------------

| Category | Rule | Requirement |
| --- | --- | --- |
| **Parameter Name** | `fields` | **MUST** |
| **Location** | Query Parameter | **MUST** |
| **Type** | Comma-separated list of field names (no spaces) | **MUST** |
| **Behavior** | Return only specified fields; ignore unknown ones or return `400` | **MUST** |
| **Default** | Full resource if `fields` not provided | **MUST** |
| **Case Sensitivity** | Field names are case-sensitive and match the schema exactly | **SHOULD** |
| **Nested Objects** | Use dot notation for sub-fields (e.g., `location.city`) | **MAY** |
| **Error Handling** | Return `400 Bad Request` with `problem+json` if invalid field names supplied | **SHOULD** |

Example: Fabric Connection Service
----------------------------------

### 🔹 Full Resource

GET /fabric/v1/connections/12345

**Response**

{
"id": "12345",
"name": "AWS-Transit-Connect",
"status": "active",
"bandwidth": {
"committed\_mbps": 1000,
"burst\_mbps": 2000
},
"location": {
"city": "Denver",
"region": "US-CENTRAL-1"
},
"provider": "aws",
"tags": ["gold", "naas"]
}

### 🔹 Partial Resource (Using `fields`)

GET /fabric/v1/connections/12345?fields=id,name,status

**Response**

{
"id": "12345",
"name": "AWS-Transit-Connect",
"status": "active"
}

Nested Field Example – MCGW Interface
-------------------------------------

GET /mcgw/v1/interfaces/7df9a?fields=id,device.name,device.state

**Response**

{
"id": "7df9a",
"device": {
"name": "edge-router-01",
"state": "up"
}
}

Validation & Governance Automation
----------------------------------

| Tool | Rule | Example |
| --- | --- | --- |
| **Spectral Linter** | Ensure `fields` is declared in GET operations where allowed | `oas-parameter-name: fields` |
| **Governance Rule** | Lint rule `naas-partial-response-allowed` | Enforced only for `GET` endpoints returning resources > 10 fields |

Performance Considerations
--------------------------

* Reduces payload size by **30–80 %** for high-volume telemetry endpoints.
* Lowers serialization/deserialization overhead in API Gateway and MCGW controllers.
* Prevents over-fetching in portal UI and CLI integrations.

Security & Compliance
---------------------

* Projection **cannot** bypass authorization. The API MUST validate that each requested field is permitted under the caller’s security scope.

Summary
-------

| Benefit | Impact |
| --- | --- |
| Reduced payloads and latency | Better API responsiveness for UI and CLI clients |
| Standardized pattern across products | Consistent developer experience |
| Governance-ready | Easily enforced via OAS and Spectral rules |