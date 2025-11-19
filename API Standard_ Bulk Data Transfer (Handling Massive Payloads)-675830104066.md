# API Standard: Bulk Data Transfer (Handling Massive Payloads)

> Confluence Page ID: 675830104066, Version: 2

1. Purpose & Scope
------------------

This standard is an extension of the [**API Standard: Long-Running Operations (Async API Pattern)**](/wiki/spaces/NAPA/pages/675812048943/API+Standard+Long-Runing+Operations+LRO).

The "Long-Running" guide solves the **TIME** problem (jobs > 60 seconds). This guide solves the **SIZE** problem (payloads > 5MB).

Together, they form the complete, mandatory pattern for any asynchronous job that processes bulk data.

2. The Problem (The Anti-Pattern)
---------------------------------

Synchronous REST APIs and API Gateways (like Apigee) are designed for small, fast, metadata-based transactions (typically `< 5MB`).

Teams **MUST NOT** attempt to use synchronous API calls for bulk data jobs. This anti-pattern includes:

* Sending multi-megabyte JSON/XML payloads in a `POST` or `PUT`.
* Expecting a multi-megabyte JSON/XML payload from a `GET`.
* Requesting gateway timeouts greater than the 60-second default.

This practice creates critical stability, security, and resource-exhaustion risks for the entire platform, affecting all other services. **Requesting longer timeouts or larger payload limits is not the correct solution.**

3. The Standard Pattern (Decoupled "Claim Check")
-------------------------------------------------

For any operation that involves a request or response payload larger than the 5MB gateway limit, the **Decoupled Data Transfer** pattern described below **MUST** be used.

This pattern separates the small, fast *control plane* (the API call) from the large, slow *data plane* (the file transfer). The API Gateway **MUST** only handle the control plane. The data plane is delegated to the cloud storage provider.

### Use Case A: Bulk Request (Massive Upload)

This flow is used when a client needs to *send* a massive file to an API.

1. **Step 1: Client Requests Upload URL (Control Plane)**

   The client makes a small, fast, and authenticated API call to the gateway to request a secure upload location.

   ```
   POST /v1/reports
   ```

   (See Metadata Schema below for required request/response formats)
2. **Step 2: API Responds with Pre-Signed URL (Control Plane)**

   The API service generates a secure, one-time-use upload URL from GCS/S3 or any cloud storage bucket and returns it to the client.

   ```
   {
     "fileId": "abc-123-xyz",
     "preSignedUploadUrl": "[https://storage.googleapis.com/bucket/upload?signature=..."
   }
   ```
3. **Step 3: Client Uploads File (Data Plane)**

   The client uploads the massive payload directly to the preSignedUploadUrl. This traffic bypasses the API Gateway.
4. **Step 4: Client Commits the File (Control Plane)**

   Once the upload is complete, the client makes a second small API call to notify the service that the file is ready for processing.

   ```
   POST /v1/reports/abc-123-xyz/process
   ```
5. **Step 5: API Responds Asynchronously (Control Plane)**

   The API Gateway now follows the API Standard: Long-Running Operations, returning an HTTP 202 Accepted with a statusUrl.

   ```
   {
     "jobId": "job-999",
     "status": "Queued",
     "statusUrl": "/v1/reports/status/job-999"
   }
   ```

### Use Case B: Bulk Response (Massive Download)

This flow is used when a client makes a request that *produces* a massive result.

1. **Step 1: Client Starts the Job (Control Plane)**

   The client makes a small, fast POST request to start the job, per the Long-Running Operations standard.

   ```
   POST /v1/reports/generate
   ```
2. **Step 2: API Responds Asynchronously (Control Plane)**

   The API Gateway instantly returns an HTTP 202 Accepted with a statusUrl. The 60-second timeout problem is solved.

   ```
   {
     "jobId": "job-777",
     "status": "InProgress",
     "statusUrl": "/v1/reports/status/job-777"
   }
   ```
3. **Step 3: Server Produces File (Background Work)**

   The backend service runs the 5-minute query and saves the massive payload as a file in a temporary cloud storage bucket, adhering to the security requirements below.
4. **Step 4: Client Polls for Status (Control Plane)**

   The client polls the statusUrl.

   ```
   GET /v1/reports/status/job-777
   ```
5. **Step 5: API Responds with Result URL (Control Plane)**

   Once the job is done, the API returns a small JSON payload containing a new, pre-signed download URL for the file in GCS/S3 or any cloud storage bucket. The massive payload problem is solved.

   200 OK

   ```
   {
     "jobId": "job-777",
     "status": "Complete",
     "resultUrl": "[https://storage.googleapis.com/bucket/download/report-777.csv?signature=..."
   }
   ```
6. **Step 6: Client Downloads File (Data Plane)**

   The client follows the resultUrl to download the massive file directly from the cloud storage bucket. This traffic bypasses the API Gateway.

### Use Case C: Failure Handling & Retries

The process is designed to be resilient.

#### Upload Failure Logic (Use Case A)

If the upload in **Step 3** fails (e.g., network error, URL expires), the `preSignedUploadUrl` is considered invalid.

* **DO NOT** retry the `PUT` to the same failed URL.
* **DO** restart the process from **Step 1** to request a **brand new** `preSignedUploadUrl`.

#### Download Failure Logic (Use Case B)

If the download in **Step 6** fails (e.g., network error, `resultUrl` expires), the job itself (`job-777`) is **still complete**.

* **DO NOT** queue a new job (do not call `POST /v1/reports/generate` again).
* **DO** restart the process from **Step 4** and call the `statusUrl` (`GET /v1/reports/status/job-777`) again.
* The server will see the job is "Complete" and generate a **brand new, fresh** `resultUrl` with a new expiration, allowing the client to retry the download.

4. Security Requirements
------------------------

All services implementing this pattern **MUST** adhere to the following security controls:

* **Short Expiry (TTL):** Pre-signed URLs MUST be generated with the shortest possible Time-to-Live (TTL).

  + **Upload URL (Use Case A):** Recommended TTL is **15 minutes**.
  + **Download URL (Use Case B):** Recommended TTL is **60 minutes**.
* **Scoped Access:** The URL MUST be scoped to the *minimum* required operation. For `preSignedUploadUrl`, this MUST be `PUT`. For `resultUrl`, this MUST be `GET`. List/Delete/Write permissions are forbidden.
* **Encryption:** All data at rest in the storage bucket (even temporary data) **MUST** be encrypted using server-side encryption (AES-256 or equivalent).
* **Auditing:** The API service **MUST** log the generation of every pre-signed URL, including the associated `fileId`/`jobId` and the authenticated client context (ideally including the W3C `trace_id`).

5. Metadata Schema Requirements
-------------------------------

The JSON payloads for the control plane (Steps 1, 2, 5) MUST be part of the API's formal OpenAPI 3.1 specification.

### Example Schema for Upload Response (Step 2)

```
{
  "type": "object",
  "properties": {
    "fileId": {
      "type": "string",
      "description": "The unique identifier for the uploaded file.",
      "example": "abc-123-xyz"
    },
    "preSignedUploadUrl": {
      "type": "string",
      "format": "uri",
      "description": "The temporary URL to PUT the file to."
    }
  },
  "required": ["fileId", "preSignedUploadUrl"]
}
```

### Example Schema for Download Response (Step 5)

This schema defines the successful response (`status: "Complete"`) and the failed job response (`status: "Failed"`).

```
{
  "type": "object",
  "properties": {
    "jobId": {
      "type": "string",
      "example": "job-777"
    },
    "status": {
      "type": "string",
      "enum": ["InProgress", "Complete", "Failed"],
      "example": "Complete"
    },
    "resultUrl": {
      "type": "string",
      "format": "uri",
      "description": "Present only when status is 'Complete'. This is the pre-signed URL to download the result file."
    },
    "error": {
      "description": "Present only when status is 'Failed'. Follows the LPDP-Mini v1.0 standard.",
      "$ref": "#/components/schemas/LpdpProblem"
    }
  },
  "required": ["jobId", "status"]
}
```

### Dependent Schemas (Error Profile)

The following schemas are required to support the `error` property above, per the **API Standard: Lumen Problem Details (LPDP-Mini v1.0)**.

```
{
  "components": {
    "schemas": {
      "LpdpProblem": {
        "type": "object",
        "additionalProperties": false,
        "description": "Minimal Problem-Details envelope (RFC 9457-compliant) per LPDP-Mini v1.0",
        "properties": {
          "title": {
            "type": "string",
            "description": "Short, human-readable summary."
          },
          "detail": {
            "type": "string",
            "description": "Optional longer explanation."
          },
          "errors": {
            "type": "array",
            "minItems": 1,
            "items": { "$ref": "#/components/schemas/LpdpError" }
          }
        },
        "required": [ "title", "errors" ]
      },
      "LpdpError": {
        "type": "object",
        "additionalProperties": false,
        "description": "Individual error item.",
        "properties": {
          "code": {
            "type": "string",
            "description": "Stable, machine-readable identifier (string form preferred)."
          },
          "message": {
            "type": "string",
            "description": "Human-readable explanation of the issue."
          },
          "meta": {
            "type": "object",
            "description": "Optional unregulated extension for machine-readable context.",
            "additionalProperties": true
          }
        },
        "required": [ "code", "message" ]
      }
    }
  }
}
```

6. Related Standards
--------------------

This standard MUST be implemented in conjunction with the following governance standards:

* [**API Standard: Long-Running Operations**](/wiki/spaces/NAPA/pages/675812048943/API+Standard+Long-Runing+Operations+LRO): This is a required component for the asynchronous flow.
* [**API Standard: Lumen Problem Details (LPDP-Mini v1.0)**](/wiki/spaces/NAPA/pages/675749954024/API+Standard+Lumen+Problem+Details+Minimal+Profile+LPDP-Mini+v1.0): This defines the mandatory error structure for a failed job.
* [**API Standard: Pagination**](/wiki/spaces/NAPA/pages/675750379523/API+Pagination+Standards): Teams should first attempt to use standard pagination to handle large *list-based* responses. This Bulk Data standard is for *single-object* massive payloads (like a generated file).

7. Automation Notes (for Governance Tools)
------------------------------------------

To support automated governance and enforce this standard, the following checks MUST be implemented in our tooling.

### 1. Design-Time Check (OAS & Metadata)

This check validates the API's *design* and *metadata* to ensure the correct pattern is being used.

* **Metadata Usage:** Developers **MUST** add a `tags` array containing `"bulk-data"` to any OpenAPI operation that follows this standard.
* **Automation Rule:** The Spectral linter in the CI pipeline MUST enforce the following:

  1. If `tags` contains `"bulk-data"`, the operation **MUST NOT** have a synchronous `200 OK` or `201 Created` response with a `content` body (it must be an async `202 Accepted`).
  2. If `tags` contains `"bulk-data"`, the operation **MUST** reference the standard schemas defined in this guide (e.g., for `preSignedUploadUrl` or `resultUrl`).

### 2. Build-Time Check (CI Pipeline & Gateway Config)

This check is the "hard guardrail" that validates the *as-built configuration* during deployment.

* **Automation Rule:** The CI/CD pipeline MUST scan the API Gateway (Apigee) proxy configuration files.
* **Action:** The build **MUST FAIL** if the configuration attempts to set the payload limit above the **5MB** enterprise standard.
* **Exception:** Any request for an exception (e.g., for a legacy system) MUST be flagged for manual review and approval by the API Governance Council.

8. Industry Precedent (The "Why")
---------------------------------

This is the standard, non-negotiable pattern used by all major cloud providers to protect their platforms. They all separate the metadata API from the data plane.

* **Amazon (AWS):** Uses [**S3 Pre-Signed URLs**](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html) to allow temporary, secure access to storage buckets.
* **Google (GCP):** Uses [**Cloud Storage Signed URLs**](https://cloud.google.com/storage/docs/access-control/signed-urls) for the same purpose.
* **Microsoft (Azure):** Uses [**Shared Access Signatures (SAS)**](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview) to provide delegated access to storage.

By adopting this standard, we are aligning with industry best practices for building stable, secure, and scalable services.