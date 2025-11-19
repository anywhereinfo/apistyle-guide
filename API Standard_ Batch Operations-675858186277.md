# API Standard: Batch Operations

> Confluence Page ID: 675858186277, Version: 1

1. Purpose
----------

Batch operations enable clients to perform **the same action on multiple resources** in a single request (e.g., create, update, or trigger actions across many items).  
This guide defines **Lumen’s governance standards** for batch APIs—latency thresholds, idempotency, status codes, response shape, and OpenAPI documentation.

Batch APIs must behave **predictably**, support **safe retries**, and meet **bounded synchronous latency**. If those guarantees can’t be met, use the **Long-Running Operation (LRO)** pattern.

---

2. Semantics
------------

| Property | Requirement |
| --- | --- |
| Verb | `POST` |
| Safe | ❌ No — modifies server state |
| Idempotent | ⚠ Recommended for synchronous; **required** for asynchronous |
| Response Type | `application/json` |
| Behavior | Processes multiple items of the same resource type atomically **per item** (not per batch) |

---

3. When to Use Batch Operations
-------------------------------

| Use Case | Example | Behavior |
| --- | --- | --- |
| Batch Create or Update | `POST /v1/customers/batch` | **201** if all succeed, **207** if mixed results, **202** if deferred. |
| Batch Action | `POST /v1/invoices/batch/void` | **200/207/202** depending on outcome. |
| Large Imports / Heavy Processing | `POST /v1/orders/import` | **202 Accepted** + `Location` to a job; follow the LRO guide. |

---

4. Execution Policy
-------------------

### 4.1 Synchronous Mode

A batch **may** execute synchronously when:

* **Predictable latency:** ≤ **30 s p95 / 60 s absolute**
* **Deterministic outcome:** same input → same observable result (conflicts included)
* **Retry safety (recommended):** supports `Idempotency-Key` for deduping retries

### 4.2 Asynchronous Mode

If synchronous boundaries can’t be guaranteed:

* Return `202 Accepted`
* Include `Location` to a job resource
* Follow the **LRO Style Guide** for polling, job states, lifecycle

---

5. Status Codes
---------------

| Code | Meaning | Applies To | Notes |
| --- | --- | --- | --- |
| **201 Created** | All items created | Batch create | May return representation or summary |
| **200 OK** | All items succeeded (non-create) | Batch action | Uniform success |
| **207 Multi-Status** | Mixed outcomes | Create/action | Return per-item results + summary |
| **202 Accepted** | Deferred job | Any | `Location` → job URI |
| **4xx/5xx** | Uniform failure | Any | Use RFC 7807 Problem Details |

**Rules:** Use **207** only when per-item outcomes differ. Use **201/200** for uniform success. Use **202** when work is deferred.

---

6. Response Structure
---------------------

### 6.1 207 Multi-Status (Partial Success)

```
HTTP/1.1 207 Multi-Status
Content-Type: application/json
{
  "results": [
    { "id": "c101", "status": 201, "message": "Created" },
    { "id": "c102", "status": 409, "message": "Duplicate customer" }
  ],
  "summary": { "total": 2, "succeeded": 1, "failed": 1 }
}
```

**Rules**

* **MUST** include `summary` (total/succeeded/failed)
* **MUST** use numeric HTTP status codes (100–599) for each item’s `status` field
* **MAY** include `message` or failure details
* **MAY** offer **compact** (failures-only) vs **full** modes
* **MUST NOT** truncate without pagination metadata
* **MAY** expose a **results** endpoint (optional, use-case dependent)

---

7. 202 Accepted (Asynchronous Jobs)
-----------------------------------

**Submit**

```
POST /v1/customers/batch
→ 202 Accepted
Location: /v1/jobs/9827
Retry-After: 10
```

**Poll**

```
GET /v1/jobs/9827
→ 200 OK
{
  "id": "9827",
  "state": "in_progress",
  "accepted": 1000,
  "processed": 300,
  "succeeded": 295,
  "failed": 5,
  "links": {
    "results": "/v1/jobs/9827/results"
  }
}
```

**Rules**

* **MUST** include `Location`
* **SHOULD** include `Retry-After`
* **MUST** dedupe via `Idempotency-Key` (same key + payload → same job)
* **MUST NOT** embed resource URIs or per-item data in Job
* See **LRO Style Guide** for states & polling

---

8. Headers
----------

| Header | Usage |
| --- | --- |
| **Idempotency-Key** | Required for all unsafe batch POSTs; dedupes retries |
| **Location** | Required for 201/202 |
| **Retry-After** | Recommended for 202/throttling |
| **Cache-Control** | `no-store` for sensitive/transient results |
| **Vary** | Include `Authorization` when results depend on user context |

---

9. Idempotency
--------------

| Requirement | Description |
| --- | --- |
| **MUST** | Deduplicate by `(method, canonical path, Idempotency-Key, request hash)` |
| **MUST** | Return identical outcome on same key + payload |
| **MUST** | Retain key + response metadata ≥ 24h (72h recommended for async) |
| **MUST** | `409 Conflict` if key reused with **different** payload |
| **RECOMMENDED** | `Retry-After` on transient failures |

Async specifics:

* Same key+payload while job queued/in-progress → **return same job** (don’t create a new one)
* After completion → **replay** stored outcome / job status

---

10. OpenAPI Authoring (Using Reusable Schemas)
----------------------------------------------

### 10.1 Batch Endpoint (sync/async)

```
paths:
  /v1/customers/batch:
    post:
      summary: Create multiple customers
      parameters:
        - in: header
          name: Idempotency-Key
          required: true
          schema: { type: string, maxLength: 255 }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: array
              items: { $ref: '#/components/schemas/CustomerCreate' }
      responses:
        '201': { description: All created successfully }
        '200': { description: All items succeeded (non-creation action)' }
        '207':
          description: Partial success (mixed outcomes)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BatchMultiStatusResponseCompact'
        '202':
          description: Accepted for asynchronous processing
          headers:
            Location:
              schema: { type: string, format: uri }
            Retry-After:
              schema: { type: integer, minimum: 1 }
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobStatus'
        '400': { description: Invalid input or schema error }
        '409': { description: Conflict or idempotency-key mismatch }
```

> If you also support **full** results inline for small batches, offer an alternate 207 response using `BatchMultiStatusResponseFull`.

### 10.2 Job Polling

```
paths:
  /v1/jobs/{jobId}:
    get:
      summary: Get job status
      parameters:
        - in: path
          name: jobId
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Job status
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobStatus'
```

### 10.3 (Optional) Exported Full Results

```
paths:
  /v1/exports/{exportId}:
    get:
      summary: Download exported batch results
      parameters:
        - in: path
          name: exportId
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Export ready
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ExportManifest'
        '202': { description: Still generating; retry later }
        '410': { description: Export expired }
```

---

11. Reusable Schemas (Components)
---------------------------------

```
components:
  schemas:

    BatchSummary:
      type: object
      description: Counts for a batch operation
      required: [total, succeeded, failed]
      properties:
        total: { type: integer, minimum: 0 }
        succeeded: { type: integer, minimum: 0 }
        failed: { type: integer, minimum: 0 }

    BatchFailure:
      type: object
      description: Per-item failure record (used in compact and full)
      required: [index, status, code, message]
      properties:
        index:
          type: integer
          minimum: 0
          description: Zero-based position of the item in the submitted array
        id:
          type: string
          nullable: true
          description: Server-assigned or client key if available
        status:
          type: integer
          description: HTTP status for this item (e.g., 409, 412, 422)
        code:
          type: string
          description: Stable application error code (string, not numeric)
        message:
          type: string
          description: Human-readable summary (safe for logs)
        details:
          $ref: '#/components/schemas/ProblemDetails'

    BatchSuccess:
      type: object
      description: Per-item success record (used only in 'full' view)
      required: [index, status]
      properties:
        index:
          type: integer
          minimum: 0
        id:
          type: string
          nullable: true
          description: Created/affected resource identifier if available
        status:
          type: integer
          description: HTTP status for this item (e.g., 200, 201)
        resource:
          type: object
          additionalProperties: true
          description: Optional representation of the created/updated resource (domain-specific)

    BatchMultiStatusResponseCompact:
      type: object
      description: Default compact batch response (summary + failures only)
      required: [summary]
      properties:
        summary:
          $ref: '#/components/schemas/BatchSummary'
        failures:
          type: array
          description: Failed items only (paged if large)
          items: { $ref: '#/components/schemas/BatchFailure' }
        page:
          $ref: '#/components/schemas/PageMeta' # refer to pagination style guide

    BatchMultiStatusResponseFull:
      type: object
      description: Full batch response (summary + successes + failures). Use only for small batches or exports.
      required: [summary]
      properties:
        summary:
          $ref: '#/components/schemas/BatchSummary'
        successes:
          type: array
          description: Successful items (optional and typically limited)
          items: { $ref: '#/components/schemas/BatchSuccess' }
        failures:
          type: array
          description: Failed items (may be paged)
          items: { $ref: '#/components/schemas/BatchFailure' }
        page:
          $ref: '#/components/schemas/PageMeta' # refer to pagination style guide

    JobStatus:
      type: object
      description: Operational status for an asynchronous batch job (no per-item data)
      required: [id, state, submitted_at, progress, links]
      properties:
        id: { type: string }
        type: { type: string, enum: [batch] }
        state: { type: string, enum: [queued, in_progress, completed, failed, canceled] }
        submitted_at: { type: string, format: date-time }
        progress:
          type: object
          required: [total, processed, succeeded, failed]
          properties:
            total: { type: integer, minimum: 0 }
            processed: { type: integer, minimum: 0 }
            succeeded: { type: integer, minimum: 0 }
            failed: { type: integer, minimum: 0 }
        links:
          type: object
          required: [self]
          properties:
            self:
              type: string
              format: uri
              description: Canonical job URI for polling
            results:
              type: string
              format: uri
              nullable: true
              description: Optional link to compact or exported results (no per-item payload inline)
        idempotency_key:
          type: string
          description: Echo of the submission key to support replay semantics
        request_hash:
          type: string
          description: Hash/fingerprint of original request body for conflict detection
        error:
          $ref: '#/components/schemas/ProblemDetails' # refer to LPDP  

    ExportManifest:
      type: object
      description: Descriptor for large exported batch results
      required: [id, state]
      properties:
        id: { type: string }
        state: { type: string, enum: [pending, generating, completed, failed, expired] }
        file_type: { type: string, enum: [application/jsonl, application/json, text/csv] }
        size_bytes: { type: integer, minimum: 0 }
        expires_at: { type: string, format: date-time }
        download_url: { type: string, format: uri }
        error:
          $ref: '#/components/schemas/ProblemDetails'
```

---

12. Governance Validation Checklist
-----------------------------------

* ✅ Latency boundaries: ≤ 30 s p95 / ≤ 60 s absolute (else LRO)
* ✅ Status codes: 200 / 201 / 207 / 202 only
* ✅ 207 response uses **BatchMultiStatusResponseCompact** (or **…Full** when allowed)
* ✅ Idempotency: `Idempotency-Key` required (async); recommended (sync)
* ✅ OAS coverage: 201 + 207 + 202 documented; job uses **JobStatus** schema
* ✅ Spectral rules: enforce headers, responses, and schemas