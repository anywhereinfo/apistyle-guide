# API Standard: Rate Limits & Error Handling Standard

> Confluence Page ID: 675890790459, Version: 2

1. Executive Summary
--------------------

This standard defines how Lumen APIs should apply rate limiting and error handling using Apigee X, IETF `RateLimit-*` headers, and LPDP-Mini.

In most cases, every public-facing API should combine **two complementary layers**:

* **Layer 1 – Client Fairness:**  
  Per-client limits enforced with **Quota** (cluster-aware, contractual).
* **Layer 2 – Global Safety:**  
  Infrastructure protection using **SpikeArrest** (node-local, non-contractual).

### 1.1 Quick Decision Guide

| Scenario | Recommended Approach | Rationale |
| --- | --- | --- |
| Per-client or per-application product limits | **Quota (Layer 1)** | Accurate, cluster-wide counters and contractual semantics |
| Backend with fixed capacity (DB, mainframe) | **SpikeArrest – Legacy (Layer 2)** | Smooths concurrency and protects fragile systems |
| Autoscaling backend (Kubernetes, FaaS, microservices) | **SpikeArrest – Modern (Layer 2)** | Allows bursts up to a safe per-node ceiling |
| Authenticated APIs | **Quota + SpikeArrest** | Combines fairness with safety |
| Anonymous public APIs | **IP-based SpikeArrest** | Basic protection against bots and abusive traffic |
| No reliable identity for clients | **SpikeArrest only** | Quota requires a trusted identifier |

> **Golden principle:**  
> For most production APIs, use **Quota (Layer 1)** for fairness and **SpikeArrest (Layer 2)** for infrastructure protection.

---

2. Core Model: The Two-Layer Defense
------------------------------------

To balance **user experience**, **fair usage**, and **backend stability**, all public-facing APIs should be designed using a two-layer funnel:

| Layer | Scope | Policy | Primary Objective |
| --- | --- | --- | --- |
| **Layer 1 — Client Fairness** | Per application (`client_id`) | **Quota (short interval)** | Enforce contract-grade, per-client limits across the cluster (e.g., 1000 requests per 5 minutes) |
| **Layer 2 — Global Safety** | Aggregate traffic | **SpikeArrest** | Protect backends from traffic bursts, thundering herds, and gateway autoscaling events |

### 2.1 Governance Intent

* **Quota** represents the **contract** between Lumen and the client (fairness, tiers, SKUs).
* **SpikeArrest** represents **infrastructure protection** and should not be interpreted as a customer-facing limit or product feature.

---

3. Platform Behavior and Cluster Awareness
------------------------------------------

### 3.1 Quota (Cluster-Aware – Required for Layer 1)

* Quota counters are **distributed across all MPs**.
* Limits are preserved even as Apigee scales horizontally.

**Required configuration:**

```
<Distributed>true</Distributed>
```

> **Quota window semantics:**
>
> * Quota uses a **fixed window** model.
> * At the end of each interval, the counter resets completely.
> * Unused quota **does not carry over** to the next window.

---

### 3.2 SpikeArrest (Node-Local – Required for Layer 2)

* SpikeArrest executes **per MP** and is **not cluster-aware**.
* Each MP enforces its own local rate limit; total cluster throughput is a product of the configured rate and the number of MPs.

**Effective cluster rate:**

```
EffectiveLimit = ConfiguredRate_per_MP × Active_MPs
```

**Governance guidance:**

* Use SpikeArrest solely for **traffic shaping and backend safety**.
* Do not treat SpikeArrest as a customer-facing or contractual limit.
* When sizing SpikeArrest, use the **expected peak number of MPs**, not the current or minimum count, and validate via load testing.

---

4. Pattern 1 — Layer 1: Client Fairness (Quota)
-----------------------------------------------

### 4.1 Choosing an Appropriate Quota Interval

Quota intervals should balance fairness, user experience, and datastore overhead.

| Interval | Advantages | Trade-offs | Recommended Usage |
| --- | --- | --- | --- |
| **1 minute** | Very tight fairness, quick recovery | Higher datastore traffic and possible “reset spikes” | Premium tiers and strict SLAs |
| **5 minutes** | Balanced fairness and overhead | Moderate burst tolerance | **Recommended default** |
| **10 minutes** | Lower load on datastore | Looser fairness, longer wait on exhaustion | High-volume, lower-value APIs |
| **1 hour** | Minimal storage pressure | Poor user experience on exhaustion | Legacy or transitional scenarios only; avoid for new designs |

> **Default:** For most APIs, a **5-minute** interval is recommended.

---

### 4.2 Example: Client Fairness Quota Policy

**Purpose:**  
Enforces a contract-grade limit of 1000 requests per 5 minutes per verified client application.

```
<Quota name="Q-Client-Fairness">
    <Allow count="1000"/>
    <Interval>5</Interval>
    <TimeUnit>minute</TimeUnit>
    <Distributed>true</Distributed>
    <!-- Identifier must be derived from verified credentials, not client-provided headers -->
    <Identifier ref="apigee.client_id"/>
</Quota>
```

**Identifier best practices:**

* `apigee.client_id` should be populated by `VerifyAPIKey` or OAuth flows.
* Avoid using client-controlled headers (e.g., `X-Client-Id`) as identifiers for Quota.
* If `apigee.client_id` is missing, that condition should be treated as an **authentication/authorization failure**, not as a rate-limit violation.

---

5. Pattern 2 — Layer 2: Global Backend Defense (SpikeArrest)
------------------------------------------------------------

### 5.1 Option A: Legacy / Fixed-Capacity Backends (Strict Mode)

Use this approach when protecting systems that are sensitive to concurrency and cannot scale elastically, such as mainframes or certain relational databases.

**Sizing guidance (Strict):**

```
ConfiguredRate_per_MP =
  (Backend_Safe_TPS × Safety_Margin) / Target_MP_Count_at_Peak_Load

Where:
- Backend_Safe_TPS  = safe throughput determined by testing
- Safety_Margin      = 0.7–0.8 (20–30% headroom)
- Target_MP_Count_at_Peak_Load = expected MP count during peak traffic
```

**Example:**

```
<SpikeArrest name="SA-Global-Legacy">
    <!-- Example value; teams must derive actual values through load tests -->
    <Rate>200ps</Rate>
    <UseEffectiveCount>false</UseEffectiveCount>
</SpikeArrest>
```

---

### 5.2 Option B: Modern Autoscaling Backends (Burst Mode)

Use this approach when backends can scale horizontally (e.g., Kubernetes, FaaS, microservices) and primarily need protection against unbounded surges.

**Sizing guidance (Burst):**

```
ConfiguredRate_per_MP =
  (Backend_Max_TPS × Safety_Margin) / Target_MP_Count_at_Peak_Load
```

**Example:**

```
<SpikeArrest name="SA-Global-Modern">
    <!-- Example value; teams must derive actual values through load tests -->
    <Rate>1000ps</Rate>
    <UseEffectiveCount>true</UseEffectiveCount>
</SpikeArrest>
```

> These numeric values are illustrative. Teams should derive real values from performance baselines and capacity planning.

---

### 5.3 Anonymous or Unauthenticated APIs (IP-Based SpikeArrest)

For endpoints that do not have a reliable authenticated identity, IP-based rate limiting can provide basic protection.

#### 5.3.1 Example: IP-Based SpikeArrest

```
<SpikeArrest name="SA-Anonymous-IP">
    <Rate>10pm</Rate>
    <Identifier ref="client.ip.normalized"/>
</SpikeArrest>
```

**IP extraction and normalization (simplified example):**

```
<ExtractVariables name="EV-Extract-Client-IP">
    <Source>request.header.x-forwarded-for</Source>
    <Pattern>{clientip},</Pattern>
    <VariablePrefix>extracted</VariablePrefix>
</ExtractVariables>

<Javascript name="JS-Normalize-IPv6" timeLimit="100">
    <Source>
        // In production, use a robust library for IPv6 normalization
        var ip = context.getVariable("extracted.clientip") ||
                 context.getVariable("client.ip");
        context.setVariable("client.ip.normalized", ip);
    </Source>
</Javascript>
```

**Considerations:**

* Carrier-grade NAT (CGNAT) can cause many users to share a single IP address.
* IPv6 address rotation within a `/64` subnet can be used to evade simple IP-based limits.
* For higher assurance, consider augmenting IP-based limits with additional signals such as User-Agent or device fingerprinting.

---

6. Rate Limit Response Headers (RFC 7231 + draft-ietf-httpapi-ratelimit-headers)
--------------------------------------------------------------------------------

Lumen adopts the IETF standard `RateLimit-*` **response headers**. These headers should only be exposed when the data is accurate at a cluster level.

### 6.1 Principles

* **Quota-based limits** have accurate distributed counters and should publish `RateLimit-*` headers.
* **SpikeArrest-based limits** are node-local and should not publish rate-limit usage headers. They may only provide a generic `Retry-After`.

---

### 6.2 Infrastructure-Level Limits (SpikeArrest)

**Scenario:** SpikeArrest has blocked a request due to high load or excessive bursts.

**Header behavior:**

| Header | Requirement | Notes |
| --- | --- | --- |
| `Retry-After` | Recommended | A small fixed value (e.g., 5–10 seconds) to signal temporary backoff |
| `RateLimit-Limit` | Must not be sent | Not accurate cluster-wide |
| `RateLimit-Remaining` | Must not be sent | Not accurate |
| `RateLimit-Reset` | Must not be sent | Not accurate |

---

### 6.3 Client Fairness or Business Limits (Quota)

**Scenario:** A client has reached its contractual or fairness limit for the current window.

**Header behavior:**

| Header | Requirement | Description |
| --- | --- | --- |
| `RateLimit-Limit` | Required | Total requests allowed in the window (e.g., `1000`) |
| `RateLimit-Remaining` | Required | Remaining requests in the current window (commonly `0` at violation) |
| `RateLimit-Reset` | Required | Seconds until the quota window resets |
| `Retry-After` | Required | Same value as `RateLimit-Reset` |

---

7. Error Format: LPDP-Mini for 429 Responses
--------------------------------------------

All `429 Too Many Requests` responses must comply with **LPDP-Mini v1.0**, using `application/problem+json`.

### 7.1 Required Structure

* `title` — short, human-readable summary.
* `errors[]` — array of error objects containing:

  + `code` — machine-readable error identifier.
  + `message` — clear explanation for human consumers.
  + `meta` — optional, structured details that do *not* reveal internal or sensitive information.

### 7.2 Example LPDP-Mini 429 Response

```
{
  "title": "Rate Limit Exceeded",
  "errors": [
    {
      "code": "traffic.limit_exceeded",
      "message": "You have exceeded the allowed request rate.",
      "meta": {
        "retry_after_seconds": "60"
      }
    }
  ]
}
```

---

8. Implementation Pattern (Shared Flow Recommended)
---------------------------------------------------

To ensure consistency, rate limiting and error handling should be implemented as **Shared Flows** and attached via FlowHooks to relevant proxies.

### 8.1 Fault Handling Structure

The following pattern illustrates how to route different fault conditions:

```
<FaultRules>
    <!-- Authentication/authorization issues -->
    <FaultRule name="Handle-Missing-ClientID">
        <Condition>(apigee.client_id = null) or (apigee.client_id = "")</Condition>
        <Step><Name>AM-Set-401-Unauthorized</Name></Step>
    </FaultRule>

    <!-- SpikeArrest violations -->
    <FaultRule name="Handle-Spike-Violation">
        <Condition>(fault.name = "SpikeArrestViolation")</Condition>
        <Step><Name>AM-Set-429-SpikeArrest</Name></Step>
    </FaultRule>

    <!-- Quota violations -->
    <FaultRule name="Handle-Quota-Violation">
        <Condition>(fault.name = "QuotaViolation")</Condition>
        <Step><Name>JS-Calculate-Reset</Name></Step>
        <Step><Name>AM-Set-429-Quota</Name></Step>
    </FaultRule>
</FaultRules>
```

---

### 8.2 Authentication Fault Response (401 Unauthorized)

```
<AssignMessage name="AM-Set-401-Unauthorized">
    <Set>
        <StatusCode>401</StatusCode>
        <ReasonPhrase>Unauthorized</ReasonPhrase>
        <Headers>
            <Header name="Content-Type">application/problem+json</Header>
        </Headers>
        <Payload contentType="application/problem+json">
            {
                "title": "Authentication Required",
                "errors": [
                    {
                        "code": "auth.missing_credentials",
                        "message": "A valid API key or OAuth token is required to access this API."
                    }
                ]
            }
        </Payload>
    </Set>
</AssignMessage>
```

---

### 8.3 Reset Calculation Logic (Reusable JavaScript Policy)

This policy computes the `flow.reset_seconds` value using the Quota expiry time. It is designed to be reusable across multiple Quota policies.

**JavaScript policy configuration:**

```
<Javascript name="JS-Calculate-Reset" timeLimit="200">
    <ResourceURL>jsc://calculate-reset.js</ResourceURL>
    <Properties>
        <!-- Default policy name; can be overridden by per-proxy configuration -->
        <Property name="quotaPolicyName">Q-Client-Fairness</Property>
    </Properties>
</Javascript>
```

**Script (**`calculate-reset.js`**):**

```
var defaultPolicyName = properties.quotaPolicyName || "Q-Client-Fairness";

// Prefer dynamic policy name if provided by the runtime
var policyName = context.getVariable("quota.policy.name") || defaultPolicyName;

var expiryTimeVar = "ratelimit." + policyName + ".expiry.time";
var expiryTime = context.getVariable(expiryTimeVar); // expected in ms epoch
var currentTime = Date.now();

if (expiryTime) {
    var deltaMs = expiryTime - currentTime;
    var resetSeconds = Math.ceil(deltaMs / 1000);
    if (resetSeconds < 0) resetSeconds = 0;
    context.setVariable("flow.reset_seconds", resetSeconds);
} else {
    // Fallback: approximate reset using an interval value if available, otherwise default to 60 seconds
    var intervalSeconds =
        context.getVariable("ratelimit." + policyName + ".interval") ||
        60;
    context.setVariable("flow.reset_seconds", intervalSeconds);
}
```

---

### 8.4 SpikeArrest 429 Response (Infrastructure Protection)

```
<AssignMessage name="AM-Set-429-SpikeArrest">
    <Set>
        <StatusCode>429</StatusCode>
        <ReasonPhrase>Too Many Requests</ReasonPhrase>
        <Headers>
            <Header name="Content-Type">application/problem+json</Header>
            <Header name="Retry-After">5</Header>
        </Headers>
        <Payload contentType="application/problem+json">
            {
                "title": "Rate Limit Exceeded",
                "errors": [
                    {
                        "code": "traffic.limit_exceeded",
                        "message": "System load is currently too high to process your request.",
                        "meta": {
                            "retry_advice": "Please wait a few seconds before retrying."
                        }
                    }
                ]
            }
        </Payload>
    </Set>
</AssignMessage>
```

---

### 8.5 Quota 429 Response (Client Fairness / Business Limits)

```
<AssignMessage name="AM-Set-429-Quota">
    <Set>
        <StatusCode>429</StatusCode>
        <ReasonPhrase>Too Many Requests</ReasonPhrase>
        <Headers>
            <Header name="Content-Type">application/problem+json</Header>

            <Header name="RateLimit-Limit">{ratelimit.Q-Client-Fairness.allowed.count}</Header>
            <Header name="RateLimit-Remaining">{ratelimit.Q-Client-Fairness.remaining.count}</Header>

            <Header name="RateLimit-Reset">{flow.reset_seconds}</Header>
            <Header name="Retry-After">{flow.reset_seconds}</Header>
        </Headers>
        <Payload contentType="application/problem+json">
            {
                "title": "Quota Exceeded",
                "errors": [
                    {
                        "code": "traffic.quota_exceeded",
                        "message": "You have reached your request allowance for the current window.",
                        "meta": {
                            "policy": "Quota"
                        }
                    }
                ]
            }
        </Payload>
    </Set>
</AssignMessage>
```

> Teams may generalize the policy name similarly to the JavaScript policy if multiple Quota definitions are used.

---

9. Monitoring and Alerting
--------------------------

Observability is essential to effective traffic management. Teams are expected to configure dashboards and alerts using Apigee Analytics and backend metrics.

### 9.1 Key Metrics

* Count and percentage of requests resulting in `fault.name = "SpikeArrestViolation"`.
* Count and percentage of `fault.name = "QuotaViolation"` by `client_id`.
* Backend performance indicators (e.g., database CPU, latency, thread utilization).
* Effective SpikeArrest rate (configured rate per MP × number of MPs).

### 9.2 Recommended Thresholds

| Metric | Warning Threshold | Critical Threshold | Suggested Action |
| --- | --- | --- | --- |
| SpikeArrest violations (share of traffic) | > 2% | > 5% | Investigate traffic pattern and backend capacity |
| Aggregate Quota violations (across clients) | > 10% of active clients | > 25% of active clients | Review quota settings and client usage profiles |
| Per-client Quota violations | > 100 per hour | > 1000 per hour | Investigate for abuse or misconfiguration |
| Backend latency (P95) during SpikeArrest activity | > 2× baseline | > 5× baseline | Review capacity, queries, and rate limits |

---

10. Security Considerations
---------------------------

* Quota identifiers should always be derived from **verified credentials** (e.g., `apigee.client_id`) and not from user-controlled headers.
* Rate limiting configuration and error responses should not expose internal implementation details such as MP counts, absolute backend thresholds, or internal resource identifiers.
* For anonymous traffic, IP-based limits can help mitigate abuse but should not be relied upon as a strong identity mechanism.

---

11. Client Behavior Expectations
--------------------------------

To interact responsibly with Lumen APIs, client implementations should:

1. Inspect `RateLimit-*` headers on all responses, not just on `429` responses.
2. Implement preemptive throttling when the remaining quota becomes low:

   ```
   remaining = int(response.headers.get("RateLimit-Remaining", "0"))
   reset = int(response.headers.get("RateLimit-Reset", "0"))

   if remaining < 10 and reset > 0:
       time.sleep(reset)
   ```
3. Respect `Retry-After` values on 429 responses and use exponential backoff:

   ```
   wait = int(response.headers.get("Retry-After", "1"))
   wait = min(wait * (2 ** attempt), 300)  # cap at 5 minutes
   time.sleep(wait)
   ```
4. Avoid tight retry loops that disregard Backoff and `Retry-After`.

---

12. Testing and Validation
--------------------------

Before an API is promoted to production, rate limiting behavior should be validated through both targeted tests and load evaluations.

### 12.1 Pre-Production Tests

1. **Quota accuracy test**

   * Send exactly `Allow` requests within a single window.
   * Confirm that the `(Allow + 1)`th request receives a 429 with correct `RateLimit-*` headers.
   * After the `RateLimit-Reset` duration, confirm that the quota is available again.
2. **SpikeArrest load test**

   * Increase traffic up to and beyond `ConfiguredRate_per_MP × Active_MPs`.
   * Confirm that SpikeArrest violations occur before backend saturation.
   * Monitor backend latency and error rates.
3. **Autoscaling test**

   * Trigger MP scale-up to the expected peak count.
   * Verify that Quota behavior does not change and remains accurate.
   * Verify that effective SpikeArrest behavior matches expectations.
4. **Multi-region test (if applicable)**

   * Generate concurrent traffic from multiple regions.
   * Confirm that Quota counters behave correctly and that any replication lag is within acceptable tolerance.

### 12.2 Production Smoke Checks

After production deployment:

* Confirm that 429 responses produced by Quota include `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, and `Retry-After`.
* Confirm that 429 responses produced by SpikeArrest include `Retry-After` only.
* Verify that `flow.reset_seconds` is consistent with actual reset timings (within a small tolerance).
* Confirm that configured alerts trigger appropriately under controlled conditions.

---

13. Common Pitfalls and Preferred Practices
-------------------------------------------

The following table summarizes common misconfigurations and the recommended alternatives:

| Pattern to Avoid | Preferred Practice | Rationale |
| --- | --- | --- |
| Using `request.header.client_id` as a Quota identifier | Use `apigee.client_id` populated by `VerifyAPIKey` or OAuth | Ensures the identifier is trusted and not spoofed |
| Setting very long Quota intervals (e.g., 1 hour) for fairness | Use 5–10 minute windows for most APIs | Reduces the impact of exhaustion on clients while maintaining control |
| Using SpikeArrest as a product tier or contractual limit | Use Quota for contractual limits and tiers | SpikeArrest is node-local and not suitable for contractual guarantees |
| Retrying 429 responses immediately without delay | Honor `Retry-After` and implement exponential backoff | Avoids retry storms and wasted capacity |
| Hard-coding policy names in JavaScript | Use runtime variables (`quota.policy.name`) or policy properties | Improves reusability and reduces configuration drift |
| Testing rate limits with only one MP | Test at the expected peak MP count | Ensures behavior is realistic under scaled conditions |