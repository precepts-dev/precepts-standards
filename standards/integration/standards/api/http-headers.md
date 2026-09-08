---
identifier: "INTG-STD-016"
name: "HTTP Header Standards"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "standard"
category: "protocol"
appliesTo: ["api", "webhooks", "events", "grpc", "graphql", "a2a", "mcp"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-6648", "RFC-8288", "RFC-9457", "RFC-6585"]
  w3c: ["Trace-Context"]
  other: ["IETF-draft-httpapi-ratelimit-headers", "Zalando-RESTful-API-Guidelines", "OWASP-Secure-Headers-Project"]

taxonomy:
  capability: "api-design"
  subCapability: "http-headers"
  layer: "contract"

enforcement:
  method: "hybrid"
  validationRules:
    correlationHeader: "X-Correlation-ID"
    customHeaderPrefix: "no X- prefix per RFC 6648"
    contentType: "required on all request and response bodies"
  rejectionCriteria:
    - "Missing X-Correlation-ID on API responses"
    - "Custom headers using deprecated X- prefix"
    - "Credentials transmitted in query strings or custom headers instead of Authorization"
    - "Missing Content-Type on request or response bodies"
  reviewChecklist:
    - "All required standard headers present"
    - "Custom headers follow organisational naming convention"
    - "Rate-limit headers present on rate-limited endpoints"
    - "Security-sensitive headers configured"

dependsOn: ["INTG-STD-004", "INTG-STD-009", "INTG-STD-029"]
supersedes: ""
---

# HTTP Header Standards

## Purpose

HTTP headers carry essential metadata — correlation identifiers, content negotiation, caching directives, security controls, and rate-limit signals — yet no existing integration standard governs them holistically. This standard unifies header requirements that are currently scattered across INTG-STD-009 (error handling), INTG-STD-029 (observability), and INTG-STD-015 (events) into a single, enforceable specification. Without it, teams independently invent header conventions, causing interoperability failures at gateways, inconsistent observability, and security gaps.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Required Standard Headers

Every HTTP API response **MUST** include the following headers:

| Header | Value | Requirement |
|---|---|---|
| `Content-Type` | Media type of the response body | **MUST** be present on all responses with a body |
| `X-Correlation-ID` | UUID v4 — end-to-end correlation | **MUST** be present on all responses |
| `X-Request-ID` | UUID v4 — per-hop request identifier | **SHOULD** be present; **MUST** be unique per request |

Every HTTP API request carrying a body **MUST** include the `Content-Type` header.

### R-2: Correlation and Tracing Headers

- If the inbound request includes `X-Correlation-ID`, the server **MUST** propagate it unchanged in the response and to all downstream calls.
- If the inbound request does not include `X-Correlation-ID`, the server **MUST** generate a new UUID v4 and include it in the response.
- The `X-Correlation-ID` response header **MUST** match the `correlationId` field in RFC 9457 error response bodies (per INTG-STD-009 R-7).
- Servers **SHOULD** propagate W3C `traceparent` and `tracestate` headers per INTG-STD-029 for distributed tracing.
- `X-Request-ID` **MUST NOT** be reused across retries; each attempt is a distinct request.

### R-3: Content Negotiation

- Clients **SHOULD** include the `Accept` header to indicate preferred response media types.
- Servers **MUST** respect `Accept` when multiple representations are available and **MUST** return `406 Not Acceptable` when the requested type cannot be served.
- `Content-Type` **MUST** include the `charset` parameter when the media type does not imply UTF-8. For `application/json`, the charset parameter is **OPTIONAL** (RFC 8259 mandates UTF-8).
- Servers **SHOULD** include `Content-Language` when serving localised content.

### R-4: Custom Header Naming

- Custom headers **MUST NOT** use the `X-` prefix for new headers (per RFC 6648, which deprecates the convention for newly defined headers).
- The only exception is `X-Correlation-ID` and `X-Request-ID`, which are retained for backward compatibility and widespread ecosystem adoption.
- Organisation-specific custom headers **SHOULD** use a short organisational namespace prefix (e.g., `Acme-Tenant-ID`, `Precepts-Trace-Source`).
- Custom header names **MUST** use `Title-Case-With-Hyphens` (HTTP/1.1 convention) and **MUST NOT** use underscores.
- Custom headers **MUST** be documented in the API specification (OpenAPI `headers` object or equivalent).

### R-5: Caching Headers

- Servers **MUST** include `Cache-Control` on cacheable responses.
- Servers **SHOULD** include `ETag` or `Last-Modified` on resource representations to support conditional requests.
- `Cache-Control: no-store` **MUST** be set on responses containing sensitive or user-specific data.
- Clients **SHOULD** send `If-None-Match` or `If-Modified-Since` for conditional GET requests.

### R-6: Rate-Limit Headers

Rate-limited endpoints **MUST** communicate limits using the following headers:

| Header | Description | Requirement |
|---|---|---|
| `RateLimit-Policy` | Quota definition (e.g., `100;w=60`) | **SHOULD** be present on all responses from rate-limited endpoints |
| `RateLimit` | Remaining quota and reset window | **SHOULD** be present on all responses from rate-limited endpoints |
| `Retry-After` | Seconds or HTTP-date to wait before retrying | **MUST** be present on 429 responses; **SHOULD** be present on 503 responses |

- `Retry-After` **MUST** use seconds (integer) format for programmatic consumption. HTTP-date format **MAY** be used additionally.
- Rate-limit header values **MUST NOT** leak internal infrastructure details (e.g., node-specific counters).

### R-7: Authentication Headers

- Authentication credentials **MUST** be transmitted in the `Authorization` header using a registered scheme (e.g., `Bearer`, `Basic`).
- Credentials **MUST NOT** be transmitted in query strings, custom headers, or cookies for API-to-API communication.
- The `Authorization` header **MUST** carry short-lived tokens (OAuth 2.0 access tokens per SEC-STD-001); API keys **MAY** be used only for legacy system identification and **MUST NOT** be the sole authentication mechanism for new integrations.
- The `WWW-Authenticate` header **MUST** be present on 401 responses, indicating the required authentication scheme.

### R-8: Security Response Headers

API responses **SHOULD** include the following security headers where applicable:

| Header | Value | Applicability |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | **MUST** on all HTTPS endpoints (per SEC-STD-003) |
| `X-Content-Type-Options` | `nosniff` | **SHOULD** on all responses |
| `X-Frame-Options` | `DENY` | **SHOULD** on API responses not intended for iframe embedding |

- API responses **MUST NOT** include `Server` headers that reveal software name or version.
- API responses **MUST NOT** include `X-Powered-By` or similar technology-disclosure headers.

### R-9: Header Size Constraints

- Total request header size **MUST NOT** exceed 8 KiB across all headers combined.
- A single header value **SHOULD NOT** exceed 4 KiB.
- Servers **MUST** return `431 Request Header Fields Too Large` when header limits are exceeded.

### R-10: Forbidden Headers

The following headers **MUST NOT** be used in API integrations:

- `X-Forwarded-For` **MUST NOT** be trusted or acted upon without gateway-level validation and sanitisation.
- `X-Real-IP` **MUST NOT** be used as a substitute for authenticated identity.
- Custom headers carrying JSON-encoded payloads **MUST NOT** be used; structured data belongs in the request or response body.

---

## Examples

### Compliant Response Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000
X-Request-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7
Cache-Control: private, max-age=300
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
RateLimit-Policy: 100;w=60
RateLimit: limit=100, remaining=87, reset=42
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
```

### Compliant 429 Response

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000
Retry-After: 30
RateLimit-Policy: 100;w=60
RateLimit: limit=100, remaining=0, reset=30
```

### Non-Compliant — Credential in Query String

```http
GET /v1/orders?api_key=sk_live_abc123 HTTP/1.1
```

**Violation:** Credentials in query string (violates R-7). Credentials **MUST** be in the `Authorization` header.

---

## Enforcement Rules

### Gateway-Level

- API gateways **MUST** inject `X-Correlation-ID` if the backend has not provided it.
- API gateways **MUST** strip `Server` and `X-Powered-By` headers from outbound responses.
- API gateways **SHOULD** validate `Content-Type` presence on request and response bodies.

### Build-Time

- OpenAPI specifications **MUST** declare required headers (`X-Correlation-ID`, `Content-Type`) on all operations.
- Linters **SHOULD** flag custom headers using the deprecated `X-` prefix for new definitions.
- Contract tests **MUST** validate that 429 responses include `Retry-After`.

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 6648 - Deprecating X- in Protocol Parameters](https://www.rfc-editor.org/rfc/rfc6648.html)
- [RFC 8288 - Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)
- [RFC 6585 - Additional HTTP Status Codes (431)](https://www.rfc-editor.org/rfc/rfc6585.html)
- [IETF draft-ietf-httpapi-ratelimit-headers](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- INTG-STD-009 - Error Handling
- INTG-STD-029 - Integration Observability
- SEC-STD-001 - Identity and Access Management
- SEC-STD-003 - Cryptography and Transport Security

## Rationale

**Why unify header standards?** Header rules are scattered across error handling (R-7 in INTG-STD-009), observability (INTG-STD-029), and event bindings (INTG-STD-015 R-8). A single authoritative document prevents contradictions and gives gateway teams one reference for enforcement configuration.

**Why retain X-Correlation-ID despite RFC 6648?** RFC 6648 deprecates the `X-` prefix for *new* headers, but `X-Correlation-ID` and `X-Request-ID` are de-facto standards with near-universal gateway, framework, and observability tooling support. Renaming would fracture the ecosystem for no material benefit.

**Why prohibit credentials in query strings?** Query strings are logged by proxies, CDNs, and browser history. OWASP and SEC-STD-001 mandate the `Authorization` header to prevent credential leakage.

**Why mandate RateLimit headers?** Without proactive rate-limit signals, clients resort to trial-and-error, causing burst overloads. The IETF draft standardises a machine-readable format that clients and gateways can consume programmatically.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
