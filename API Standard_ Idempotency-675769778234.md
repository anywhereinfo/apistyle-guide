# API Standard: Idempotency

> Confluence Page ID: 675769778234, Version: 1

none

1. What is Idempotency?
-----------------------

An **idempotent** operation is an operation that can be performed multiple times without changing the result beyond the initial application.

In the context of a REST API, this means that making the same request multiple times (e.g., due to a network error and retry) will produce the same result and system state as making it just once.

2. Why is Idempotency a Standard?
---------------------------------

Idempotency is not an optional feature; it is a **critical component of a robust and reliable API platform.** Clients (mobile apps, web frontends, other services) operate on unreliable networks. A client may send a request, the server may process it, but the response may be lost before it reaches the client.

Without idempotency, this client has no safe way to recover.

* **The Risk:** The client doesn't know if the request succeeded.If it retries a `POST /users` request, it may create two identical users. If it retries a `POST /charges` request, it may charge the customer twice.
* **The Solution:** This standard ensures all endpoints are safe to retry, protecting our systems and our customers from data duplication and unintended side effects.

3. Idempotency by HTTP Method
-----------------------------

This standard defines the required behavior for each HTTP method.

|  |  |  |
| --- | --- | --- |
| **Method** | **Idempotent by Standard?** | **Notes** |
| `GET` | **MUST** | `GET` requests are *safe* (they don't change state) and must be idempotent. |
| `PUT` | **MUST** | A `PUT` request replaces a resource. `PUT /users/123` twice is the same as once. |
| `DELETE` | **MUST** | A `DELETE` request deletes a resource. The first call deletes it (returning `204`). The second call (and all subsequent) should return `404 Not Found`. The system state remains the same ("deleted"). |
| `PATCH` | **MUST** | A `PATCH` operation must be idempotent. Server logic must ensure that re-applying the same patch does not produce a different result. (e.g., `PATCH /item/123 {"name": "new"}` twice is fine). |
| `POST` | **MUST** support idempotency | `POST` is not naturally idempotent. This standard **requires** all `POST` endpoints to support an idempotency-key mechanism to *make* them idempotent. |

---

4. `POST` Idempotency: The `idempotency-key` Header
---------------------------------------------------

Since `POST` requests create new resources, they are the highest risk for duplication. To mitigate this, all `POST` endpoints that create resources **MUST** support idempotency via a request header.

### The Standard

`POST` endpoints **MUST** honor the `idempotency-key` header.

### Client Responsibility

When a client sends a `POST` request, it **SHOULD** generate a unique key (e.g., a **UUID**) and send it in the `idempotency-key` header.

```
HTTP
```

```
POST /v1/customers
idempotency-key: a8b4-12a8-8f81-9b48-18e0-128a
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane.doe@example.com"
}
```

If the client suffers a network error and needs to retry, it **MUST** send the *exact same* `idempotency-key` with the retry request.

### Server Responsibility

The server **MUST** perform the following logic for any `POST` request containing an `Idempotency-Key`:

1. **Check for Key:** The server checks an internal cache (e.g., Redis) for this key.
2. **Key Not Found (New Request):**

   * This is the first time this request has been seen.
   * The server processes the request as normal (e.g., creates the customer in the database).
   * The server then **stores the HTTP response** (e.g., the `201 Created` status and the JSON body) in the cache, using the `Idempotency-Key` as the cache key.
   * The server returns the response to the client.
3. **Key is Found (Retry):**

   * This is a retry. The server **MUST NOT** re-process the request.
   * It immediately fetches the *original, saved response* from the cache.
   * It returns that exact same response to the client.

This flow guarantees that a client can safely retry a `POST` without fear of creating duplicate resources.

### Key Details

* **Key Format:** `idempotency-key` values **SHOULD** be a **UUID** to ensure uniqueness.
* **Key Lifecycle:** The server **SHOULD** store idempotency keys for at least **24 hours** to allow clients ample time to retry failed requests.