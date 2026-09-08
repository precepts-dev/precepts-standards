---
identifier: "INTG-STD-019"
name: "Pagination Standard"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "standard"
category: "protocol"
appliesTo: ["api", "graphql"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-8288"]
  w3c: []
  other: ["Google-AIP-158", "Zalando-RESTful-API-Guidelines", "Relay-GraphQL-Cursor-Connections-Spec"]

taxonomy:
  capability: "api-design"
  subCapability: "pagination"
  layer: "contract"

enforcement:
  method: "hybrid"
  validationRules:
    defaultStrategy: "cursor-based"
    maxPageSize: 100
    defaultPageSize: 25
  rejectionCriteria:
    - "Collection endpoint without pagination support"
    - "Offset-based pagination without cursor alternative on high-cardinality endpoints"
    - "Missing page_size parameter on collection endpoints"
    - "Page size exceeding documented maximum without explicit override"
  reviewChecklist:
    - "Pagination strategy documented in API specification"
    - "Default and maximum page sizes configured"
    - "Stable sort order guaranteed for paginated results"
    - "Total count included only where performance permits"

dependsOn: ["INTG-STD-004", "INTG-STD-008", "INTG-STD-016"]
supersedes: ""
---

# Pagination Standard

## Purpose

INTG-STD-008 mandates that collection endpoints **MUST** support pagination (rejection criteria: "Missing pagination on collection endpoints") but prescribes no implementation rules. Without a uniform pagination contract, consumers face inconsistent query parameters, unpredictable response shapes, and unstable result sets across APIs. This standard defines the required pagination strategies, query parameters, response envelope, and constraints.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Cursor-Based Pagination as Default

- All collection endpoints **MUST** support cursor-based pagination as the primary strategy.
- Offset-based pagination **MAY** be offered as an additional option but **MUST NOT** be the sole strategy for endpoints with high cardinality (> 10,000 items).
- The choice of strategy **MUST** be documented in the API specification.

### R-2: Standard Query Parameters

Collection endpoints **MUST** accept the following query parameters:

| Parameter | Type | Default | Requirement | Description |
|---|---|---|---|---|
| `page_size` | integer | 25 | **MUST** | Number of items per page |
| `page_token` | string | (none) | **MUST** (cursor) | Opaque cursor for the next page |
| `offset` | integer | 0 | **MAY** (offset) | Zero-based offset for offset pagination |

- Parameter names **MUST** follow `snake_case` per INTG-STD-004.
- `page_token` values **MUST** be opaque strings. Clients **MUST NOT** construct, parse, or modify cursor values.
- Servers **MUST** return `400 Bad Request` for invalid or expired `page_token` values.

### R-3: Response Envelope

Paginated responses **MUST** use the following envelope structure:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `data` | array | **MUST** | Array of result items for the current page |
| `pagination` | object | **MUST** | Pagination metadata object |

The `pagination` object **MUST** include:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `next_cursor` | string \| null | **MUST** | Cursor for the next page; `null` when no more pages exist |
| `has_more` | boolean | **MUST** | `true` if additional pages exist beyond the current page |

The `pagination` object **MAY** include:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `previous_cursor` | string \| null | **MAY** | Cursor for the previous page; enables bidirectional navigation |
| `total_count` | integer | **MAY** | Total number of items across all pages (see R-6) |

### R-4: Page Size Constraints

- Servers **MUST** enforce a maximum page size and return `400 Bad Request` if the requested `page_size` exceeds it.
- The maximum page size **SHOULD** be 100 unless a higher limit is explicitly justified and documented.
- The default page size **SHOULD** be 25.
- A `page_size` of 0 **MUST** be rejected with `400 Bad Request`.
- Servers **MAY** silently cap `page_size` to the maximum instead of rejecting, but **MUST** reflect the actual page size used in the response.

### R-5: Stable Sort Order

- Paginated results **MUST** maintain a deterministic, stable sort order across pages.
- If the client does not specify a sort parameter, the server **MUST** apply a default sort that guarantees stability (e.g., by primary key, creation timestamp + ID).
- The default sort order **MUST** be documented in the API specification.
- Results **MUST NOT** shift between pages due to concurrent inserts or deletes when using cursor-based pagination.

### R-6: Total Count

- Servers **MAY** include `total_count` in the pagination response when the count can be computed efficiently.
- Servers **SHOULD NOT** include `total_count` when the computation requires a full table scan on high-cardinality datasets.
- When `total_count` is not provided, clients **MUST** rely on `has_more` to determine if additional pages exist.
- The decision to include or omit `total_count` **MUST** be documented in the API specification and **MUST** be consistent across requests to the same endpoint.

### R-7: Link Headers

- Servers **SHOULD** include RFC 8288 `Link` headers with `rel="next"` (and optionally `rel="prev"`) pointing to the next page URI.
- Link headers **MUST** include the full URI including query parameters.
- Link headers complement (not replace) the `pagination` response body object.

### R-8: Deep Pagination Protection

- Servers **MUST** reject offset-based pagination requests where `offset` exceeds a configured threshold (e.g., 10,000) to prevent performance degradation.
- Servers **SHOULD** return `400 Bad Request` with an error message recommending cursor-based pagination for deep result sets.
- Cursor-based pagination has no such limit by design — cursors encode the position within the result set without requiring the server to skip rows.

### R-9: Empty Collections

- An empty page **MUST** return `200 OK` with `data` as an empty array (`[]`) and `has_more` as `false`.
- Empty pages **MUST NOT** return `404 Not Found`; the collection resource exists, it simply has no items.

### R-10: GraphQL Pagination

- GraphQL APIs **MUST** implement the Relay Cursor Connections specification for paginated fields.
- Connection types **MUST** include `edges`, `nodes`, and `pageInfo`.
- `pageInfo` **MUST** include `hasNextPage`, `hasPreviousPage`, `startCursor`, and `endCursor`.
- The `first`/`after` and `last`/`before` argument pairs **MUST** be supported.

---

## Examples

### Cursor-Based — Request

```http
GET /v1/orders?page_size=10&page_token=eyJpZCI6Ijk5OSJ9 HTTP/1.1
Accept: application/json
```

### Cursor-Based — Response

```json
{
  "data": [
    { "id": "order-1000", "status": "shipped" },
    { "id": "order-1001", "status": "pending" }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6IjEwMDEifQ",
    "has_more": true,
    "total_count": 4521
  }
}
```

```http
Link: </v1/orders?page_size=10&page_token=eyJpZCI6IjEwMDEifQ>; rel="next"
```

### Last Page — Response

```json
{
  "data": [
    { "id": "order-4520", "status": "delivered" }
  ],
  "pagination": {
    "next_cursor": null,
    "has_more": false,
    "total_count": 4521
  }
}
```

### Non-Compliant — Missing Pagination on Collection

```json
HTTP/1.1 200 OK
{
  "orders": [...]
}
```

**Violation:** No `pagination` object, no `page_size` support. All collection endpoints **MUST** support pagination.

---

## Enforcement Rules

### Gateway-Level

- API gateways **SHOULD** enforce maximum `page_size` values to protect backend services from oversized page requests.
- Gateways **SHOULD** block offset-based requests exceeding the deep pagination threshold.

### Build-Time

- OpenAPI specifications **MUST** define `page_size` and `page_token` parameters on all collection operations.
- OpenAPI response schemas **MUST** include the `pagination` object with `next_cursor` and `has_more`.
- Linters **SHOULD** flag collection endpoints missing pagination query parameters.

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 8288 - Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html) — Link headers with rel="next"
- [Google AIP-158 - Pagination](https://google.aip.dev/158)
- [Zalando RESTful API Guidelines - Pagination](https://opensource.zalando.com/restful-api-guidelines/#pagination)
- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
- INTG-STD-004 - Naming Conventions
- INTG-STD-008 - Resource Design

## Rationale

**Why cursor-based as default?** Offset pagination degrades linearly with depth — a query with `OFFSET 100000` must skip 100,000 rows. Cursor-based pagination uses indexed lookups regardless of depth, making it O(1) for page retrieval.

**Why opaque cursors?** Exposing cursor internals (e.g., raw IDs) creates tight coupling between client and server implementation. Opaque tokens allow servers to change cursor encoding without breaking clients.

**Why not keyset pagination as the default name?** "Keyset pagination" and "cursor-based pagination" describe the same mechanism. "Cursor" is the more widely understood term across REST and GraphQL ecosystems (Google AIP-158, Stripe, Relay).

**Why default page size of 25?** Small enough to be fast across most datasets, large enough to reduce round trips. Google (20), Stripe (10), GitHub (30) cluster around this range.

**Why protect against deep offset pagination?** Without a threshold, `offset=10000000` forces the database to materially scan and skip rows, consuming CPU and memory proportional to the offset. This is a denial-of-service vector on public APIs.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
