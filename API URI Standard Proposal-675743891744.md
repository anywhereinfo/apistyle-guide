# API URI Standard Proposal

> Confluence Page ID: 675743891744, Version: 4

Overview & Purpose
------------------

This document proposes a standardized URI structure for Lumen APIs. The goal is to create a consistent, predictable, and governable namespace that clarifies the purpose and scope of each API, supports different customer segments (Enterprise vs. Wholesale), and facilitates platform evolution.

Core URI Structure
------------------

The proposed core structure follows a `{domain}/{version}/{resource}` pattern:

* `{domain}`: Identifies the primary functional area or customer segment (e.g., `product`, `business-function`, `wholesale`).
* `{version}`: The major API version (e.g., `v1`, `v2`).
* `{resource}`: The specific resource (noun) being acted upon (e.g., `users`, `keys`, `orders`).

Proposed API Domains
--------------------

The following top-level domains are proposed to categorize Lumen APIs:

### Product Function APIs

* **Description:** Enterprise APIs performing product-specific functions.
* **URI Template:** `/{product}/{version}/{resource}`
* **Example:** `/fabric/v1/ports`, `/mcgw/v1/connections`

### Business Function APIs

* **Description:** Enterprise APIs performing cross-cutting business functions (e.g., identity, billing, ordering).
* **URI Template:** `/{business-function}/{version}/{resource}`
* **Example:** `/admin/v1/users`, `/ordering/v1/orders`
* **Note:** If a specific business function needs product-specific variations (e.g., inventory responses differing by product), a separate URI within the function domain might be necessary, rather than embedding product names directly.

### Wholesale APIs

* **Description:** A dedicated domain for APIs serving wholesale customers, covering both product and business functions.
* **URI Templates:**

  + `/wholesale/{product}/{version}/{resource}`
  + `/wholesale/{business-function}/{version}/{resource}`
* **Rationale:** Provides a distinct namespace for wholesale use cases. This enables:

  + **Impact Isolation:** Separates wholesale traffic from enterprise traffic.
  + **Tailored SLOs:** Allows different performance characteristics (e.g., higher timeouts for bulk wholesale orders).
  + **Targeted Auditing & Metrics:** Simplifies tracking of wholesale usage.
* **Implementation Note:** While the gateway URI is separate, a Wholesale API *can* potentially route to the same backend implementation as an Enterprise API to avoid code duplication where functionality aligns.
* **Cost Implication:** Each distinct gateway deployment (like `/wholesale/...`) may incur separate proxy deployment costs in Apigee X.

### Partner APIs (Open Question)

* **Discussion Needed:** Do we need a distinct `/partner/...` domain? What specific use cases (Pricing, Quoting, Marketplace integration, shared customer experiences) fall under this category? What are the partner types (Indirect, Hyperscalers, VARs)? This requires further discussion.

Examples: Business Function APIs
--------------------------------

This table illustrates how common business functions might map to the proposed URI structure. *Note: Many comments and questions here require discussion by the working group.*

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Business Function** | **API Name** | **Comment / Questions** | **Method** | **Proposed URI** |
| **User Administration** | *(N/A - UI Only?)* | New User Setup and role change not supported through API? Functions performed by admin in UI? | - | - |
|  | `resetUserPassword` |  | `POST` | `/admin/v1/user/{user_id}/password` |
|  | `deactivateUser` |  | `POST` | `/admin/v1/user/{user_id}/accounts` *(Note: URI seems mismatched? Should it be* `/admin/v1/users/{user_id}/status`*?)* |
|  | `listEntitlements` | Enables customers to consume on-demand products. Should this be administered through UI only? | `GET` | `/admin/v1/user/entitlements` *(Note: Per user or all? Needs clarification -* `/admin/v1/users/{user_id}/entitlements`*?)* |
|  | `addEntitlement` | Add entitlement to a user | `POST` | `/admin/v1/user/entitlements/{entitlement_id}` *(Note: Needs user context -* `/admin/v1/users/{user_id}/entitlements`*?)* |
|  | `removeEntitlement` | Remove entitlement from a user | `DELETE` | `/admin/v1/user/entitlements/{entitlement_id}` *(Note: Needs user context -* `/admin/v1/users/{user_id}/entitlements/{entitlement_id}`*?)* |
| **API Key Management** | `createAPIKey` | Create API Key. Requires existing key for admin APIs? Need mapping of key to allowed endpoints/methods. | `POST` | `/api-key/v1/keys` |
|  | `getAPIKeyList` | List all API Keys for a given user. | `GET` | `/api-key/v1/keys` |
|  | `getAPIkeyDetails` | List APIs a key is allowed to access. | `GET` | `/api-key/v1/keys/{api_key}` |
|  | `deleteAPIKey` | Delete an API key. | `DELETE` | `/api-key/v1/keys/{api_key}` |
| **Authorization** | `requestAccessToken` | Needs API Key. No user/pass auth. Key created via platform or API. | `POST` | `/oauth/v2/token` |
| **Service Availability** | `listServicesAvailableAtAddress` | Replaces current location API. Use address, not MasterSiteID. | `GET` | `/services/v1/availability` |
| **Inventory Management** | `getAvailableInventoryByLocation` | Get list of available product inventory at a location. | `GET` | `/inventory/v1/product` *(Note: Ambiguous URI? Needs location?* `/inventory/v1/locations/{loc_id}/products`*?)* |
|  | `getAvailableInventoryByProduct` | Get list of available product inventory (globally?). Priority for MCGW. Dependency on Blue Planet migration? | `GET` | `/inventory/v1/product` *(Note: Ambiguous URI? Needs product filter?* `/inventory/v1/products/{prod_id}/locations`*?)* |
|  | `getAvailableInventoryByProductByLocation` | Get list of available product inventory at a given location. | `GET` | `/inventory/v1/product` *(Note: Ambiguous URI? Needs both filters?* `/inventory/v1/locations/{loc_id}/products/{prod_id}`*?)* |
| **Pricing** | `getPricing` | Unauthenticated standard pricing. Covers most on-demand products. | `GET` | `/pricing/v1/catalogue` |
| **Quoting** | `createQuote` | Custom pricing for authenticated customers. More for classic products? Wholesale only? | `POST` | `/quoting/v1/price-request` |
|  | `saveQuote` | Save a quote. | `POST` | `/quoting/v1/price-request/{quote_id}` |
|  | `retrieveQuotes` | Get list of saved quotes. | `GET` | `/quoting/v1/price-request` |
| **Ordering** | `createOrder` | Creates order from saved quote (classic products). On-demand service order created behind the scenes. Wholesale only? | `POST` | `/ordering/v1/order` |
|  | `getOrdersList` |  | `GET` | `/ordering/v1/order` |
|  | `getOrderDetails` |  | `GET` | `/ordering/v1/order/{order_id}` |
|  | `checkOrderStatus` |  | `GET` | `/order/v1/order-status/{order_id}` |
|  | `cancelOrder` |  | `POST` | `/order/v1/resource` *(Note: URI seems incorrect?* `/ordering/v1/orders/{order_id}/cancel`*?)* |
| **Service Management** | `getServiceLists` | High Priority for MCGW | `GET` | `/services/v1/services` |
|  | `getServiceDetails` | Including usage, alerts. What does "service" mean? Align with Portal view. | `GET` | `/services/v1/services/{service_id}` |
|  | `runServiceDiagnostic` |  | `POST` | `/services/v1/services/{service_id}/diagnostics` *(Proposed)* |
|  | `createServiceTicket` |  | `POST` | `/support/v1/tickets` *(Proposed)* |
|  | `listServiceTickets` |  | `GET` | `/support/v1/tickets` *(Proposed)* |
|  | `checkServiceTicketStatus` |  | `GET` | `/support/v1/tickets/{ticket_id}/status` *(Proposed)* |
|  | `cancelServiceTicket` |  | `POST` | `/support/v1/tickets/{ticket_id}/cancel` *(Proposed)* |
| **Scheduled Maintenance** | *(TBD)* |  |  |  |
| **Billing** | *(TBD)* |  |  |  |
| **Telemetry** | *(TBD)* | APIs monitoring and usage |  |  |

Refined Business Function Categories Proposed
---------------------------------------------

This is an atttempt to group the functions into clearer, more standard domains:

* `identity`: Covers users, authentication, authorization, API keys, and entitlements. (Replaces `User Administration`, `API Key Management`, `Authorization`, parts of `admin`).
* `catalog (or offerings)`: Focuses on discovering *what* services are available *where*. (Replaces `Service Availability`).
* `inventory`: Manages the status and details of specific network resources/assets.
* `pricing`: Handles retrieval of standard pricing information.
* `quoting`: Manages the creation and retrieval of customer-specific quotes.
* `ordering`: Handles the submission and tracking of orders.
* `services`: Manages *provisioned* or *active* customer services (post-order). (Replaces `Service Management`).
* `support`: Manages trouble tickets and diagnostics related to active services. (Extracted from `Service Management`).
* `billing`: (Kept from original TBD).

Improved URIs Based on Refined Categories
-----------------------------------------

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Business Function (Domain)** | **API Name / Action** | **Method** | **Proposed URI** | **Original URI Notes Addressed** |
| **Identity** | Reset User Password | `POST` | `/identity/v1/users/{user_id}/actions/reset-password` | Uses action pattern, clearer than `/password`. |
|  | Deactivate User | `PATCH` | `/identity/v1/users/{user_id}` (Body: `{"status": "inactive"}`) | Uses `PATCH` for state change, corrects mismatched `/accounts` URI. |
|  | List User Entitlements | `GET` | `/identity/v1/users/{user_id}/entitlements` | Clarifies per-user scope. |
|  | Add Entitlement to User | `POST` | `/identity/v1/users/{user_id}/entitlements` (Body: `{"entitlement_id": "...", ...}`) | Creates an entitlement *assignment*. |
|  | Remove Entitlement from User | `DELETE` | `/identity/v1/users/{user_id}/entitlements/{entitlement_id}` | Deletes the *assignment*. |
|  | Create API Key for User | `POST` | `/identity/v1/users/{user_id}/api-keys` | Assumes keys belong to users. |
|  | List User's API Keys | `GET` | `/identity/v1/users/{user_id}/api-keys` |  |
|  | Get API Key Details | `GET` | `/identity/v1/users/{user_id}/api-keys/{key_id}` |  |
|  | Delete API Key | `DELETE` | `/identity/v1/users/{user_id}/api-keys/{key_id}` |  |
| **(Standard OAuth)** | Request Access Token | `POST` | `/oauth/v2/token` | Kept standard URI. |
| **Catalog/** **Offerings** | Check Service Offering Availability | `GET` | `/catalog/v1/availability?address=...` (or other filters) |  |
| **Inventory** | Get Product Inventory | `GET` | `/inventory/v1/product-inventory?location_id=...&product_id=...` | Uses query parameters for flexible filtering by location, product, or both. Addresses ambiguity. Resource named `product-inventory`. |
| **Pricing** | Get Price Catalog | `GET` | `/pricing/v1/rates` |  |
| **Quote** | Create Quote | `POST` | `/quote/v1/quotes` |  |
|  | Save/Update Quote | `PUT` / `PATCH` | `/quote/v1/quotes/{quote_id}` |  |
|  | Retrieve Quotes | `GET` | `/quote/v1/quotes` |  |
|  | Retrieve Specific Quote | `GET` | `/quote/v1/quotes/{quote_id}` |  |
| **Order** | Create Order | `POST` | `/order/v1/orders` (Body includes `quote_id` if needed) |  |
|  | List Orders | `GET` | `/order/v1/orders` |  |
|  | Get Order Details | `GET` | `/order/v1/orders/{order_id}` |  |
|  | Cancel Order | `POST` | `/order/v1/orders/{order_id}/actions/cancel` |  |
| **Services (Provisioned)** | List Services | `GET` | `/services/v1/services` |  |
|  | Get Service Details | `GET` | `/services/v1/services/{service_id}` | (Includes usage, status, alerts etc.). |
|  | Run Service Diagnostic | `POST` | `/services/v1/services/{service_id}/actions/run-diagnostics` | Uses action pattern. |
| **Support** | Create Support Ticket | `POST` | `/support/v1/tickets` | Moved ticketing to dedicated domain. |
|  | List Support Tickets | `GET` | `/support/v1/tickets` |  |
|  | Get Ticket Details | `GET` | `/support/v1/tickets/{ticket_id}` | (Includes status). |
|  | Cancel Support Ticket | `POST` | `/support/v1/tickets/{ticket_id}/actions/cancel` | Uses action pattern. |
| **Billing** | *(TBD)* | *(TBD)* | *(TBD)* |  |

Open Issues & Discussion Points for Working Group
-------------------------------------------------

* **Partner APIs:** Confirm need, scope, and URI structure (`/partner/...`?).
* **Wholesale Exclusivity:** Confirm if Quoting/Ordering APIs are truly Wholesale-only or need Enterprise equivalents.
* **Product-Specific Business Functions:** Finalize approach for handling variations within business functions (e.g., Inventory).
* **UI vs. API Scope:** Clarify which functions (like User Admin, Entitlements) are intentionally UI-only vs. candidates for APIs.
* **Inventory API Structure:** Proposed URIs seem ambiguous and need refinement to properly filter by location/product. Confirm dependency on Blue Planet migration.
* **Service Management Scope:** Define "service" clearly (align with Portal?). Refine proposed URIs for diagnostics/ticketing.
* **URI Mismatches:** Review URIs noted in the table above (e.g., `deactivateUser`, `cancelOrder`) for correctness.
* **Authentication/Authorization Details:** Clarify API Key management flow, especially creation and permissions mapping.

Next Steps
----------

1. Review proposed domain structure (`product`, `business-function`, `wholesale`).
2. Discuss and decide on the "Partner API" question.
3. Review the Business Function examples, address embedded questions, and refine URIs for clarity and consistency.
4. Assign owners/actions for resolving open issues.
5. Formalize approved structure in the main API Style Guide.