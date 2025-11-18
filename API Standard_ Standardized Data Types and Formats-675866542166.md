# API Standard: Standardized Data Types and Formats

> Confluence Page ID: 675866542166, Version: 4

none

### **Purpose**

To ensure consistency, readability, and interoperability across all APIs published by Lumen, this section defines the canonical data types, formats, and reusable schemas to be used in all OpenAPI 3.0.3 specifications.

---

### **Principles**

* APIs **MUST** use only the data types and formats listed here.
* All fields that accept structured values (e.g., country, currency, language, etc.) **MUST** include:

  + a `description` explaining the expected standard,
  + a `pattern` (regex) to make the rule self-explanatory to external developers, and
  + an `example` value.
* Custom `format` identifiers (e.g., `country-code`) are **advisory only**.

  + They are intended for internal governance, linting, and future migration to OAS 3.1.
  + External consumers should be able to use the schema **without** referencing any style guide.

---

### **Rationale: Custom** `format` **vs Regex**

| Aspect | Custom `format` | Regex + Example |
| --- | --- | --- |
| **Purpose** | Governance keyword for Spectral, SDKs, portals | Human & machine validation rule |
| **External Developer Value** | Low – tooling hint only | High – immediately clear |
| **Governance Value** | High – centralized semantics | Medium – must duplicate everywhere |
| **Recommended Use** | Together – `format` + `pattern` + `example` |  |

---

### **Standard Data Types (OAS 3.0.3)**

| Type | OAS `type` | OAS `format` | Description | Example |
| --- | --- | --- | --- | --- |
| Boolean | `boolean` | — | `true` or `false` | `true` |
| Integer (32 bit) | `integer` | `int32` | −2,147,483,648 … 2,147,483,647 | `42` |
| Integer (64 bit) | `integer` | `int64` | −9,223,372,… 7, use ≤ `2^53 – 1` for I-JSON | `9007199254740991` |
| Number (float) | `number` | `float` | IEEE-754 single precision | `3.14` |
| Number (double) | `number` | `double` | IEEE-754 double precision | `3.1415926535` |
| String | `string` | — | UTF-8 text | `"Lumen rocks!"` |
| Date | `string` | `date` | RFC 3339 date | `"2025-11-11"` |
| Date-Time | `string` | `date-time` | RFC 3339 timestamp (UTC `Z`) | `"2025-11-11T14:05:00Z"` |
| Byte (Base64) | `string` | `byte` | Base64 RFC 4648 § 4 | `"VGVzdA=="` |
| Binary | `string` | `binary` | Raw octet stream (upload/download) | *(file body)* |

---

### **Custom (Governed) Formats**

> ✅ These are optional keywords for internal tooling and do **not** alter wire format.

| Logical Type | OAS `type` | OAS `format` | Description / Rule | Example |
| --- | --- | --- | --- | --- |
| URI | `string` | `uri` | URI per RFC 3986 | `"https://example.com/x"` |
| Email | `string` | `email` | RFC 5322 address | `"user@example.com"` |
| UUID | `string` | `uuid` | RFC 4122 UUID | `"279fc665-d04d-4dba-bcad-17c865489dfa"` |
| Time | `string` | `time` | RFC 3339 partial time | `"08:26:40Z"` |
| Decimal | `string` | `decimal-string` | Arbitrary precision string | `"1234.5678"` |
| Language | `string` | `language-tag` | ISO 639-1 / BCP 47 code | `"en-US"` |
| Country | `string` | `country-code` | ISO 3166-1 alpha-2 uppercase | `"US"` |
| Currency | `string` | `currency-code` | ISO 4217 alpha-3 uppercase | `"USD"` |

Each field **must also define a pattern and example** so the meaning is clear even if clients ignore `format`.

---

### **Reusable Schema Definitions**

#### **CountryCode**

```
CountryCode:
  type: string
  description: ISO 3166-1 alpha-2 country code (uppercase).
  format: country-code
  pattern: '^[A-Z]{2}$'
  example: "US"
```

#### **CurrencyCode**

```
CurrencyCode:
  type: string
  description: ISO 4217 currency code (uppercase).
  format: currency-code
  pattern: '^[A-Z]{3}$'
  example: "USD"
```

#### **LanguageTag**

```
LanguageTag:
  type: string
  description: Language tag per ISO 639-1 / BCP 47.
  format: language-tag
  pattern: '^[A-Za-z]{2,3}(-[A-Za-z0-9]{2,8})*$'
  example: "en-US"
```

#### **Email**

```
Email:
  type: string
  description: Email address per RFC 5322.
  format: email
  pattern: '^[^@\s]+@[^@\s]+\.[^@\s]+$'
  example: "user@example.com"
```

#### **UUID**

```
UUID:
  type: string
  description: Universally Unique Identifier per RFC 4122.
  format: uuid
  pattern: '^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$'
  example: "279fc665-d04d-4dba-bcad-17c865489dfa"
```

#### **URI**

```
URI:
  type: string
  description: URI string per RFC 3986.
  format: uri
  pattern: '^(https?|ftp)://[^\s/$.?#].[^\s]*$'
  example: "https://api.lumen.com/v1/services"
```

---

#### **Money Object ( Integer Minor Units)**

> Represents a monetary amount using ISO 4217 minor units (integer-based).

```
Money:
  type: object
  required: [amount, currency]
  properties:
    amount:
      type: integer
      minimum: 0
      maximum: 9007199254740991   # I-JSON safe (2^53 – 1)
      description: >
        Monetary amount expressed in ISO 4217 minor units.
        Example: USD 12.34 → amount=1234, currency="USD"
    currency:
      $ref: '#/components/schemas/CurrencyCode'
  example:
    amount: 1234
    currency: "USD"
```

---

#### **Address Object**

A common postal address representation used consistently across all domains.

```
Address:
  type: object
  description: Postal address (country-specific formatting may apply).
  additionalProperties: false
  properties:
    line1:
      type: string
      description: Primary street line (street, number, unit).
      maxLength: 256
      example: "100 Main St Apt 5B"
    line2:
      type: string
      description: Secondary line (building, suite, floor) if needed.
      maxLength: 256
      example: "Building A, Floor 3"
    city:
      type: string
      description: City, town, or locality.
      maxLength: 128
      example: "Denver"
    state:
      type: string
      description: State, region, or province (country-specific).
      maxLength: 128
      example: "CO"
    postal_code:
      type: string
      description: Postal/ZIP code (country-specific format).
      maxLength: 32
      example: "80202"
    country:
      $ref: '#/components/schemas/CountryCode'
  example:
    line1: "100 Main St Apt 5B"
    city: "Denver"
    state: "CO"
    postal_code: "80202"
    country: "US"
```

**Context-specific variants:**

```
UsShippingAddress:
  allOf:
    - $ref: '#/components/schemas/Address'
    - type: object
      required: [line1, city, state, postal_code, country]
```

```
InternationalBillingAddress:
  allOf:
    - $ref: '#/components/schemas/Address'
    - type: object
      required: [line1, city, postal_code, country]
```

---

### **Governance Validation Rules**

| Rule | Enforcement |
| --- | --- |
| Disallow unrecognized `format` values | Spectral lint |
| Require `pattern`, `example`, `description` when custom `format` used | Spectral lint |
| Validate Money.amount ≤ 2⁵³ – 1 | Spectral + CI schema check |
| Enforce ISO regex patterns for country/currency/language | Spectral rule |
| Warn if `float`/`double` used for monetary values | Spectral rule |

---

### **Summary**

* **External developers** should understand schemas immediately through **description + regex + example**, without needing to read the style guide.
* **Governance tooling** (Spectral, SDK generators, portals) can rely on **custom formats** for consistency and automation.
* **Money, Address, Country, Currency, and Language** fields are standardized across all domains.