---
identifier: "INTG-GDL-001"
name: "Idempotency Design"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "guideline"
category: "reliability"
appliesTo: ["api", "events", "webhooks", "batch"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-7232"]
  w3c: []
  other: ["IETF-draft-idempotency-header", "Stripe-Idempotent-Requests", "AWS-Builders-Library"]

taxonomy:
  capability: "reliability"
  subCapability: "idempotency"
  layer: "contract"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "POST and PATCH endpoints support Idempotency-Key header"
    - "Idempotency store TTL configured and documented"
    - "Event consumers deduplicate using source + id"
    - "Conditional request headers supported on mutation endpoints"
    - "Retry behaviour documented for duplicate requests"

dependsOn: ["INTG-STD-008", "INTG-STD-015", "INTG-STD-016", "INTG-STD-034"]
supersedes: ""
---

# Idempotency Design

## Purpose

Idempotency is referenced across multiple integration standards — INTG-STD-034 rejects "retry without idempotency guarantee on non-idempotent operations," and INTG-STD-015 permits event ID reuse for re-delivery — but no document defines *how* to implement idempotency. Without explicit guidance, teams implement conflicting deduplication strategies, leading to duplicate resource creation, lost updates, and retry failures. This guideline establishes patterns for API idempotency keys, event deduplication, and conditional request handling.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Naturally Idempotent Methods

The following HTTP methods are naturally idempotent per RFC 9110 and require no additional mechanism:

| Method | Idempotent | Safe | Notes |
|---|---|---|---|
| `GET` | Yes | Yes | **MUST** not produce side effects |
| `HEAD` | Yes | Yes | **MUST** not produce side effects |
| `OPTIONS` | Yes | Yes | **MUST** not produce side effects |
| `PUT` | Yes | No | Replaces entire resource; repeated calls produce the same state |
| `DELETE` | Yes | No | Deleting an already-deleted resource **SHOULD** return `204` or `404` |

- `POST` and `PATCH` are **NOT** naturally idempotent and **MUST** use the Idempotency-Key pattern (R-2) to enable safe retries.

### R-2: Idempotency-Key Header

- Non-idempotent endpoints (POST, PATCH) **SHOULD** accept an `Idempotency-Key` request header.
- The value **MUST** be a client-generated UUID v4 or ULID.
- If a request with a previously seen `Idempotency-Key` arrives, the server **MUST** return the stored response from the original request without re-executing the operation.
- The server **MUST** return `409 Conflict` if the same `Idempotency-Key` is received with a different request body than the original.
- The `Idempotency-Key` header **MUST** be documented in the API specification.

### R-3: Server-Side Idempotency Store

- Servers implementing idempotency **MUST** maintain a store mapping `Idempotency-Key` values to their corresponding responses.
- The store **MUST** capture the complete response (status code, headers, body) of the first execution.
- Entries **MUST** have a time-to-live (TTL) after which they expire. The TTL **SHOULD** be between 24 and 72 hours.
- The TTL **MUST** be documented in the API specification.
- The store **MUST** be shared across all instances of the service (not node-local) to handle requests routed to different nodes.

### R-4: In-Flight Duplicate Handling

- If a duplicate request arrives while the original is still being processed, the server **SHOULD** return `409 Conflict` with an RFC 9457 error body indicating the request is in progress.
- Alternatively, the server **MAY** block and wait for the original to complete, then return the cached response.
- The server **MUST NOT** execute the operation a second time concurrently.

### R-5: Conditional Requests (Optimistic Concurrency)

- Endpoints supporting resource updates **SHOULD** use `ETag` and conditional request headers for optimistic concurrency:
  - `If-Match` on PUT/PATCH/DELETE — the server **MUST** return `412 Precondition Failed` if the ETag does not match.
  - `If-None-Match: *` on POST — the server **MUST** return `412 Precondition Failed` if the resource already exists.
- ETag values **MUST** change whenever the resource representation changes.
- ETag values **SHOULD** be opaque strings; clients **MUST NOT** parse or construct ETag values.

### R-6: Event Consumer Deduplication

- Event consumers **MUST** deduplicate events using the combination of `source` + `id` per INTG-STD-015 R-2.
- Consumers **MUST** maintain a deduplication store with a configurable TTL.
- The deduplication window **SHOULD** be at least 7 days or match the event retention period of the broker, whichever is longer.
- Consumers **MUST** process events idempotently — reprocessing a previously seen event **MUST** produce no additional side effects.

### R-7: Idempotency in Batch Operations

- For batch endpoints, each item in the batch **MAY** carry its own idempotency key in the request body.
- The top-level `Idempotency-Key` header applies to the entire batch request.
- If per-item idempotency keys are supported, the server **MUST** deduplicate at the item level, not the batch level.
- The chosen idempotency granularity (batch-level vs. item-level) **MUST** be documented in the API specification.

### R-8: Idempotency Key Validation

- The server **SHOULD** validate that `Idempotency-Key` values conform to UUID or ULID format.
- The server **MUST** return `400 Bad Request` if the `Idempotency-Key` is missing when required by the endpoint.
- Empty or whitespace-only `Idempotency-Key` values **MUST** be rejected.

---

## Examples

### Idempotent POST — First Request

```http
POST /v1/payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 01HZX3KQVB8E72GQJHF5RM6YWN
Authorization: Bearer eyJhbGci...

{
  "amount": { "value": "49.99", "currency": "USD" },
  "recipient": "merchant-1234"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "id": "pay-77f3a",
  "status": "completed"
}
```

### Idempotent POST — Duplicate Request (Same Key, Same Body)

```http
HTTP/1.1 201 Created
Content-Type: application/json
X-Correlation-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7

{
  "id": "pay-77f3a",
  "status": "completed"
}
```

Response is the cached original — no new payment created.

### Idempotent POST — Duplicate Key, Different Body

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/idempotency-key-reuse",
  "title": "Idempotency Key Already Used",
  "status": 409,
  "detail": "The Idempotency-Key has been used with a different request body.",
  "correlationId": "3c6e0b8a-9c0a-45af-9db8-0b2e1f4b5c7d",
  "errorCode": "REQUEST_IDEMPOTENCY_KEY_MISMATCH"
}
```

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) — method idempotency definitions
- [RFC 7232 - HTTP Conditional Requests](https://www.rfc-editor.org/rfc/rfc7232.html) — ETag, If-Match, If-None-Match
- [IETF draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) — Idempotency-Key header specification
- [Stripe - Idempotent Requests](https://stripe.com/docs/api/idempotent_requests) — industry reference implementation
- [AWS Builders Library - Making retries safe](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- INTG-STD-015 - Event Envelope
- INTG-STD-034 - Retry Policy

## Rationale

**Why a separate guideline?** Idempotency is a cross-cutting concern affecting API design, event processing, batch operations, and retry logic. Embedding it in any single standard would fragment the guidance.

**Why client-generated keys?** Server-generated idempotency tokens require an extra round trip. Client-generated UUIDs are collision-resistant without coordination, aligning with the approach used by Stripe, PayPal, and Google.

**Why 24-72 hour TTL?** Long enough to cover extended retry windows and weekends, short enough to prevent unbounded storage growth. Stripe uses 24 hours; most payment processors use 24-48 hours.

**Why 409 for mismatched bodies?** Silently ignoring the mismatch would hide bugs. Returning `422` implies the *new* request is invalid, which is misleading. `409 Conflict` accurately describes the state conflict — the key is already bound to a different operation.

**Why ETag over version counters?** ETags are HTTP-native, widely supported by caches and proxies, and semantically represent resource state. Version counters are application-specific and leak implementation details.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
