# OAS Specification Best Practices

> Confluence Page ID: 675743727627, Version: 6

none

This document provides a set of standardized guidelines for creating high-quality, maintainable, and developer-friendly OpenAPI specifications. Adhering to these practices ensures consistency and clarity across all our APIs.

---

OAS Version and Format
----------------------

### **Adopt OAS 3.0.3 as the Standard**

* Utilize OAS 3.0.3 for all new API specifications to ensure alignment with ApigeeX industry practices.
* Include the correct version declaration at the start of the document:

  ```
  openapi: 3.0.3
  ```

### **Use YAML as the Documentation Format**

* Employ YAML for writing OAS documents for better readability and maintainability.
* Use a consistent indentation of two spaces to structure the document clearly.

---

API Metadata (`info` object)
----------------------------

The `info` object is the first thing a developer sees. It must clearly articulate the API's purpose and business value.

* `info.description`: This field **MUST** describe the business capability the API unlocks. It should answer the question, "What problem does this solve for the consumer?" Avoid technical jargon.

  + ***Bad***: "An API for managing MCGW connections."
  + ***Good***: "Use the Fabric Connections API to programmatically create, manage, and delete high-speed, private connections between your on-premises infrastructure and cloud providers like AWS, Azure, and GCP."

---

API Documentation Structure (`tags` and `summary`)
--------------------------------------------------

These fields control how your API is organized and displayed in documentation tools, making them critical for usability and context.

* `tags`: All operations **MUST** be grouped using `tags`. The tags **MUST** be based on the resource name (e.g., a "Connections" tag for all operations related to connections) to organize the API around its core business objects.
* `summary`: Every operation **MUST** have a `summary`. It must be a short, action-oriented phrase from the user's perspective that clearly states what the endpoint does.

  + *Bad*: "GET connections"
  + *Good*: "List all active connections"

---

Operation IDs (`operationId`)
-----------------------------

The `operationId` is used by code generation tools to create method names in SDKs. Inconsistent or missing IDs lead to messy, unpredictable code and a poor developer experience.

* **Requirement**: Every operation **MUST** have a unique `operationId`.
* **Convention**: The `operationId` **MUST** follow a consistent naming convention, such as `verbResource` (e.g., `listConnections`, `createConnection`). This ensures the generated code is predictable and intuitive.

---

Security
--------

Apply security requirements at the individual path or operation level rather than globally. This approach provides precise control and accommodates varying security needs for different endpoints.

```
paths:
  /items:
    get:
      security:
        - OAuth2:
            - read
```

---

Reusability and Naming
----------------------

* **Reuse Schemas**: Leverage the `components` section to define and reuse schemas, parameters, and headers. This ensures consistency and makes the specification easier to maintain.
* **Descriptive Naming Conventions**: Use meaningful names for paths, parameters, and other elements, and follow the company's established naming standard.

### How to Document Reusable Components

The `components` section is the central repository for all reusable definitions.

#### Reusable Request Parameters (Query & Path)

Define reusable query and path parameters under

```
components.parameters.
```

#### Declare Reusable Parameters example

```
components:
  parameters:
    ResourceId:
      name: id
      in: path
      description: The unique identifier for the resource.
      required: true
      schema:
        type: string
        format: uuid
    LimitParam:
      name: limit
      in: query
      description: The maximum number of items to return per page.
      schema:
        type: integer
        default: 20
```

#### Reference them in your paths

```
paths:
  /v1/items/{id}:
    parameters:
      - $ref: '#/components/parameters/ResourceId'
```

#### Reusable Request Headers

Request headers are treated as parameters with “in: header” and are also defined under

```
components.parameters
```

#### Define the reusable header as a parameter

```
components:
  parameters:
    CorrelationIdHeader:
      name: X-Correlation-ID
      in: header
      description: Tracks a request across multiple services for debugging.
      required: true
      schema:
        type: string
        format: uuid
```

#### Reference it in your path

```
paths:
  /v1/items:
    post:
      parameters:
        - $ref: '#/components/parameters/CorrelationIdHeader'
```

#### Reusable Response Headers

Response headers have their own dedicated section for reusability:

```
components.headers
```

#### Define the reusable response header

```
components:
  headers:
    RateLimitRemaining:
      description: The number of requests left for the current time window.
      schema:
        type: integer
```

#### Reference it in a response definition

```
paths:
  /v1/items:
    get:
      responses:
        '200':
          description: A successful response.
          headers:
            RateLimit-Remaining:
              $ref: '#/components/headers/RateLimitRemaining'
```

---

Data Types and Examples
-----------------------

* **Define Reusable Schemas**: Define reusable schemas for complex data types in the `components` section. Clearly specify ***data types, formats, constraints, and examples***.
* **Provide Realistic Examples**:

  + Include examples that cover various real-world scenarios (common, edge, and error cases).
  + Use the `example` or `examples` keyword within schema definitions for both requests and responses.
  + Ensure examples use realistic and relevant data.

#### OAS `examples` Example

```
paths:
  /v1/items:
    post:
      summary: Create a new item
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Item'
            examples:
              # First example: A simple item with only required fields
              simpleItem:
                summary: A basic item
                description: An example of creating an item with the minimum required information.
                value:
                  name: "Standard Widget"
                  price: 19.99

              # Second example: An item with optional fields included
              itemWithOptionalFields:
                summary: An item with optional data
                description: An example of creating an item that includes optional fields like a description and SKU.
                value:
                  name: "Premium Widget"
                  price: 29.99
                  description: "A high-quality, durable widget."
                  sku: "PREM-WID-001"
```

Casing Requirements
-------------------

Schema objects defined in the `components` section **MUST** use **PascalCase** (also known as UpperCamelCase). This convention makes it easy to distinguish schema objects from `snake_case` field names within the specification.

```
# ✅ Good: Uses PascalCase
CustomerOrder:
  type: object
  properties:
    order_id:
      type: string
# ❌ Bad: Uses snake_case
customer_details:
  type: object
```

---

Best Practices for Naming
-------------------------

### Use Singular Nouns

Schema object names **MUST** represent a single instance of that object. Use singular nouns for clarity. The concept of a collection is handled by defining an `array` that references the singular object.

* *Good*: `Customer`, `Order`, `Invoice`
* *Bad*: `Customers`, `OrdersList`

### Be Descriptive and Unambiguous

Names should be clear and specific, avoiding generic or vague terms that could cause confusion.

* *Good*: `BillingAddress`, `ShippingAddress`, `ProductInventory`
* *Bad*: `Data`, `Items`, `Record`, `Object`

### Avoid Jargon and Abbreviations

Names **SHOULD** use simple, common terms and avoid internal project codenames, jargon, or unnecessary abbreviations.

* *Good*: `MultiCloudGateway`, `Invoice`
* *Bad*: `McgwObject`, `BillingRecord`

### Use Suffixes for Clarity

When a schema has different variations (e.g., for requests vs. responses), use consistent suffixes to distinguish them.

* **For Create/Update Operations**: Use suffixes like `Request` or `Update`.

  + `CreateCustomerRequest`
  + `UpdateCustomerRequest`
* **For Different Views**: Use suffixes that describe the context.

  + `CustomerSummary`
  + `CustomerDetails`

### Use API Prefixes to Prevent Namespace Collisions

To prevent conflicts when developers generate code from multiple OAS files, all schema names **MUST** be prefixed with a short, unique **API Prefix**.

An **API Prefix** is a short identifier, unique to an API, that is added to the beginning of every schema object's name. Its purpose is to guarantee that all generated class names will be unique, even when a developer consumes multiple APIs that have schemas with similar names.

* **The Problem**: If the MCGW API has an `Order` schema and the IOD API also has an `Order` schema, code generation tools will create two different classes with the same name, causing a conflict.
* **The Solution**: By using prefixes, the schemas are named `McgwOrder` and `IodOrder`, which generate unique and conflict-free classes.