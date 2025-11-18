# API Pagination Standards

> Confluence Page ID: 675750379523, Version: 3

none

**1. Introduction**
-------------------

Pagination is the process of dividing a large set of data into smaller, discrete pages. Proper pagination is essential for creating APIs that are performant, scalable, and easy for our customers and partners to consume. Failure to implement a consistent pagination strategy leads to slow response times, server strain, and a poor developer experience.

This document establishes the proposed official standards for implementing pagination within Lumen's API ecosystem. All development teams are required to adhere to these guidelines to ensure a cohesive and predictable platform.

---

**2. The Default Standard: Offset-Based Pagination**
----------------------------------------------------

For the vast majority of use cases, **Offset-based pagination** should be the default method. It is intuitive, flexible, and meets the needs of most applications where users need to navigate to specific pages.

### **Query Parameters**

Paginated endpoints MUST accept the following query parameters:

* `limit`: An integer specifying the maximum number of items to return per page.

  + If not provided, the API MUST default to `25`.
  + The API MUST enforce a maximum value of `100`. Requests for more than 100 items should result in a `400 Bad Request` error.
* `offset`: An integer specifying the number of records to skip. This is how the client navigates to subsequent pages.

  + If not provided, the API MUST default to `0` (the first page).

**Example Request:** `GET /v1/services?limit=50&offset=100` (Returns 50 services, starting after the 100th service).

### **Response Structure**

The response body for a paginated request MUST be a JSON object containing two top-level keys: `data` and `pagination`.

* `data`: An array containing the resource objects for the current page.
* `pagination`: An object containing the following metadata fields:

  + `total`: The total number of items available in the entire collection.
  + `limit`: The `limit` value used for the current page.
  + `offset`: The `offset` value used for the current page.

**Example Response:**

```
{
  "data": [
    { "id": "svc-abc-101", "name": "Service 101" },
    // ... 49 more items
    { "id": "svc-xyz-150", "name": "Service 150" }
  ],
  "pagination": {
    "total": 473,
    "limit": 50,
    "offset": 100
  }
}
```

---

**3. High-Performance Standard: Keyset (Cursor-Based) Pagination**
------------------------------------------------------------------

For endpoints that expose very large datasets (millions of records) or data that changes frequently (e.g., event logs, real-time data), **Keyset pagination** MUST be used. This method is more performant and provides a stable window into the data, avoiding issues where items are skipped or repeated as new data is added.

### **Query Parameters**

* `limit`: Same definition as in Offset pagination (default `25`, max `100`).
* `after`: A string representing an opaque cursor that points to the last item of the previous page. The API will return items that come after this cursor.

**Example Request:** `GET /v1/events?limit=50&after=NGM5YTYxYjItM2RiYi00YjU4LTg5ZmMtMTdiYmI5ZjIzMjc5`

### **Response Structure**

The response structure for Keyset pagination is simplified to facilitate "infinite scroll" style navigation.

* `data`: An array containing the resource objects for the current page.
* `pagination`: An object containing the following metadata:

  + `next_cursor`: The cursor string for the next page of results. This value is passed into the `after` parameter in the subsequent request. It is `null` if there are no more pages.
  + `has_next_page`: A boolean indicating if more pages are available.
  + `next`: A full URL string pointing to the next page of results, including the `after` parameter.

**Example Response:**

```
JSON
```

```
{
  "data": [
    { "id": "evt-123", "timestamp": "2025-10-15T14:30:00Z" },
    // ... 49 more items
    { "id": "evt-456", "timestamp": "2025-10-15T14:28:00Z" }
  ],
  "pagination": {
    "next_cursor": "ZjkzMjc5YTYxYjItM2RiYi00YjU4LTg5ZmMtMTdiYmI5Zj",
    "has_next_page": true,
    "next": "https://api.lumen.com/v1/events?limit=50&after=ZjkzMjc5YTYxYjItM2RiYi00YjU4LTg5ZmMtMTdiYmI5Zj"
  }
}
```

---

**4. Summary of Guidance**
--------------------------

1. **Default to Offset:** Use Offset pagination for all standard resource collections.
2. **Use Keyset for Scale:** Use Keyset (Cursor) pagination for event streams, logs, and any dataset exceeding one million records where performance and data consistency are critical.
3. **Provide Hypermedia Links:** Always include fully-qualified URLs for `next` and `previous` links. This reduces the burden on the client and makes the API more discoverable (HATEOAS).
4. **Enforce Limits:** Always enforce a default and maximum `limit` to protect the API from misuse.
5. **Document Clearly:** The chosen pagination method and its parameters must be clearly documented in the OpenAPI specification for every applicable endpoint.