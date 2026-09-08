---
identifier: "INTG-BP-001"
name: "Webhook Design"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "best-practice"
category: "protocol"
appliesTo: ["webhooks", "events"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-9457"]
  w3c: []
  other: ["CNCF-CloudEvents-v1.0.2", "Standard-Webhooks-Spec", "Stripe-Webhooks", "GitHub-Webhooks"]

taxonomy:
  capability: "event-driven"
  subCapability: "webhook-design"
  layer: "contract"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Webhook payloads use CloudEvents envelope"
    - "HMAC-SHA256 signature verification implemented"
    - "Delivery retry with exponential backoff configured"
    - "Dead letter queue configured for permanently failed endpoints"
    - "Subscription management API available"
    - "Health/ping events implemented for endpoint validation"

dependsOn: ["INTG-STD-015", "INTG-STD-016", "INTG-STD-034", "INTG-STD-035"]
supersedes: ""
---

# Webhook Design

## Purpose

Webhooks are listed in `appliesTo` across nearly every integration standard — error handling, observability, resilience, retries — yet no document addresses webhook-specific design patterns. Without unified guidance, teams build webhook systems with incompatible registration APIs, inconsistent signature schemes, no retry policies, and no dead-letter handling. This best practice consolidates webhook design patterns from Stripe, GitHub, and the Standard Webhooks specification into a cohesive guide for both webhook producers and consumers.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Payload Format

- Webhook payloads **MUST** use the CloudEvents envelope per INTG-STD-015.
- Webhooks **MUST** use structured content mode (JSON) per INTG-STD-015 R-7.
- The `Content-Type` header **MUST** be `application/cloudevents+json; charset=utf-8`.
- The payload **MUST** include all required CloudEvents attributes (`specversion`, `id`, `source`, `type`).

### R-2: Subscription Management API

Webhook producers **SHOULD** provide a subscription management API with the following operations:

| Operation | Method | Endpoint | Description |
|---|---|---|---|
| Create | POST | `/v1/webhooks/subscriptions` | Register a new webhook endpoint |
| List | GET | `/v1/webhooks/subscriptions` | List active subscriptions |
| Get | GET | `/v1/webhooks/subscriptions/{id}` | Get a specific subscription |
| Update | PATCH | `/v1/webhooks/subscriptions/{id}` | Update endpoint URL or event filters |
| Delete | DELETE | `/v1/webhooks/subscriptions/{id}` | Remove a subscription |

- Subscriptions **MUST** include at minimum: `url` (the delivery endpoint), `events` (list of event types to receive), and `secret` (the signing secret, returned only at creation time).
- Subscription endpoints **MUST** validate URL reachability via a verification request (R-6) before activating.

### R-3: Signature Verification

- Every webhook delivery **MUST** include a cryptographic signature for payload integrity and authenticity verification.
- The signing algorithm **MUST** be HMAC-SHA256.
- The signature **MUST** be transmitted in a dedicated header:

| Header | Format | Description |
|---|---|---|
| `Webhook-Signature` | `v1,{base64-encoded-hmac}` | Versioned signature value |
| `Webhook-Timestamp` | Unix timestamp (seconds) | Timestamp of signature generation |
| `Webhook-ID` | UUID | Unique delivery attempt identifier |

- The HMAC input **MUST** be constructed as: `{webhook_id}.{timestamp}.{raw_body}` — concatenated with periods.
- The signing secret **MUST** be at least 32 bytes of cryptographic randomness.
- Signing secrets **MUST** be rotatable without downtime; producers **SHOULD** support dual-signing during rotation (both old and new secrets generate signatures).
- Signing secrets **MUST** be stored and managed per SEC-STD-002 (Secrets Management).

### R-4: Consumer Signature Validation

- Consumers **MUST** verify the HMAC signature on every received webhook before processing the payload.
- Consumers **MUST** reject payloads where the signature does not match with `401 Unauthorized` or `403 Forbidden`.
- Consumers **MUST** validate that `Webhook-Timestamp` is within a tolerance window (e.g., ±5 minutes) to prevent replay attacks.
- Consumers **MUST** use constant-time comparison for HMAC verification to prevent timing attacks.
- Consumers **MUST NOT** process webhook payloads without signature verification, even in development or staging environments.

### R-5: Delivery Retry Policy

- Producers **MUST** retry failed deliveries using exponential backoff with jitter per INTG-STD-034.
- A delivery is considered failed if the consumer returns a non-2xx HTTP status or the connection times out.
- The retry schedule **SHOULD** follow:

| Attempt | Delay | Cumulative |
|---|---|---|
| 1 (immediate) | 0s | 0s |
| 2 | ~30s | ~30s |
| 3 | ~2m | ~2.5m |
| 4 | ~15m | ~17.5m |
| 5 | ~1h | ~1h 17m |
| 6 | ~4h | ~5h 17m |
| 7 (final) | ~12h | ~17h 17m |

- After exhausting retries, the event **MUST** be moved to a dead letter queue (R-8).
- Consumer endpoints **MUST** respond within the timeout budget (per INTG-STD-035); producers **SHOULD** use a 30-second delivery timeout.
- A `429` response from the consumer **MUST** trigger backoff respecting the `Retry-After` header.

### R-6: Endpoint Verification

- Before activating a subscription, producers **SHOULD** verify the endpoint by sending a verification request.
- Verification **SHOULD** use the CloudEvents webhook validation handshake: an HTTP OPTIONS request with `WebHook-Request-Origin` header.
- The consumer **MUST** respond with `200 OK` and the `WebHook-Allowed-Origin` header matching the requested origin.
- Alternatively, producers **MAY** send a ping event (R-7) and require a `200` response.

### R-7: Ping and Health Events

- Producers **SHOULD** support a `ping` event type (e.g., `com.example.webhook.ping`) for endpoint health checks.
- Consumers **MUST** respond to ping events with `200 OK` and an empty body.
- Producers **MAY** periodically send ping events to detect endpoint failures before real events fail.
- Ping events **MUST** conform to the CloudEvents envelope (R-1) but **MAY** have an empty `data` field.

### R-8: Dead Letter Queue

- Producers **MUST** implement a dead letter queue (DLQ) for webhook events that exhaust all retry attempts.
- Dead-lettered events **MUST** be retained for at least 30 days.
- Producers **SHOULD** expose an API or dashboard for consumers to inspect and replay dead-lettered events.
- Dead letter records **MUST** include: the original event, the subscription ID, the number of delivery attempts, and the last error response.

### R-9: Consumer Idempotency

- Consumers **MUST** process webhooks idempotently per INTG-GDL-001.
- Deduplication **MUST** use the CloudEvents `source` + `id` combination (per INTG-STD-015 R-2).
- Consumers **MUST NOT** assume webhooks are delivered exactly once; at-least-once delivery is the guarantee.

### R-10: Subscription Expiry

- Subscriptions **MAY** have an expiry date, after which they are automatically deactivated.
- If subscriptions expire, the producer **MUST** notify the consumer via email or a dedicated notification channel at least 7 days before expiry.
- Producers **SHOULD** support subscription renewal to extend the expiry.

---

## Examples

### Webhook Delivery Request

```http
POST /webhooks/receive HTTP/1.1
Content-Type: application/cloudevents+json; charset=utf-8
Webhook-ID: msg_01HZX3KQVB8E72GQJHF5RM6YWN
Webhook-Timestamp: 1725811200
Webhook-Signature: v1,K7gNU3sdo+OL0wNhqoVWhr3g6s1xYv72ol/pe/Unols=

{
  "specversion": "1.0",
  "id": "01HZX3KQVB8E72GQJHF5RM6YWN",
  "source": "//orders.example.com/checkout",
  "type": "com.example.order.created",
  "time": "2026-09-08T12:00:00.000Z",
  "datacontenttype": "application/json",
  "data": {
    "orderId": "order-8842",
    "totalAmount": { "value": "149.99", "currency": "USD" }
  }
}
```

### Consumer Response — Success

```http
HTTP/1.1 200 OK
```

### Consumer Response — Signature Verification Failed

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://consumer.example.com/problems/invalid-signature",
  "title": "Invalid Webhook Signature",
  "status": 403,
  "detail": "HMAC signature verification failed."
}
```

---

## References

- [Standard Webhooks Specification](https://www.standardwebhooks.com/)
- [CloudEvents HTTP Protocol Binding](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/bindings/http-protocol-binding.md)
- [Stripe Webhooks](https://stripe.com/docs/webhooks) — industry reference implementation
- [GitHub Webhooks](https://docs.github.com/en/webhooks) — industry reference implementation
- INTG-STD-015 - Event Envelope
- INTG-STD-034 - Retry Policy
- INTG-STD-035 - Timeout Standard
- INTG-GDL-001 - Idempotency Design
- SEC-STD-002 - Secrets Management

## Rationale

**Why CloudEvents for webhook payloads?** Using a standard envelope eliminates per-provider payload parsing. CloudEvents is already mandated by INTG-STD-015 for all events crossing service boundaries; webhooks are no exception.

**Why HMAC-SHA256 specifically?** HMAC-SHA256 is the de-facto standard across Stripe, GitHub, Shopify, and the Standard Webhooks spec. It balances security strength with computational efficiency and requires no PKI infrastructure (unlike RSA or ECDSA signatures).

**Why constant-time comparison?** Standard string comparison short-circuits on the first mismatched byte, leaking information about how many bytes match. An attacker can exploit this to forge signatures byte-by-byte.

**Why at-least-once instead of exactly-once?** Exactly-once delivery requires two-phase commit between producer and consumer, which is impractical over HTTP. At-least-once with consumer-side idempotency achieves the same functional result with dramatically simpler infrastructure.

**Why 30-day dead letter retention?** 30 days covers investigation cycles and allows consumers to diagnose and fix endpoint issues. Events older than 30 days are typically stale and reprocessing would cause data inconsistencies.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
