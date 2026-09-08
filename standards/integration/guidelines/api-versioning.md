---
identifier: "INTG-GDL-002"
name: "API Versioning"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "guideline"
category: "versioning"
appliesTo: ["api", "graphql", "grpc", "events"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-8594"]
  w3c: []
  other: ["Google-AIP-181", "Zalando-RESTful-API-Guidelines", "Microsoft-REST-API-Guidelines"]

taxonomy:
  capability: "versioning"
  subCapability: "api-versioning"
  layer: "contract"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "API version in URL path for REST endpoints"
    - "Deprecation and Sunset headers configured for deprecated versions"
    - "Migration guide published before sunset date"
    - "Maximum concurrent versions not exceeded"
    - "Event schema version suffix present for breaking changes"

dependsOn: ["INTG-STD-004", "INTG-STD-006", "INTG-STD-008", "INTG-STD-015", "INTG-STD-016"]
supersedes: ""
---

# API Versioning

## Purpose

INTG-STD-006 establishes backward and forward compatibility rules — *what* constitutes a breaking change and how to evolve schemas without breaking consumers. This guideline addresses the complementary question: *how* to version APIs when breaking changes are unavoidable, how to communicate deprecation, and how long to maintain legacy versions. Without explicit versioning guidance, teams version inconsistently (URL path vs. header vs. query parameter) and deprecate without notice, causing consumer breakage.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: URL-Path Versioning as Default

- REST APIs **MUST** include the major version in the URL path as the first path segment after the base path: `/v{major}/...` (e.g., `/v1/orders`, `/v2/orders`).
- The version segment **MUST** use the format `v{N}` where `N` is a positive integer (no leading zeros, no `v0`).
- Only the major version number appears in the URL; minor and patch changes **MUST** be backward compatible and **MUST NOT** require a URL change (per INTG-STD-006).

### R-2: Alternative Versioning Strategies

- Header-based versioning (e.g., `Accept: application/vnd.example.v2+json`) **MAY** be used for internal APIs with tightly coupled consumers, but **MUST NOT** be the sole strategy for public or partner APIs.
- Query parameter versioning (e.g., `?version=2`) **MUST NOT** be used; it pollutes cache keys and is not semantically part of the resource identifier.
- When header-based versioning is used, the API **MUST** define a default version for requests without the version header and **MUST** document this default.

### R-3: Breaking Change Definition

A change is breaking and requires a new major version if it:

- Removes or renames an existing field, endpoint, or query parameter.
- Changes the type or semantics of an existing field.
- Narrows an enum (removes allowed values).
- Changes error response codes for the same input.
- Removes or restricts a previously permitted operation.
- Alters authentication or authorisation requirements.

Non-breaking changes (additive fields, new optional parameters, new endpoints, widening enums) **MUST NOT** trigger a new major version. See INTG-STD-006 for the full compatibility taxonomy.

### R-4: Deprecation Lifecycle

API versions **MUST** follow a three-phase lifecycle:

| Phase | Duration | Description |
|---|---|---|
| **ACTIVE** | Indefinite | Fully supported; receives bug fixes and security patches |
| **DEPRECATED** | Minimum 6 months | Functional but no new features; consumers are notified to migrate |
| **SUNSET** | End of life | Version is permanently shut down; requests return `410 Gone` |

- The minimum deprecation period **MUST** be 6 months for internal APIs and **SHOULD** be 12 months for public or partner APIs.
- APIs **MUST NOT** move directly from ACTIVE to SUNSET without a deprecation period.

### R-5: Deprecation and Sunset Headers

- Deprecated API versions **MUST** include the following headers on every response:

| Header | Format | Requirement | Description |
|---|---|---|---|
| `Deprecation` | HTTP-date (RFC 9110 §5.6.7) | **MUST** | Date when the version was deprecated |
| `Sunset` | HTTP-date (RFC 8594) | **MUST** | Date when the version will be shut down |
| `Link` | `<migration-guide-url>; rel="successor-version"` | **SHOULD** | Link to the migration guide or successor version |

- Sunset responses (after the sunset date) **MUST** return `410 Gone` with an RFC 9457 error body referencing the successor version.

### R-6: Migration Guide Requirements

- Before deprecating a version, a migration guide **MUST** be published documenting:
  - Breaking changes between the old and new versions.
  - Field-by-field mapping from old to new schemas.
  - Code examples showing before/after request and response patterns.
  - Timeline with deprecation and sunset dates.
- The migration guide **SHOULD** be linked from the `Deprecation` response header via `Link`.

### R-7: Maximum Concurrent Versions

- No more than **2** major versions of an API **SHOULD** be concurrently active.
- If a third major version is introduced, the oldest active version **MUST** enter the DEPRECATED phase.
- Exceptions require explicit approval from the Integration Architecture Board and **MUST** be documented with a justification.

### R-8: Event Schema Versioning

- Event schema versions **MUST** be conveyed via the CloudEvents `type` suffix per INTG-STD-015 R-4 (e.g., `com.example.order.created.v2`).
- A new version suffix **MUST** only be appended for breaking schema changes.
- Consumers **MUST** be able to subscribe to specific versions of an event type.
- Producers **SHOULD** emit both the old and new event versions during the deprecation period to allow gradual consumer migration.

### R-9: gRPC Versioning

- gRPC services **MUST** version via the Protobuf `package` name (e.g., `example.orders.v1`, `example.orders.v2`).
- Both old and new packages **MUST** be served concurrently during the deprecation period.
- Individual RPC methods **MAY** be deprecated using the `deprecated` field option in the `.proto` definition.

### R-10: GraphQL Versioning

- GraphQL APIs **SHOULD** avoid versioning the entire schema. Instead, deprecated fields **MUST** be marked with `@deprecated(reason: "...")`.
- Deprecated fields **MUST** remain functional for the minimum deprecation period (R-4).
- When an entire GraphQL schema is restructured, a new endpoint path **MAY** be used (e.g., `/graphql/v2`).

---

## Examples

### URL-Path Versioned API

```http
GET /v1/orders/order-8842 HTTP/1.1
Accept: application/json
```

### Deprecated Version Response Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json
Deprecation: Sat, 01 Mar 2026 00:00:00 GMT
Sunset: Mon, 01 Sep 2026 00:00:00 GMT
Link: <https://docs.example.com/api/migration/v1-to-v2>; rel="successor-version"
```

### Post-Sunset Response

```json
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/version-sunset",
  "title": "API Version Sunset",
  "status": 410,
  "detail": "API v1 was permanently shut down on 2026-09-01. Please migrate to v2.",
  "instance": "/v1/orders/order-8842",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "errorCode": "API_VERSION_SUNSET"
}
```

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 8594 - The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)
- [Google AIP-181 - Stability Levels and Versioning](https://google.aip.dev/181)
- [Zalando RESTful API Guidelines - Deprecation](https://opensource.zalando.com/restful-api-guidelines/#deprecation)
- [Microsoft REST API Guidelines - Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
- INTG-STD-006 - Backward and Forward Compatibility
- INTG-STD-015 - Event Envelope

## Rationale

**Why URL-path versioning over headers?** URL-path versioning is the most visible and debuggable approach — you can see the version in browser address bars, curl commands, and log files. Header-based versioning is invisible at the HTTP level and requires tooling support.

**Why prohibit query parameter versioning?** Query parameters are part of the cache key. Version-in-query causes cache fragmentation and is semantically incorrect — the version is a property of the resource contract, not a filter on the data.

**Why minimum 6-month deprecation?** Consumer migration involves code changes, testing, and deployment cycles. Six months accommodates quarterly release cycles. Public APIs need longer because external consumers have less control over their release schedules.

**Why max 2 concurrent versions?** Each additional version multiplies testing, documentation, and operational burden. More than 2 concurrent versions indicates a failure to sunset aggressively and typically leads to divergent implementations.

**Why emit both event versions during deprecation?** Unlike APIs where consumers actively call a version, event consumers passively subscribe. Dual-emitting allows consumers to migrate their subscription at their own pace without a hard cutover.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
