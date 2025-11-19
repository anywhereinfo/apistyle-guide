# API Standard: Long-Runing Operations (LRO)

> Confluence Page ID: 675812048943, Version: 3

1. Summary
----------

This document defines the standard asynchronous Long-Running Operation (LRO) pattern for all Lumen APIs. This pattern **must** be used when an API operation cannot be completed within a standard, synchronous HTTP request-response cycle (e.g., > 2-3 seconds).

Holding an HTTP connection open is fragile and resource-intensive. The LRO pattern solves this by immediately acknowledging the client's request, returning a "job" or "operation" resource the client can poll for status, and decoupling the request's acceptance from its final completion.

This pattern provides:

* Deterministic asynchronous behavior
* Standard client polling and progress tracking
* Reliable retries via idempotency
* A clear separation of **resource state** vs. **operation state**
* Structured error reporting (per our **LPDP-Mini v1.0** profile) for failures

---

2. Why This Pattern Matters for Lumen
-------------------------------------

Many of our core business processes are complex and not instantaneous. Synchronous APIs are unsuitable and unreliable for these tasks. This pattern is the standard for:

* **Service Provisioning:** Activating a new network service involves orchestration across multiple systems and can take minutes or hours.
* **Wholesale & Bulk Operations:** A partner submitting a POST for a wholesale order containing 5,000 circuits cannot be handled in a single synchronous transaction.
* **Network Decommissioning:** Tearing down a complex service requires a series of ordered steps that must be tracked.
* **Large Data Exports:** Generating a large-scale report (e.g., "export all billing data for the last 12 months").

---

3. The LRO Pattern Explained
----------------------------

### 3.1 Workflow

|  |  |  |
| --- | --- | --- |
| **Step** | **Description** | **Example** |
| 1. **Initiate** | Client sends request (POST, PUT, PATCH, DELETE) to start a long-running transaction. | `DELETE /ports/{id}` |
| 2. **Accept** | Server validates, starts background job; responds 202 Accepted. | Headers: `Location: /operations/{opId}` |
| 3. **Poll** | Client polls `GET /operations/{operationId}` until terminal. | Payload: status, percent, error? |
| 4. **Result** | On success, link to result; on failure, return Problem Details. | `resource.href: "/ports/123"` |
| 5. **Cleanup** | Operation record is purged after a defined TTL. | `GET /operations/{opId}` → 404 |

### 3.2 Mandatory Operation Resource Schema

The `GET /operations/{operationId}` endpoint is the "source of truth" for the job. It **must** return a response body adhering to the following fixed schema.

Here is an **example** of a `FAILED` operation response:

```
{
  "operationId": "a1b2-c3d4-e5f6",
  "status": "FAILED",
  "percent": 33,
  "startedAt": "2025-10-29T16:00:00Z",
  "completedAt": "2025-10-29T16:00:15Z",
  "metadata": {
    "message": "Failed during partner verification.",
    "step": "PARTNER_VERIFY"
  },
  "resource": null,
  "error": {
    "title": "Partner timeout",
    "detail": "The operation failed because a downstream partner did not respond.",
    "errors": [
      {
        "code": "PARTNER_TIMEOUT",
        "message": "Partner did not confirm teardown within SLA",
        "meta": {
          "traceId": "traceId-abc123"
        }
      }
    ]
  }
}
```

#### 3.2.1 Formal OAS 3.1 Schema Definitions

```
# These schemas MUST be used for the GET /operations/{id} response
components:
  schemas:
    Operation:
      type: object
      required: [operationId, status, startedAt]
      properties:
        operationId:
          type: string
          description: Unique identifier for the operation.
          example: "a1b2-c3d4-e5f6"
        status:
          type: string
          description: The current state of the operation.
          enum: [PENDING, RUNNING, SUCCEEDED, FAILED, CANCELLED]
        percent:
          type: integer
          format: int32
          description: Estimated progress percentage (0-100).
          example: 33
        startedAt:
          type: string
          format: date-time
          description: UTC timestamp when the operation was accepted.
          example: "2025-10-29T16:00:00Z"
        completedAt:
          type: string
          format: date-time
          description: UTC timestamp when the operation entered a terminal state.
          example: "2025-10-29T16:00:30Z"
        resource:
          type: object
          description: Present on SUCCEEDED for POST/PUT/PATCH.
          properties:
            id:
              type: string
              description: The ID of the created/modified resource.
              example: "port-1234"
            href:
              type: string
              format: uri
              description: The full URI to GET the resource.
              example: "/v1/ports/1234"
        error:
          description: Present on FAILED. Follows the LPDP Problem Details profile.
          $ref: '#/components/schemas/LpdpProblem'
        metadata:
          type: object
          description: >
            Service-specific metadata. This object contains rich, real-time 
            status information.
          properties:
            message:
              type: string
              description: The current human-readable status message.
              example: 'Step 2 of 4: Verifying configuration...'
            step:
              type: string
              description: A machine-readable code for the current sub-state.
              example: 'VERIFYING_CONFIG'
          additionalProperties: true
    
    # ------ LPDP Problem Details Profile Schemas ------
    LpdpProblem:
      type: object
      additionalProperties: false
      description: Minimal Problem-Details envelope (RFC 9457-compliant)
      properties:
        title:
          type: string
          description: Short, human-readable summary.
        detail:
          type: string
          description: Optional longer explanation.
        errors:
          type: array
          minItems: 1
          items: { $ref: '#/components/schemas/LpdpError' }
      required: [ title, errors ]

    LpdpError:
      type: object
      additionalProperties: false
      description: Individual error item.
      properties:
        code:
          type: string
          description: Stable, machine-readable identifier (string form preferred).
        message:
          type: string
          description: Human-readable explanation of the issue.
        meta:
          type: object
          description: >
            Optional unregulated extension for machine-readable context.
            Lumen does not prescribe its structure. Teams may define custom
            schemas if needed.
          additionalProperties: true
      required: [ code, message ]
```

### 3.3 Industry Standard Schemas (Google & Microsoft)

Major cloud providers use fixed schemas for operations response. This predictability is why the pattern is successful.

* **Google Cloud (AIP-151):** Uses a fixed schema for `google.longrunning.Operation`. It has standard fields: `name` (the op ID), `done` (boolean), a `result` field (for `error` or `response`), and a `metadata` object. This `metadata` object is used to hold the exact kind of qualitative, real-time status (e.g., 'Verifying configuration...') that our standard now requires.
* **Microsoft Azure:** Also uses a fixed schema. The polling URL (from `Location` or `Azure-AsyncOperation` headers) returns a standard object with fields like `status`, `id`, `startTime`, `endTime`, `percentComplete`, and `error`.

Our schema in 3.2 is an industry-aligned "best-of-both" approach, standardizing on the `operationId`, `status`, and our `LpdpProblem` error structure.

### 3.4 HTTP Semantics by Use Case

|  |  |  |
| --- | --- | --- |
| **Use Case** | **URI** | **Expected Codes** |
| Async Create | POST /resource | 201 (sync) or 202 (async) |
| Async Replace/Update | PUT /resource/{id} | 200/204 or 202 |
| Async Partial Update | PATCH /resource/{id} | 200 or 202 |
| Async Delete | DELETE /resource/{id} | 204 (sync) or 202 (async) |
| **Poll for Status** | **GET /operations/{id}** | **200 (status) → 404 after TTL** |
| List Operations | GET /operations | 200 |
| Request Cancellation | DELETE /operations/{id} | 202 |

### 3.5 State Machine

#### 3.5.1 State Definitions

* **RUNNING:** The operation is actively being processed.
* **SUCCEEDED:** The operation completed successfully.
* **FAILED:** The operation failed due to an error. The `error` field in the response **must** be populated.
* **CANCELLED:** The operation was successfully cancelled by a client request (`DELETE /operations/{id}`) and did not complete.

#### 3.5.2 State Transitions

Operation states are terminal. Once an operation is SUCCEEDED, FAILED, or CANCELLED, its state **must not** change.

* RUNNING → SUCCEEDED, FAILED, CANCELLED
* SUCCEEDED → (No transitions)
* FAILED → (No transitions)
* CANCELLED → (No transitions)

### 3.6 Client Polling Strategy

Clients **must** implement a robust polling strategy.

* **Backoff:** Clients **should** use **exponential backoff with jitter** to avoid overwhelming the server (e.g., 1s, 2s, 4s, 8s... capped at 30-60s).
* **Retry-After Header:** If the `202 Accepted` or `200 OK` (on a poll) response includes a `Retry-After` header, the client **must** respect it. This is the server's primary backpressure mechanism.
* **Rate Limiting:** Polling endpoints (`GET /operations/{id}`) **are** subject to API rate limits. Excessive polling (e.g., ignoring backoff) **will** result in a `429 Too Many Requests` error, which should also include a `Retry-After` header.
* **Header Format:** Per RFC 9110, `Retry-After` can be seconds (`Retry-After: 30`) or an HTTP-date (`Retry-After: Wed, 29 Oct 2025 16:30:00 GMT`). Clients **must** support both formats.

---

4. Explaining the Pattern: Operation Polling vs. Resource Polling
-----------------------------------------------------------------

This section details *why* this LRO pattern (polling `/operations`) is the standard and *why* the common anti-pattern (polling the resource itself) is disallowed.

### 4.1 The Correct Pattern: Polling the Operation Resource

The core of this pattern is the separation of the *job's* state from the *resource's* state. The `GET /operations/{id}` endpoint is the **only** source of truth for the *job*.

This section defines what a client should expect from the *business resource* (`GET /resource/{id}`) while an operation is in progress.

#### 4.1.1 Behavior of `GET /resource` During an LRO

This is the standard for what a `GET` on the resource itself must return while a DELETE operation is in progress.

1. **While In-Progress (Job status: "RUNNING"):**

   * The `GET /resource/{id}` endpoint **must** return `200 OK` with the full, normal resource representation.
   * **Caching:** The response **should** include `Cache-Control: no-store` or `max-age=10`.
   * **(Optional) Discoverability:** The response **may** include a `Link: </operations/{operationId}>; rel="operation"`.
   * **(Optional) Lifecycle State:** You **may** add a *minimal, read-only* flag: `"lifecycleStatus": "DELETING"`. Clients **must not** poll this field for completion.
2. **After Completion (Job status: "SUCCEEDED"):**

   * `GET /resource/{id}` **must** return `404 Not Found` or `410 Gone`.

#### 4.1.2 Policy: 410 Gone vs. 404 Not Found

* `410 Gone` is **preferred** for a short "tombstone" period (e.g., 1-24 hours) after a successful DELETE. It asserts the resource *did* exist. The response body **may** include an `LpdpProblem` body with the `deletedAt` time and `operationId` in the `meta` block.
* After the tombstone TTL, the server **should** revert to returning a standard `404 Not Found`.

#### 4.1.3 Concurrency During Deletion

While a resource is being deleted (LRO status: "RUNNING"), all access to that resource must be handled explicitly.

|  |  |  |
| --- | --- | --- |
| **Request** | **Expected Response** | **Error Code (code)** |
| `GET /resource/{id}` | **200 OK** | N/A |
| `PUT` or `PATCH /resource/{id}` | **409 Conflict** or **423 Locked** | `RESOURCE_LOCKED` |
| `DELETE /resource/{id}` (duplicate) | **202 Accepted** | N/A (Returns original `Location` header) |
| `POST` (dependent sub-resources) | **409 Conflict** | `OPERATION_IN_PROGRESS` |

### 4.2 The Anti-Pattern (Disallowed): Polling the Business Resource

A common but flawed alternative is to return `202 Accepted` and then have the client poll `GET /resource/{id}`, waiting for its state to change (e.g., for `status` to become `DELETED` or for it to return `404`). This anti-pattern is **disallowed** for the following reasons.

#### 4.2.1 Flaw: Poor Observability

* **Problem:** The LRO pattern provides a "job log" (the operation resource). The anti-pattern only provides the final state of the resource. You cannot debug a failed process by only looking at the final object.
* **Example (Anti-Pattern):**

  + A `DELETE /ports/123` fails. The team adds a `status: "DELETE_FAILED"` field to the `GET /ports/123` response.
  + **The Issue:** An operations team tries to debug. They have no idea *why* it failed (the error message doesn't belong on the port model). They don't know *when* the delete was attempted, *how long* it ran before failing, or *which* trace ID to look up. They can't answer: "What is our p95 time-to-delete?" or "What is our error rate for port deletions?"
* **Example (LRO Pattern):**

  + The `GET /operations/op-abc` response shows `status: "FAILED"`, `startedAt: "...", completedAt: "..."`, and a full `error` object with `code: "PARTNER_TIMEOUT"` and `meta: { "traceId": "..." }`.
  + **The Benefit:** The operations team can now build a dashboard. They can query `GET /operations?status=FAILED` to find all failures. They can aggregate `errors[0].code` to see the top-most failure reason. They can compute exact durations (`completedAt` - `startedAt`) to get p95 latency SLOs.

#### 4.2.2 Flaw: Security Risk & Information Leaks

* **Problem:** The resource (e.g., `/ports/{id}`) and the operation (e.g., `/operations/{op-abc}`) often have different security audiences. The resource is for *anyone* who can read it. The operation's *result* is often only for the *initiator*.
* **Example (Anti-Pattern):**

  + To "fix" the observability flaw, a team adds an `errorInfo: "Downstream partner 'XYZ-Corp' timed out (trace-id: 99a-88b)"` field to the `GET /ports/123` response.
  + **The Risk:** A low-privilege, read-only user from a *different tenant* (who just has read-access to the port) now polls the resource. They have just learned:

    1. A delete was attempted (which they shouldn't know).
    2. The internal name of our downstream partner (`XYZ-Corp`).
    3. An internal `trace-id` they can use for other attacks.
  + This is a major information leak.
* **Example (LRO Pattern):**

  + The read-only user polls `GET /ports/123` and sees the normal `200 OK` response. They have no idea a delete is in progress.
  + The initiator (with correct auth) polls `GET /operations/op-abc` and gets the `FAILED` status and error.
  + The read-only user tries to access `GET /operations/op-abc` and gets a `404 Not Found` (or `403 Forbidden`). The sensitive error details are protected.

#### 4.2.3 Flaw: High Server Load

* **Problem:** A `GET` on a business resource (e.g., `/ports/{id}`) is often a "heavy" query. A `GET` on an operation is a "light" query. Forcing clients to poll a heavy endpoint creates unnecessary load.
* **Example (Anti-Pattern):**

  + A client polls `GET /ports/123` every 5 seconds. This endpoint is "heavy" because it's designed for UI/customer use. To build the response, the server must:

    1. Join 5 tables in the database (port details, customer info, location data, child circuits, config).
    2. Call 2-3 downstream microservices.
    3. Serialize a large 100KB JSON payload.
  + A thousand clients polling this every 5 seconds creates massive read load on the main business database and downstream services.
* **Example (LRO Pattern):**

  + A client polls `GET /operations/op-abc` every 5 seconds. This endpoint is "light" because it's *designed* for polling.
  + The server does one thing: `SELECT * FROM operations_table WHERE id = 'op-abc'` (or even better, a Redis `GET`).
  + It returns a tiny 1KB JSON payload (`{ "status": "RUNNING", "percent": 30 }`).
  + This has almost zero impact on the core system. Furthermore, the server can use the `ETag` header. If the status hasn't changed, it returns `304 Not Modified` (an empty body), saving bandwidth and processing.

---

5. Applying the Pattern to Workflows
------------------------------------

### 5.1 Individual Example (DELETE)

1. Client: `DELETE /connections/con-123`
2. Server: → `202 Accepted`, `Location: /operations/op-abc`
3. Client: `GET /operations/op-abc` (polls until terminal)
4. Server: → `200 OK`, `{"status": "FAILED", "error": {...}}`

### 5.2 Bulk / Wholesale Example

This LRO pattern is the *same* one used for bulk interfaces; no new pattern is needed. The operation resource simply contains a list of sub-operations. (Note: Detailed bulk API design is a separate topic).

* **Partial Failure Policy:** The overall operation `status` reflects the batch:

  + `SUCCEEDED`: All sub-operations succeeded.
  + `FAILED`: *Any* sub-operation failed.
* **Client Resubmission:** The client is responsible for inspecting the `subOperations` array in the response and re-submitting *only* the items that `FAILED`, using new `Idempotency-Key` headers for the new batch request.
* **Limits:** Services **must** define and enforce:

  + A maximum number of items per bulk request (e.g., 200).
  + A per-tenant concurrent operation limit (e.g., 10 active `RUNNING` operations).

---

6. Governance Requirements
--------------------------

All LRO-enabled APIs **must** adhere to these standards.

### 6.1 Idempotency

* **Requirement:** All LRO-initiating requests (POST, PUT, PATCH, DELETE) **must** require an `Idempotency-Key` header.
* **Scope:** The key's uniqueness is scoped *per tenant* and *per operation*.
* **Lifetime:** The server **must** cache the idempotency key and its result for a minimum of 24-72 hours.
* **Behavior:**

  + If a request is retried *while the operation is active* (RUNNING), the server **must** return the `202 Accepted` with the *same* `Location` header.
  + If a request is retried *after the operation is terminal* (SUCCEEDED/FAILED), the server **must** return the final stored result (e.g., `200 OK` with the final op body, or `410 Gone` if the resource was deleted) until the idempotency key expires.

### 6.2 Cancellation

* **Mechanism:** Clients **may** request cancellation by sending `DELETE /operations/{id}`.
* **Allowable States:** Cancellation can only be attempted on operations in `RUNNING` state.
* **Response:**

  + `202Accepted`: The cancellation request was accepted (best-effort). The client must poll the operation, which will eventually transition to `CANCELLED` or `FAILED`.
  + `409 Conflict`: The operation cannot be cancelled. The server must return an application/problem+json response body. The detail or message should state: “The operation is already in a terminal state (SUCCEEDED, FAILED, CANCELLED) and cannot be cancelled, or cancellation is not supported for this RUNNING operation.”
  + `404 Not Found`: The operation is unknown or has expired.

### 6.3 Security & Tenancy

* **Visibility:** An operation resource `GET /operations/{id}` **must** be protected by the same authorization rules as the resource it's acting upon. An operation **must only** be visible to the initiator, resource owners, or tenant admins.
* **Listing:** `GET /operations` **must** be filtered by tenancy. A user must **never** see operations from another tenant.
* **Data Redaction:** Error messages (`message` and `detail`) **must not** contain PII, secrets, or internal stack traces.

### 6.4 Rate Limiting & Quotas

* **Requirement:** All LRO endpoints **must** be protected by rate limits.
* **Limits:** Services **must** enforce limits on:

  1. Max concurrent `RUNNING` operations per tenant.
  2. Max items per bulk request.
  3. Polling frequency.
* **Response:** When a limit is exceeded, the server **must** return `429 Too Many Requests` with a `Retry-After` header.

### 6.5 Error Taxonomy (Problem Details)

All errors **must** use `application/problem+json` and our `LPDP-Mini v1.0` profile. The `code` is the canonical identifier.

|  |  |  |
| --- | --- | --- |
| **Error Code (code)** | **HTTP Status** | **Meaning** |
| `OPERATION_EXPIRED` | 404 | Polled an operation ID that is past its TTL. |
| `RESOURCE_LOCKED` | 409 / 423 | PUT/PATCH failed, delete in progress. |
| `OPERATION_IN_PROGRESS` | 409 | POST (sub-resource) failed, parent delete in progress. |
| `VALIDATION_ERROR` | 400 | The initial request body failed validation. |
| `PARTNER_TIMEOUT` | 504 | A downstream system failed to respond in time. |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests or concurrent operations. |

### 6.6 Caching & Headers

* `GET /operations/{id}`: **Must** return `Cache-Control: no-store`. **Should** support `ETag` and `If-None-Match` to allow clients to poll efficiently and receive a `304 Not Modified` if the status hasn't changed.
* `GET /resource/{id}` (during delete): **Must** return `Cache-Control: no-store` or a very small `max-age` (e.g., 10 seconds).

### 6.7 Operation Resource Lifecycle

|  |  |  |
| --- | --- | --- |
| **Phase** | **Description** | **Response Code** |
| **Active** | Operation created (PENDING, RUNNING). | 200 OK |
| **Terminal** | Operation reaches SUCCEEDED, FAILED, or CANCELLED. | 200 OK (for the duration of the TTL) |
| **Expired (purged)** | Operation record past retention TTL (7-30 days). | 404 Not Found with `code: "OPERATION_EXPIRED"` |
| **Manually deleted** | Client performs `DELETE /operations/{id}`. | 202 Accepted, then 404 Not Found |

---

7. Industry References (for Architects)
---------------------------------------

* **Google Cloud:** [AIP-151 – Long-Running Operations](https://google.aip.dev/151)
* **Microsoft Azure:** [Asynchronous Operations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/async-operations)
* **RFC 9457:** [Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)

---

8. OAS 3.1 Snippets (For Implementation)
----------------------------------------

### 8.1 LRO Initiator (e.g., Async DELETE

```
paths:
  /resources/{id}:
    delete:
      summary: Deletes a resource asynchronously
      parameters:
        - in: header
          name: Idempotency-Key
          required: true
          schema: { type: string, minLength: 8 }
      responses:
        '202':
          description: Accepted; poll the operation resource for status.
          headers:
            Location:
              schema: { type: string, format: uri }
              description: The URL of the /operations/{operationId} resource.
            Retry-After:
              schema: { type: integer, example: 10 }
        '404': # ... standard errors
        '409': # ... concurrency/lock error
        '429':
          description: Rate limit exceeded (e.g., too many concurrent ops).
          headers:
            Retry-After: { schema: { type: integer, example: 60 } }
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/LpdpProblem' }
        '503':
          description: Service Unavailable (e.g., job queue is full).
          headers:
            Retry-After: { schema: { type: integer, example: 120 } }
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/LpdpProblem' }
```

### 8.2 Get Operation Status (Polling)

```
paths:
  /operations/{operationId}:
    get:
      summary: Gets the status of a long-running operation.
      parameters:
        - in: header
          name: If-None-Match
          schema: { type: string }
          description: ETag for a 304 Not Modified response.
      responses:
        '200':
          description: Operation status.
          headers:
            ETag: { schema: { type: string } }
            Retry-After: { schema: { type: integer, example: 10 } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Operation' }
        '304':
          description: Not Modified. Operation status has not changed.
        '404':
          description: Operation expired or not found.
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/LpdpProblem' }
```

### 8.3 Cancel Operation

```
paths:
  /operations/{operationId}:
    delete:
      summary: Requests cancellation of an operation.
      responses:
        '202':
          description: Cancellation request accepted (best-effort).
        '404':
          description: Operation not found or expired.
        '409':
          description: Conflict. Operation is already in a terminal state.
```

---

9. Implementation & Observability
---------------------------------

### 9.1 Time & Consistency

* **Timestamps:** All timestamps (`startedAt`, `completedAt`) **must** be in UTC and formatted as **ISO-8601**.
* **Consistency:** A small delay (eventual consistency) is permitted between the LRO terminal state (`SUCCEEDED`) and the resource state (`GET /resource/{id}` returning `404`). The `GET /operations/{id}` endpoint is **always** the source of truth for the *job's* status.

### 9.2 Observability & SLOs

* **Required Metrics:** Services **must** emit metrics for:

  + `operation_started_total` (Counter, tagged by type, tenant)
  + `operation_completed_total` (Counter, tagged by type, tenant, status: [SUCCEEDED, FAILED, CANCELLED])
  + `operation_duration_seconds` (Histogram, tagged by type, status)
* **Tracing:** The `operationId` **must** be correlated with the distributed trace context (e.g., in `error.errors[0].meta.traceId`).
* **Target SLOs:** Teams **must** define and publish SLOs for LROs (e.g., p95 completion for DELETE < 5 minutes).

---

10. Optional Extensions: Event-Based Notifications
--------------------------------------------------

Polling is the default, but services **may** offer event-based notifications for advanced clients.

* **Methods:** Server-Sent Events (SSE), Webhooks, or Kafka topics.
* **Payload:** The event payload **should** match the Operation schema.
* **Webhooks:** If webhooks are used, the service **must** provide a way for clients to register a callback URL and **must** use an HMAC signature (e.g., in `Webhook-Signature` header) for verification.