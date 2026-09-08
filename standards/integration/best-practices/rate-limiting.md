---
identifier: "INTG-BP-002"
name: "Rate Limiting and Throttling"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "best-practice"
category: "reliability"
appliesTo: ["api", "webhooks", "events", "graphql", "grpc"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-6585"]
  w3c: []
  other: ["IETF-draft-httpapi-ratelimit-headers", "Google-Cloud-API-Design-Guide", "Stripe-Rate-Limiting"]

taxonomy:
  capability: "reliability"
  subCapability: "rate-limiting"
  layer: "infrastructure"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Rate limits configured on all public-facing endpoints"
    - "Rate-limit response headers present on rate-limited endpoints"
    - "429 responses include Retry-After header"
    - "Consumer backoff logic implemented for 429 responses"
    - "Rate limits documented in API specification"
    - "Quota design reviewed for fairness across consumers"

dependsOn: ["INTG-STD-009", "INTG-STD-016", "INTG-STD-033", "INTG-STD-034"]
supersedes: ""
---

# Rate Limiting and Throttling

## Purpose

Rate limiting is mentioned in INTG-STD-009 (HTTP 429 status code), INTG-STD-015 (429 triggers backoff for event delivery), and INTG-STD-016 (rate-limit response headers), but no document addresses rate-limiting strategy, quota design, or throttling patterns holistically. Without unified guidance, APIs are either over-restrictive (blocking legitimate consumers) or under-protected (vulnerable to abuse and cascade failures). This best practice establishes patterns for both rate-limit providers and consumers.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Rate Limiting Required

- All public-facing API endpoints **MUST** enforce rate limits.
- Internal service-to-service APIs **SHOULD** enforce rate limits to prevent cascade failures.
- Rate limits **MUST** be documented in the API specification, including limits, window size, and reset behaviour.

### R-2: Rate-Limiting Algorithms

Producers **SHOULD** select a rate-limiting algorithm based on their traffic pattern:

| Algorithm | Behaviour | Best For |
|---|---|---|
| **Token bucket** | Allows bursts up to bucket capacity; refills at a steady rate | APIs with bursty traffic patterns |
| **Sliding window** | Counts requests in a rolling time window | Smooth, predictable rate enforcement |
| **Fixed window** | Counts requests in fixed time intervals | Simple implementation; acceptable for low-stakes endpoints |
| **Leaky bucket** | Processes requests at a fixed rate; queues excess | Smoothing traffic to downstream dependencies |

- **Token bucket** is **RECOMMENDED** as the default algorithm for most APIs because it balances burst tolerance with steady-state enforcement.
- **Fixed window** **SHOULD NOT** be used on high-traffic endpoints due to the boundary-burst problem (double the allowed rate at window boundaries).

### R-3: Rate-Limit Response Headers

Rate-limited endpoints **MUST** include response headers per INTG-STD-016 R-6:

- `RateLimit-Policy` — quota definition.
- `RateLimit` — current consumption and reset timing.
- `Retry-After` — **MUST** be present on 429 responses; **SHOULD** be present on 503 responses.

### R-4: HTTP 429 Response

When a consumer exceeds the rate limit:

- The server **MUST** return `429 Too Many Requests`.
- The response body **MUST** use RFC 9457 Problem Details format (per INTG-STD-009).
- The `Retry-After` header **MUST** be present with the number of seconds until the consumer can retry.
- The `errorCode` **SHOULD** distinguish between rate limit types (e.g., `RATE_LIMIT_PER_CLIENT`, `RATE_LIMIT_PER_ENDPOINT`, `RATE_LIMIT_GLOBAL`).

### R-5: Quota Design

- Rate limits **SHOULD** be scoped per authenticated client identity (OAuth 2.0 `client_id` or equivalent credential).
- Unauthenticated endpoints **MAY** scope limits per source IP address, but producers **MUST** account for shared IP addresses (NAT, proxies, corporate egress).
- API keys **MAY** be used as a client identification mechanism for legacy systems, but **MUST NOT** be the authentication mechanism (per SEC-STD-001). OAuth 2.0 client credentials **SHOULD** be used for new integrations.
- Producers **SHOULD** define tiered quota levels (e.g., free, standard, premium) with documented escalation paths.
- Quotas **MAY** be scoped at multiple levels simultaneously (per-client AND per-endpoint).

### R-6: Consumer-Side Rate Limit Handling

- Consumers **MUST** respect `429 Too Many Requests` responses and back off per the `Retry-After` header.
- Consumers **MUST NOT** retry immediately on 429; they **MUST** wait at least the duration specified by `Retry-After`.
- Consumers **SHOULD** implement proactive throttling by monitoring `RateLimit` response headers and reducing request rate before hitting limits.
- Consumers **SHOULD** implement exponential backoff with jitter (per INTG-STD-034) when `Retry-After` is not present.
- Consumers **MUST NOT** retry indefinitely on persistent 429 responses; a maximum retry count **MUST** be configured.

### R-7: Burst Allowance

- Rate limiters **SHOULD** allow short bursts above the sustained rate to accommodate legitimate traffic spikes.
- The burst allowance **SHOULD** be documented (e.g., "100 requests/minute sustained, burst up to 20 requests in 1 second").
- Burst capacity **MUST NOT** be replenished instantly after exhaustion; it **SHOULD** refill at the sustained rate.

### R-8: Quota Reset Semantics

- The rate-limit window reset time **MUST** be communicated via the `RateLimit` response header.
- Consumers **MUST NOT** assume a specific reset schedule (e.g., top of the minute); they **MUST** rely on the values in the response headers.
- Reset times **SHOULD** be staggered across clients to prevent thundering herd at window boundaries.

### R-9: Rate Limiting in Event-Driven Systems

- Event consumers **SHOULD** implement backpressure mechanisms when they cannot keep up with the event production rate.
- Event brokers **SHOULD** support consumer group-level rate limits.
- Webhook producers **MUST** respect consumer `429` responses and apply backoff per INTG-BP-001 R-5.
- Event consumers **MUST NOT** reject events solely due to rate; they **SHOULD** buffer and process at their sustainable rate.

### R-10: API Gateway Enforcement

- Rate limits **SHOULD** be enforced at the API gateway layer, not within individual services.
- Gateway-enforced limits **MUST** use a shared, distributed counter (not node-local) to prevent per-node bypass.
- Gateways **MUST** inject rate-limit response headers even when the request is within limits.
- When the gateway rate limiter is unavailable, the gateway **SHOULD** fail-open (allow requests) rather than fail-closed (reject all), to avoid total outage during limiter failures. The fail-open/fail-closed policy **MUST** be explicitly configured.

### R-11: Graceful Degradation

- When approaching capacity, services **SHOULD** implement progressive degradation before hard rejection:

| Stage | Action |
|---|---|
| **Normal** | Full functionality |
| **Warning** (80% of limit) | Reduce response fidelity (e.g., omit `total_count` in pagination) |
| **Throttled** (100% of limit) | Return 429 with `Retry-After` |
| **Overloaded** | Shed low-priority traffic; return 503 |

- The degradation stages **SHOULD** be monitored and alerted on per INTG-STD-029 (Observability).
- Circuit breakers (INTG-STD-033) **SHOULD** be configured to trip when downstream 429 rates exceed a threshold.

---

## Examples

### Rate Limit Exceeded — 429 Response

```json
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 30
RateLimit-Policy: 100;w=60
RateLimit: limit=100, remaining=0, reset=30
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "type": "https://api.example.com/problems/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Client has exceeded 100 requests per 60 seconds. Retry after 30 seconds.",
  "instance": "/logs/errors/r4t3-l1m1t",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "errorCode": "RATE_LIMIT_PER_CLIENT",
  "timestamp": "2026-09-08T14:30:00.000Z"
}
```

### Normal Response with Rate-Limit Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json
RateLimit-Policy: 100;w=60
RateLimit: limit=100, remaining=87, reset=42
X-Correlation-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7
```

### Consumer Backoff Pseudocode

```
function call_api(request):
    response = http.send(request)

    if response.status == 429:
        wait_seconds = parse_int(response.headers["Retry-After"])
        sleep(wait_seconds + random_jitter())
        return call_api(request)  // retry

    if response.headers["RateLimit"]:
        remaining = parse_remaining(response.headers["RateLimit"])
        if remaining < 10:
            // proactively slow down
            sleep(1)

    return response
```

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 6585 - Additional HTTP Status Codes (429)](https://www.rfc-editor.org/rfc/rfc6585.html)
- [IETF draft-ietf-httpapi-ratelimit-headers](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Stripe Rate Limiting](https://stripe.com/docs/rate-limits) — industry reference
- [Google Cloud API Design Guide - Quotas](https://cloud.google.com/apis/design/design_patterns#quota)
- [INTG-STD-009 - Error Handling](internal) — 429 response format
- [INTG-STD-016 - HTTP Header Standards](internal) — rate-limit headers
- [INTG-STD-033 - Resilience Patterns](internal) — circuit breakers
- [INTG-STD-034 - Retry Policy](internal) — exponential backoff with jitter
- [SEC-STD-001 - Identity and Access Management](internal) — OAuth 2.0 client credentials for client identification

## Rationale

**Why token bucket as default?** Token bucket naturally handles bursty traffic patterns typical of API consumers (batch operations followed by idle periods). Sliding window is stricter but penalises legitimate bursts. Fixed window has the boundary-burst problem where clients can send double the rate at window boundaries.

**Why gateway-level enforcement?** Enforcing rate limits within individual services means every service must implement rate limiting independently, and scaling services adds rate limit capacity. Gateway-level enforcement provides a single control plane, consistent policy, and prevents bypass via service-to-service calls.

**Why fail-open on limiter failure?** A failed rate limiter that rejects all traffic causes a total outage. Fail-open preserves availability at the cost of temporary over-admission, which is typically less severe than a complete service outage.

**Why prohibit API keys as authentication?** API keys are static, long-lived, and lack standard expiry, rotation, or scope mechanisms. They are routinely leaked in logs, URLs, and source repositories. OAuth 2.0 client credentials provide short-lived tokens with built-in expiry, rotation, and scope. API keys are acceptable **ONLY** as a client *identification* mechanism (for quota tracking) on legacy systems that cannot adopt OAuth.

**Why progressive degradation?** Hard cutoffs create cliff effects — one request over the limit causes full rejection. Progressive degradation preserves partial functionality, improving consumer experience and reducing retry storms.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
