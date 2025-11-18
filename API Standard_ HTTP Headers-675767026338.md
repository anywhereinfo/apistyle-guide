# API Standard: HTTP Headers

> Confluence Page ID: 675767026338, Version: 3

none

1. Naming & Formatting
----------------------

* **MUST** use lowercase kebab-case ASCII for all HTTP header names.

  + ✅ `idempotency-key`, `traceparent`, `content-type`, `if-match`
  + ❌ `X_Lumen_Entity_ID`, `X-Lumen-Entity-Id`, `RequestId`, non-ASCII
* **MUST NOT** use the legacy `x-` prefix for new headers (per RFC 6648).
* **MUST** document any legacy interoperability exceptions in the API's local specification.

---

2. Header Registry (Allow-list)
-------------------------------

To ensure client simplicity and enforceable governance, APIs **MUST** restrict headers to the following allow-list. Additions require a formal exception via the API Governance process.

### 2.1 Standard Headers (Allowed)

* **Content & Negotiation:** `accept`, `content-type`, `content-length`, `content-encoding`
* **Authentication:** `authorization`
* **Caching:** `cache-control`, `etag`, `last-modified`, `expires`, `vary`
* **Conditional Requests:** `if-match`, `if-none-match`, `if-modified-since`, `if-unmodified-since`
* **Conformance & Links:** `link`, `location`, `retry-after`, `date`
* **Range (when applicable):** `range`, `content-range`
* **Tracing & Correlation:** `traceparent`, `tracestate` (W3C)

Any new custom headers **MUST** be defined with a purpose, schema, max length, examples, and security considerations before being added to this list.

---

3. Authentication & Security
----------------------------

* **MUST** use `authorization: Bearer <access_token>` for OAuth2/JWT authentication.
* **MUST NOT** introduce proprietary authentication headers without a formal review from the Security team.
* **MUST NOT** place secrets or PII in headers or URLs. Always use request bodies over TLS.
* **SHOULD** include the `traceparent` in all responses for correlation.

---

4. Content Negotiation & Errors
-------------------------------

* **Clients SHOULD** send `accept: application/json`. Clients must only request media types the API supports.
* **Servers MUST** set `content-type: application/json` for all JSON response bodies.
* **Servers MUST** use `content-type: application/problem+json` and conform to **RFC 9457 (Problem Details)** for all 4xx and 5xx error responses.
* **Servers MUST** return a `406 Not Acceptable` if the client's `accept` header does not match a supported media type.

---

5. Caching & Conditional Requests
---------------------------------

* `ETag`: Servers **MUST** include an `ETag` header on all `GET` responses for cacheable resources. This is preferred over `Last-Modified`.
* `If-None-Match`: Servers **MUST** honor the `If-None-Match` request header, returning a `304 Not Modified` if the `ETag` matches.
* `If-Match`: Servers **MUST** use the `If-Match` header to enable optimistic concurrency control for `PATCH`, `PUT`, or `DELETE` requests.
* `Cache-Control`: **MUST** be set according to the resource's caching policy (e.g., `no-store` for sensitive data, `max-age` for public metadata).
* `Location`: **MUST** be returned with a `201 Created` response, containing the canonical URI of the newly created resource.

---

6. Observability & Correlation
------------------------------

* **MUST** implement W3C Trace Context (`traceparent`, `tracestate`).
* **Clients SHOULD** generate and send the `traceparent` header.
* **Gateways MUST** ensure every request has a `traceparent`. If a client request is missing the `traceparent` header, the gateway **MUST** generate one before forwarding the request to internal services.
* **Servers MUST** propagate the `traceparent` header across all internal service boundaries.
* **Servers MUST** include the `traceparent` value in all error responses for end-to-end tracing.

---

7. Rate Limiting
----------------

Gateway should handle ratelimiting and use below headers.

* **SHOULD** use the IETF-standardized `RateLimit` response fields:

  + `ratelimit-limit`: The quota for the time window.
  + `ratelimit-remaining`: The number of requests remaining.
  + `ratelimit-reset`: The remaining time in seconds until the quota resets.
* **MUST** include the `Retry-After` header (in seconds) when returning a `429 Too Many Requests` or `503 Service Unavailable` status due to throttling.

---

8. Size Constraints
-------------------

* **Size Limits:** Header values **MUST** be ASCII. Total header size **SHOULD** remain within gateway limits (e.g., 8KB).

---

9. Governance & Linting (Spectral Snippets)
-------------------------------------------

The following Spectral rules SHOULD be used to enforce these header standards.

### Header Name Format

```
rules:
  lumen-header-name-format:
    given: "$..parameters[?(@.in=='header')].name"
    then:
      function: pattern
      functionOptions: { match: "^[a-z][a-z0-9-]*$" }
```

---

10. Change Control
------------------

* Additions to the allow-list or new custom headers **MUST** go through API Governance with Security review.