# API Standard: Caching & Concurrency with ETag

> Confluence Page ID: 675767910645, Version: 1

none

1. What is an ETag?
-------------------

An **ETag (Entity Tag)** is an HTTP header that provides an identifier for a specific version of a resource.

Think of it as a "**fingerprint**" or "**version hash**" for a resource's state. When a resource (like a JSON object for a user) is created or modified, the server **MUST** generate a new, unique ETag value for it.

Clients **MUST** treat the ETag as an opaque string. They should store it, but never try to parse or interpret its contents.

Example ETag Response Header:

```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "abc-123-xyz-789"
{
  "id": "user-42",
  "name": "Jane Doe"
}
```

---

1.1. Standard Mandate: Strong vs. Weak ETags
--------------------------------------------

While all ETags serve a purpose, our standard **MANDATES** the use of **Strong ETags** because they are required for reliable concurrency control.

|  |  |  |
| --- | --- | --- |
| **Feature** | **Strong ETag (Required for our API)** | **Weak ETag (Not allowed for Concurrency)** |
| **Primary Use** | **Concurrency Control** (Optimistic Locking) and reliable caching. | Caching only (when byte-for-byte fidelity is not critical). |
| **Guarantee** | The resource is **byte-for-byte identical**. Any change, no matter how small, **MUST** generate a new ETag. | The resource is **semantically equivalent**, but minor presentation changes (like a footer timestamp) may not generate a new ETag. |
| **Format** | Enclosed in standard quotes (no prefix). `"v1-hash-xyz"` | Preceded by `W/`. `W/"v1-hash-xyz"` |

**Crucial Note:** Only the **Strong ETag** provides the guarantee necessary for the server to safely allow a destructive update (`PUT`, `PATCH`, `DELETE`). The server **MUST NOT** accept an `If-Match` header containing a Weak ETag.

---

2. Why This Standard is Required
--------------------------------

ETags are not optional. They are our platform's standard for solving two critical, high-level API problems:

* **Efficient Caching:** Prevents clients from re-downloading data they already have, saving bandwidth and improving client-side performance.
* **Concurrency Control:** Prevents the "lost update" problem, ensuring that clients don't accidentally overwrite each other's changes.

This guide defines the two ways ETags **MUST** be used.

---

3. Use Case 1: Caching (Conditional GET)
----------------------------------------

This flow prevents a client from re-downloading a resource that has not changed.

### The Flow

1. Server **MUST** Send ETag: All `GET` responses for a single, cacheable resource **MUST** include an `ETag` header.
2. Client **SHOULD** Store ETag: The client application (web app, mobile app) **SHOULD** cache the resource's body and its `ETag` value.
3. Client **MUST** Send If-None-Match: The next time the client wants to `GET` that same resource, it **MUST** include the stored ETag value in an `If-None-Match` request header.
4. Server **MUST** Validate: The server receives the request and compares the `If-None-Match` value with the resource's current ETag.

### The Result

* If the ETags **MATCH**: The client's version is up-to-date. The server **MUST** stop and return an empty **304 Not Modified** response. This saves the server from computing the response and saves bandwidth by not sending the data again.
* If the ETags **DO NOT MATCH**: The resource has changed. The server **MUST** proceed as normal, returning a **200 OK** with the full, new resource and the new `ETag`.

---

4. Use Case 2: Concurrency Control (Optimistic Locking)
-------------------------------------------------------

This is the most critical function of ETags. It prevents clients from overwriting each other's work.

### The "Lost Update" Problem

1. Alice `GET` /users/123. She receives the user's data and the ETag: `"v1"`.
2. Bob also `GET` /users/123. He receives the same data and the same ETag: `"v1"`.
3. Alice sends a `PATCH` to change the user's name. Her request succeeds. The server updates the user and generates a new ETag: `"v2"`.
4. Bob now sends a `PATCH` to change the user's phone number. His request is based on the old `"v1"` data. Bob's update overwrites Alice's name change. Alice's work is lost.

### The ETag Solution (Mandatory)

All state-changing methods (`PUT`, `PATCH`, `DELETE`) **MUST** be protected against this.

1. Server **MUST** Enforce If-Match: Servers **MUST** require the `If-Match` header on all `PUT`, `PATCH`, and `DELETE` requests.
2. Client **MUST** Send If-Match: A client **MUST** include the ETag of the resource it thinks it's updating in the `If-Match` header.

### The Correct Flow

1. Alice `GET` /users/123. She receives ETag: `"v1"`.
2. Bob `GET` /users/123. He also receives ETag: `"v1"`.
3. Alice sends `PATCH` /users/123 with the header `If-Match: "v1"`.
4. The server checks: `If-Match` value (`"v1"`) matches the current ETag (`"v1"`).
5. The server accepts the update, processes it, and generates a new ETag: `"v2"`.
6. Bob now sends `PATCH` /users/123 with his header `If-Match: "v1"`.
7. The server checks: `If-Match` value (`"v1"`) **does not match** the current ETag (`"v2"`).
8. The server **rejects** the request with **412 Precondition Faile**d .

   * Bob's update fails, which is the correct behavior. He is notified that the resource changed underneath him. He must now re-`GET` the resource to get the new data (and the `"v2"` ETag) and then retry his change.

5. Server Response Summary
--------------------------

This table summarizes the server's mandatory behavior when `ETag`-related headers are used.

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Request Header Sent** | **Use Case** | **Client ETag vs. Server ETag** | **Server Response** | **What It Means** |
| `If-None-Match` | Caching (`GET`) | **Match** | `304 Not Modified` | "The client's cached version is still good. Don't send the body." |
| `If-None-Match` | Caching (`GET`) | **No Match** | `200 OK` | "The resource has changed. Here is the new version and new `ETag`." |
| `If-Match` | Concurrency (`PUT`, `PATCH`, `DELETE`) | **Match** | `200 OK` (or `204`) | "The client was working on the correct version. The update is accepted." |
| `If-Match` | Concurrency (`PUT`, `PATCH`, `DELETE`) | **No Match** | `412 Precondition Failed` | "**Conflict!** The resource was changed by someone else. The update is rejected." |

---

6. OpenAPI Specification (OAS 3.1) Examples
-------------------------------------------

### Example: Defining ETag on a GET Response

```
paths:
  /users/{userId}:
    get:
      summary: Get a single user
      responses:
        '200':
          description: The user resource.
          headers:
            etag:
              schema:
                type: string
              description: The ETag for this version of the resource.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

### Example: Requiring If-Match on a PATCH Request

```
paths:
  /users/{userId}:
    patch:
      summary: Update a user
      parameters:
        - in: header
          name: if-match
          description: The ETag of the resource to update, for concurrency control.
          required: true
          schema:
            type: string
      requestBody:
        # ... request body definition
      responses:
        '200':
          description: Update successful.
          headers:
            etag:
              schema:
                type: string
              description: The new ETag for the updated resource.
        '412':
          description: Precondition Failed. The resource has been modified.
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
```