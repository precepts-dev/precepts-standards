---
identifier: "INTG-STD-018"
name: "Partial and Multi-Status Responses"
version: "1.0.0"
status: "DRAFT"

domain: "INTEGRATION"
documentType: "standard"
category: "protocol"
appliesTo: ["api", "batch", "webhooks"]

lastUpdated: "2026-09-08"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: []
  rfc: ["RFC-9110", "RFC-9457", "RFC-4918"]
  w3c: []
  other: ["Google-Cloud-API-Design-Guide", "Zalando-RESTful-API-Guidelines"]

taxonomy:
  capability: "api-design"
  subCapability: "response-handling"
  layer: "contract"

enforcement:
  method: "hybrid"
  validationRules:
    batchResponseFormat: "per-item status with RFC 9457 errors"
    multiStatusCode: 207
  rejectionCriteria:
    - "Batch endpoint returning single top-level status for mixed outcomes"
    - "Per-item errors not using RFC 9457 Problem Details format"
    - "Missing per-item status code in batch response"
    - "Async bulk endpoint not returning 202 with status URI"
  reviewChecklist:
    - "Batch semantics (all-or-nothing vs. best-effort) documented"
    - "Per-item error format follows INTG-STD-009"
    - "Retry guidance provided for partial failures"

dependsOn: ["INTG-STD-008", "INTG-STD-009", "INTG-STD-034"]
supersedes: ""
---

# Partial and Multi-Status Responses

## Purpose

Many integration scenarios involve operations on multiple resources in a single request — batch imports, bulk updates, multi-entity deletions, or composite transactions. When some items succeed and others fail, a simple `200 OK` or `400 Bad Request` loses critical per-item information. This standard defines response formats for partial success, multi-status results, and asynchronous bulk operations, extending INTG-STD-009's error handling to multi-item contexts.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Rules

### R-1: Batch Semantics Declaration

- Every batch or bulk endpoint **MUST** declare its semantics in the API specification as one of:

| Semantics | Behaviour | Use When |
|---|---|---|
| **All-or-nothing** | Entire batch succeeds or fails atomically | Transactional consistency required (e.g., financial ledger entries) |
| **Best-effort** | Each item processed independently; partial success allowed | Independent items (e.g., notification delivery, data imports) |
| **Ordered-sequential** | Items processed in order; processing stops on first failure | Order-dependent pipelines |

- The chosen semantics **MUST** be documented in the API specification and **MUST NOT** change without a major version increment.

### R-2: Multi-Status Response Format

For **best-effort** batch operations with mixed outcomes, the response **MUST** use HTTP `207 Multi-Status` and conform to the following structure:

- The top-level response **MUST** contain a `results` array.
- Each entry in `results` **MUST** include:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `index` | integer | **MUST** | Zero-based position of the item in the request array |
| `status` | integer | **MUST** | HTTP status code for this item (e.g., 201, 400, 409) |
| `resource` | object | **SHOULD** | Created or updated resource representation (on success) |
| `error` | object | **MUST** (on failure) | RFC 9457 Problem Details object (per INTG-STD-009) |

- The top-level response **SHOULD** include a `summary` object with counts:

| Field | Type | Description |
|---|---|---|
| `total` | integer | Total items in the batch |
| `succeeded` | integer | Items that completed successfully |
| `failed` | integer | Items that failed |

### R-3: All-or-Nothing Response

- For **all-or-nothing** batch operations, the server **MUST** use standard HTTP status codes:
  - `200 OK` or `201 Created` when the entire batch succeeds.
  - `422 Unprocessable Content` with an RFC 9457 error response when validation fails, including an `errors` array identifying which items failed (per INTG-STD-009 R-5).
- The server **MUST** roll back all changes if any item fails.
- The response **MUST NOT** use `207` for all-or-nothing operations.

### R-4: Batch Size Limits

- Servers **MUST** enforce a maximum batch size and **MUST** return `413 Content Too Large` when the limit is exceeded.
- The maximum batch size **MUST** be documented in the API specification.
- Servers **SHOULD** support a maximum of 100 items per batch unless a higher limit is explicitly justified and documented.
- Clients **SHOULD** implement client-side chunking for workloads exceeding the batch limit.

### R-5: Asynchronous Bulk Operations

For long-running bulk operations that cannot complete within a synchronous request:

- The server **MUST** return `202 Accepted` with a response body containing:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `operation_id` | string | **MUST** | Unique identifier for the bulk operation |
| `status_uri` | string | **MUST** | URI to poll for operation status |
| `estimated_completion` | string | **MAY** | ISO-8601 UTC timestamp of estimated completion |

- The status endpoint **MUST** return the current state of the operation:

| State | HTTP Status | Description |
|---|---|---|
| `PENDING` | 200 | Operation has not started processing |
| `PROCESSING` | 200 | Operation is in progress; **SHOULD** include progress percentage |
| `COMPLETED` | 200 | All items processed; response includes `results` per R-2 |
| `FAILED` | 200 | Operation failed entirely; response includes RFC 9457 error |

- The server **SHOULD** include a `Retry-After` header on status responses when the operation is `PENDING` or `PROCESSING`.

### R-6: Partial Content (HTTP 206)

- `206 Partial Content` **MUST** only be used for range-based responses on a single resource (e.g., large file downloads).
- Range requests **MUST** use the `Range` header and servers **MUST** include `Content-Range` in the response.
- `206` **MUST NOT** be used for batch or multi-item responses; use `207` instead.

### R-7: Correlation and Tracing

- Multi-status and async bulk responses **MUST** include `X-Correlation-ID` per INTG-STD-016.
- For async operations, the `X-Correlation-ID` from the initial `202` response **MUST** be associated with the status endpoint and all subsequent status responses.
- Per-item errors in batch responses **SHOULD** include `correlationId` in each error object.

### R-8: Retry Guidance for Partial Failures

- Multi-status responses **MUST** include enough information for clients to identify and retry only the failed items.
- The `index` field in each result **MUST** correspond to the original request array position.
- Clients **SHOULD** retry only items with `5xx` status codes; items with `4xx` status codes indicate client errors that require correction before retry.
- Servers **SHOULD** support idempotent retry of individual items within a batch (see INTG-GDL-001).

---

## Examples

### Multi-Status Response (Best-Effort Batch)

```json
HTTP/1.1 207 Multi-Status
Content-Type: application/json
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  },
  "results": [
    {
      "index": 0,
      "status": 201,
      "resource": { "id": "usr-001", "email": "alice@example.com" }
    },
    {
      "index": 1,
      "status": 422,
      "error": {
        "type": "https://api.example.com/problems/validation-error",
        "title": "Validation Error",
        "status": 422,
        "detail": "Email address is already registered.",
        "instance": "/logs/errors/b472f-9a31",
        "correlationId": "550e8400-e29b-41d4-a716-446655440000",
        "errorCode": "USER_VALIDATION_DUPLICATE_EMAIL"
      }
    },
    {
      "index": 2,
      "status": 201,
      "resource": { "id": "usr-003", "email": "charlie@example.com" }
    }
  ]
}
```

### Async Bulk Operation — Initial Response

```json
HTTP/1.1 202 Accepted
Content-Type: application/json
X-Correlation-ID: 3c6e0b8a-9c0a-45af-9db8-0b2e1f4b5c7d

{
  "operation_id": "bulk-import-77f3a",
  "status_uri": "/v1/operations/bulk-import-77f3a",
  "estimated_completion": "2026-09-08T15:30:00.000Z"
}
```

---

## Enforcement Rules

### Gateway-Level

- API gateways **MUST** allow `207 Multi-Status` to pass through without transformation.
- Gateways **MUST NOT** replace `207` with `200` or `500`.

### Build-Time

- OpenAPI specifications for batch endpoints **MUST** document `207` responses with the multi-status schema.
- Contract tests **MUST** validate that per-item errors conform to RFC 9457 format.
- Batch semantics (all-or-nothing vs. best-effort) **MUST** be documented in the operation description.

---

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) — HTTP status codes including 202, 206, 207
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html) — per-item error format
- [RFC 4918 - HTTP Extensions for WebDAV](https://www.rfc-editor.org/rfc/rfc4918.html) — 207 Multi-Status origin
- [Google Cloud API Design Guide - Common Patterns](https://cloud.google.com/apis/design/design_patterns#list_sub-collections) — long-running operations
- INTG-STD-009 - Error Handling
- INTG-STD-016 - HTTP Header Standards

## Rationale

**Why HTTP 207 over 200 with embedded errors?** A `200` response implies complete success to clients, caches, and monitoring tools. `207` explicitly signals mixed outcomes, allowing gateways and observability systems to treat partial failures differently from full successes.

**Why require per-item RFC 9457 errors?** Reusing the same error format from INTG-STD-009 means clients need only one error-parsing strategy across single-resource and batch operations. It also ensures machine-readable error codes and correlation IDs are available for every failed item.

**Why limit batch size to 100?** Large batches increase timeout risk, memory pressure, and retry cost. 100 items balances throughput with reliability. Systems needing higher throughput should use async bulk operations (R-5).

**Why separate async pattern?** Operations exceeding timeout budgets (INTG-STD-035) cannot complete synchronously. The 202 + status URI pattern gives clients a standard way to track long-running work without holding connections open.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-09-08 | Initial definition |
