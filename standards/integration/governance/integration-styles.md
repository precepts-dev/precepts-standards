---
identifier: "INTG-GOV-001"
name: "Integration Styles"
version: "1.1.0"
status: "MANDATORY"

domain: "INTEGRATION"
documentType: "governance"
category: "governance"
appliesTo: ["api", "events", "a2a", "files", "mcp", "webhooks", "grpc",
            "graphql", "batch", "streaming"]

lastUpdated: "2026-04-12"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: ["ISO-IEC-42001", "ISO-IEC-18384"]
  rfc: ["RFC-9110"]
  w3c: []
  other: ["Hohpe-Woolf-EIP", "CNCF-CloudEvents-1.0.2",
          "AsyncAPI-3", "MCP-Spec-2025-11-25", "Google-A2A-2025"]

taxonomy:
  capability: "integration-patterns"
  subCapability: "style-governance"
  layer: "infrastructure"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Integration style selected from the approved catalogue"
    - "Style selection justified against the Selection Framework"
    - "Companion standards identified and compliance acknowledged"
    - "Machine-readable schema specification committed (OpenAPI/AsyncAPI/Protobuf)"
    - "Bounded context boundaries respected per R-7"
    - "AI/agent semantic descriptions present where applicable"
    - "Human oversight gate defined for AI agent side-effect operations"
    - "MCP/A2A tools registered in tool catalogue before exposure"
    - "Lifecycle state documented (ACTIVE / DEPRECATED / SUNSET)"

dependsOn: ["INTG-STD-004", "INTG-STD-006", "INTG-STD-015", "INTG-STD-029",
            "INTG-STD-033", "INTG-STD-034", "INTG-STD-035"]
supersedes: ""
---

# Integration Styles

## Purpose

Every service boundary is an integration contract. The style chosen
determines its coupling characteristics, failure modes, observability
requirements, and long-term evolution cost — for the humans maintaining
it, the services consuming it, and the AI agents navigating it. This
governance document establishes a decision framework, an approved style
catalogue, and lifecycle governance for all integration boundaries in
the Precepts domain. Implementation rules live in companion standards;
this document governs the selection process and the criteria that make
selection correct.

> *Normative language (**MUST**, **SHOULD**, **MAY**) follows
> RFC 2119 semantics.*

---

## Conceptual Model

### Coupling Taxonomy

An **integration style** is a communication pattern governing how two or
more participants — services, agents, humans, or AI models — exchange
information across a boundary. Each style commits to a coupling profile
across four dimensions:

| Dimension | Definition |
|---|---|
| **Temporal** | Whether both sides must be co-available at the moment of exchange |
| **Semantic** | Depth of shared understanding of data meaning, schema, and intent required |
| **Platform** | Infrastructure dependency introduced (broker, codec, runtime) |
| **Interface** | Explicitness and rigidity of the published contract |

| Style | Temporal | Semantic | Platform | Interface | Direction |
|---|---|---|---|---|---|
| **REST / HTTP API** | High | Moderate | Low | Explicit | Request-response |
| **GraphQL** | High | High | Moderate | Explicit | Request-response |
| **gRPC** | High | High | Moderate | Very explicit | Request-response / stream |
| **Event-Driven** | None | High | High | Explicit | Unidirectional push |
| **Webhooks** | Moderate | Moderate | Low | Moderate | Unidirectional push |
| **File / Batch** | None | High | Moderate | Implicit | Transfer |
| **WebSocket / SSE** | High | Moderate | Moderate | Moderate | Bi / server-push |
| **MCP** | High | **Very high** | Moderate | Explicit | Request-response |
| **A2A** *(experimental)* | Varies | **Very high** | Moderate | Emerging | Task delegation |

Temporal = None means producers and consumers operate fully
independently. Semantic = Very High flags styles where a technically
correct contract can be functionally unusable if natural-language
descriptions are imprecise — this applies specifically to all
AI-consumed integrations.

### Core Selection Dimensions

Evaluate these five dimensions before selecting a style:

| Dimension | Key question |
|---|---|
| **Response need** | Must the caller receive a result before proceeding? |
| **Consumer type** | Service, human client, or AI agent / LLM? |
| **Data characteristic** | CRUD, hierarchical, high-volume stream, bulk transfer, real-time bidirectional? |
| **Coupling tolerance** | Same team, different teams, or external party? |
| **Persistence need** | Must consumers replay history or reconstruct past state? |

---

## Integration Style Profiles

### REST / HTTP API

Synchronous resource-oriented HTTP communication. Stateless, cacheable,
universally understood by proxies, gateways, and tooling. The default
integration style for request-response interactions.

**Use when:**
- The caller requires a synchronous response before proceeding
- The operation is resource-oriented (CRUD on a named entity)
- The consumer population is diverse or includes external third parties
- HTTP caching of responses is desirable

**Do not use when:**
- The operation is inherently asynchronous (prefer events)
- Sub-millisecond p99 latency between internal services is required (prefer gRPC)
- The primary consumer is an AI agent needing tool semantics (prefer MCP)
- Continuous server-push updates are required (prefer SSE)

**Key production trade-off:** Temporal coupling is unavoidable. In a
chain of five synchronous REST calls, end-to-end latency is the sum of
all hops. Without deadline propagation (INTG-STD-035) and circuit
breakers (INTG-STD-033), a single slow downstream causes full-chain
degradation.

**Companion standards:** INTG-STD-008, INTG-STD-009, INTG-STD-004,
INTG-STD-006, INTG-STD-033, INTG-STD-034, INTG-STD-035

**AI/Agent:** Every REST API **MUST** have a current, machine-readable
OpenAPI 3.x specification. APIs without specs or with naming violations
(INTG-STD-004) are effectively opaque to LLM-based agents regardless of
technical correctness.

---

### GraphQL

Schema-first query API where clients declare exactly the fields they
need. Strongly typed, introspectable, optimised for diverse consumer
shapes accessing shared underlying data.

**Use when:**
- Multiple consumer types need substantially different data shapes from
  the same source
- Over-fetching or under-fetching is a documented problem
- A Backend-for-Frontend layer serves web, mobile, and voice clients
- Hierarchical data traversal is the primary access pattern

**Do not use when:**
- Operations are uniform CRUD (REST is simpler and more cacheable)
- The team cannot commit to DataLoader batching — the N+1 problem is
  imperceptible at low traffic and catastrophic at production volume
- POST-based queries bypassing HTTP cache infrastructure is a concern
- Simple event notification is needed

**Key production trade-off:** GraphQL resolvers can over-fetch from
backing services even when clients request only a few fields.
DataLoader batching and server-side query depth/complexity limits are
prerequisites before exposing GraphQL to external consumers.

**Companion standards:** INTG-STD-004, INTG-STD-006 (field deprecation),
INTG-STD-029, INTG-STD-009

**AI/Agent:** GraphQL's introspection capability (`__schema`) allows AI
agents to discover the full schema programmatically — a significant
discoverability advantage. Depth and complexity limits **MUST** be
enforced server-side to prevent agents from issuing unbounded expensive
queries.

---

### gRPC

High-performance RPC over HTTP/2 using Protocol Buffers as the contract.
Generates strongly-typed, cross-language client/server stubs.
60–80% bandwidth reduction vs JSON.

**Use when:**
- High-throughput, low-latency internal service-to-service communication
  is required
- Polyglot environments benefit from generated type-safe clients
- Bidirectional or server-side streaming is a core interaction model
- Payload bandwidth efficiency is critical

**Do not use when:**
- Browser clients are primary consumers (requires gRPC-Web proxy)
- Standard HTTP proxies, API gateways, or WAFs are in the path
- External partner APIs are the use case (prefer REST externally)
- Contract volatility is high — protobuf field number management is a
  permanent discipline

**Key production trade-off:** Field numbers in protobuf definitions are
permanent. Reusing or renaming a field number creates silent
wire-compatibility breaks. Teams choosing gRPC commit to protobuf
discipline across the lifetime of that interface.

**Companion standards:** INTG-STD-004 (proto field naming), INTG-STD-006
(field number and wire-compatibility rules), INTG-STD-029 (deadline
propagation), INTG-STD-035 (gRPC deadline configuration)

**AI/Agent:** gRPC is poorly suited for direct AI agent consumption — no
language-agnostic discovery, generated stubs are language-specific.
Prefer REST or MCP when AI agents are primary consumers.

---

### Event-Driven (Queue / Pub-Sub / Stream)

Asynchronous integration where producers emit events to a broker and
consumers subscribe independently. Temporal coupling is eliminated;
semantic coupling through event schemas must be actively managed.

Fowler identifies four sub-patterns with distinct trade-offs:

| Sub-pattern | What it does | When to prefer |
|---|---|---|
| **Event Notification** | Signals state change; consumer queries for details | Minimal payload; acceptable when secondary fetch latency is tolerable |
| **Event-Carried State Transfer** | Event contains full state; consumer is self-sufficient | Preferred for AI consumption; removes need for callback |
| **Event Sourcing** | Events are the system of record; state derived by replay | Audit trails, CQRS, temporal queries |
| **CQRS** | Separated read/write models; read fed by events | Complex domains with asymmetric read/write scale — not a default pattern |

Infrastructure variants:
- **Message Queue** — point-to-point work distribution to one consumer
- **Publish-Subscribe** — fan-out to independent consumers
- **Event Streaming** — persistent ordered log; consumer groups; replay

**Use when:**
- Temporal decoupling is required — producer must not block on consumer
- Multiple independent consumers react to the same business fact
- Audit trail or replay capability is required
- Domain events represent facts other domains must observe without
  direct dependency

**Do not use when:**
- The caller needs an immediate synchronous result
- Strict cross-service transactional consistency is required without
  implementing the Saga pattern with compensating transactions
- Event volume is low and broker overhead is not justified

**Key production trade-off:** Schema leakage and vague event names
(e.g., `CustomerChanged` with no indication of causation) are the
leading production failure modes. Consumers coupled to an imprecise
schema cannot evolve independently. INTG-STD-015 (CloudEvents) +
INTG-STD-004 naming discipline + FULL_TRANSITIVE schema registry
compatibility are non-negotiable controls.

**Companion standards:** INTG-STD-015, INTG-STD-004 (R-17–R-20),
INTG-STD-034 (retry and DLQ), INTG-STD-006 (FULL_TRANSITIVE schema
registry), INTG-STD-029

**AI/Agent:** Events with full state (event-carried state transfer) are
natural for AI agent consumption — an auditable record of system changes
without requiring secondary API calls. Events **MUST** carry sufficient
state for an AI consumer to act without calling back to the producer.

---

### Webhooks

HTTP callbacks from a server to a pre-registered consumer URL. Low
infrastructure overhead. Reverse of REST — the producer initiates.

**Use when:**
- Notifying an external third party of a state change
- Message broker infrastructure is not justified
- The consumer is a known, stable, publicly reachable endpoint
- Delivery volume is moderate and retry semantics are manageable

**Do not use when:**
- Fan-out to many independent consumers is needed (use pub-sub)
- The consumer endpoint is ephemeral, private, or AI agent-hosted
- Strict delivery ordering is required
- High throughput makes per-delivery HTTP overhead prohibitive

**Key production trade-off:** Network topology coupling. The producer
must reach the consumer's endpoint. Firewall rules, dynamic
infrastructure, or private-network receivers cause silent webhook
failure without a DLQ. This makes webhooks unsuitable for
machine-to-machine integration in dynamic cloud infrastructure.

**Companion standards:** INTG-STD-015 (CloudEvents HTTP binding),
INTG-STD-034 (retry schedule and DLQ), INTG-STD-035 (receiver-side
timeout), INTG-STD-006 (payload schema versioning)

**AI/Agent:** Webhooks require a stable, publicly addressable HTTP
endpoint — architecturally incompatible with ephemeral AI agent
runtimes. AI agents **SHOULD** consume domain events via a broker
rather than receiving webhooks.

---

### File and Batch Transfer

Bulk data exchange via structured files (CSV, JSON Lines, Parquet, Avro)
over object storage or file transfer protocols. The lowest-overhead
style for large datasets at the cost of latency.

**Use when:**
- Data volume makes per-record API overhead prohibitive
- The receiving system has limited API capability (legacy, partner,
  regulated counterparty)
- Full-dataset or historical loads are required (ETL, backfill,
  regulatory reporting)
- Latency is not a constraint and batch windows are acceptable

**Do not use when:**
- Per-record latency matters (use events or REST)
- Fine-grained per-record error handling is required
- The consumer needs to query subsets of the data

**Key production trade-off:** File schema changes affect all consumers
simultaneously and are hard to detect automatically. Without schema
versioning embedded in file names, headers, and manifest metadata,
format drift breaks downstream consumers silently — often discovered
only when the next batch window runs.

**Companion standards:** INTG-STD-004 (R-31/R-32 file naming),
INTG-STD-001/002 (date/datetime in records and filenames), INTG-STD-003
(monetary format in records), INTG-STD-006 (schema versioning)

**AI/Agent:** Batch files are the hardest style for AI agents — no
discovery, no schema introspection, no per-record error signalling. AI
data pipelines **SHOULD** prefer event streaming. When files are
unavoidable, manifests with embedded schema URIs enable automated
AI-driven validation.

---

### WebSocket / SSE (Streaming)

WebSocket provides full-duplex bidirectional communication over a
persistent TCP connection. Server-Sent Events (SSE) provides
unidirectional server-to-client streaming over standard HTTP. SSE covers
approximately 95% of real-time use cases at lower operational cost.

**Use SSE when:**
- Server pushes a continuous stream to a client (dashboards,
  notifications, LLM token streaming)
- Unidirectional server-push is sufficient
- HTTP/2 compatibility and standard proxy infrastructure are valued

**Use WebSocket when:**
- Full bidirectional communication is simultaneously required
  (collaborative editing, real-time trading)
- Low-latency bidirectional exchange is the core product feature

**Do not use either when:**
- Server-to-server integration is the use case (use events)
- The interaction is a discrete request-response (use REST)
- Connection count at scale is a concern

**Key production trade-off:** Persistent connections require a different
operational model — load balancers must support connection affinity;
horizontal scaling is more complex; reconnection logic is mandatory
client-side. SSE avoids most of this by using standard HTTP.

**Companion standards:** INTG-STD-029, INTG-STD-035 (stale transfer
detection), INTG-STD-004 (message format naming)

**AI/Agent:** SSE is the natural delivery mechanism for streaming AI
model outputs (token-by-token LLM responses). AI systems generating
progressive output **SHOULD** use SSE rather than buffering full
responses in a REST endpoint.

---

### MCP (Model Context Protocol)

JSON-RPC-based protocol (Anthropic, 2024) enabling AI models and
LLM-based agents to invoke **tools** (actions with side effects), read
**resources** (contextual data), and receive **prompts** (structured
interaction templates) from external systems. MCP is the integration
style of choice when the primary consumer is a language model.

Its unique governance requirement is not technical correctness but
**semantic completeness** — a tool that an AI agent cannot correctly
interpret is equivalent to a broken API.

**Use when:**
- The primary consumer is an AI model or LLM-based agent
- Capabilities must be described semantically for AI reasoning, not
  just structurally for code generation
- The integration is part of an AI-augmented workflow where intent
  drives system actions
- Tool discoverability by AI agents is the governing requirement

**Do not use when:**
- The consumer is a traditional service or human client (use REST)
- High-throughput machine-to-machine integration is required
- Authorization enforcement at the protocol level is required —
  MCP delegates all authorization to the implementor

**Key production trade-off:** Tool descriptions are effectively prompts
for the AI. Vague or incomplete descriptions cause AI agents to misuse
tools, hallucinate parameters, or fail to invoke tools at all. This
failure mode is invisible in unit tests and only discovered in agent
evaluation.

**Companion standards:** INTG-STD-004 (tool and resource URI naming),
INTG-STD-008 (if REST endpoints back MCP tools), INTG-STD-034
(all tools **MUST** be idempotent or document non-idempotency with
idempotency key support), INTG-STD-035 (tool call timeout),
INTG-STD-029 (tracing for tool invocations)

**AI/Agent — mandatory description elements:** Every MCP tool **MUST**
include: (1) what the tool does in one sentence, (2) side effects or an
explicit "no side effects" statement, (3) each parameter's meaning and
valid range, (4) conditions under which the tool must NOT be called.

---

### A2A — Agent-to-Agent *(Experimental)*

Protocol (Google, April 2025) standardising communication between AI
agents across system boundaries — task delegation, capability discovery,
and coordinated multi-agent workflows.

**Status: EXPERIMENTAL.** The protocol is under active development;
production failure patterns are not fully documented; governance
frameworks are nascent. Teams **MUST** follow R-6 before production
adoption.

**Use when:**
- Multiple specialised AI agents must collaborate on a task exceeding
  a single agent's capability or context window
- Genuine capability distribution across agents justifies coordination
  overhead
- Capability discovery across an agent network is a product requirement

**Do not use when:**
- A single AI agent with appropriate MCP tools can accomplish the task
- Strict determinism and full auditability are required — multi-agent
  coordination produces emergent behaviours harder to audit than
  single-agent workflows
- The A2A protocol has not been evaluated for production stability in
  your stack

**Key production trade-off:** Multi-agent systems introduce emergent
failure modes absent from single-agent systems — coordination loops,
capability duplication, cross-agent state inconsistency. Without
immutable per-action logging, reconstructing what the agent network did
and why is infeasible in regulated environments.

**Companion standards:** INTG-STD-004, INTG-STD-029 (distributed trace
propagation across agent boundaries is mandatory), INTG-STD-006
(capability schema evolution)

**Human oversight requirement:** Any A2A workflow with real-world side
effects (writes, communications, financial transactions) **MUST** include
a defined human oversight gate before execution. The gate design,
escalation path, and override mechanism **MUST** be documented in the
service integration manifest.

---

## Selection Framework

### Primary Decision Flow

```
Is an immediate synchronous response required?

YES →
  Primary consumer is an AI agent / LLM?
    YES → MCP
  Real-time bidirectional (collaborative editing, live trading)?
    YES → WebSocket
  Server-push stream (LLM output, live dashboard, notifications)?
    YES → SSE
  High-throughput low-latency internal service-to-service?
    YES → gRPC
  Complex hierarchical data serving diverse consumer shapes?
    YES → GraphQL
  Default → REST / HTTP API

NO →
  Bulk data exchange (ETL, partner files, regulatory reporting)?
    YES → File / Batch
  Lightweight notification to external callback URL?
    YES → Webhook
  Multi-agent task delegation? (evaluate experimental readiness first)
    YES → A2A
  Default → Event-Driven:
    Multiple independent consumers need the same event?
      → Publish-Subscribe
    Work distributed to exactly one consumer?
      → Message Queue
    Replay or historical audit trail required?
      → Event Streaming
```

### Selection Criteria Matrix

| Criterion | REST | GQL | gRPC | Events | WH | Batch | WS/SSE | MCP | A2A |
|---|---|---|---|---|---|---|---|---|---|
| Sync response | ✓ | ✓ | ✓ | — | — | — | ✓ | ✓ | ~ |
| Temporal decoupling | — | — | — | ✓ | ✓ | ✓ | — | — | ~ |
| >10k RPS throughput | ~ | ~ | ✓ | ✓ | — | ✓ | — | — | — |
| AI agent consumer | ~ | ~ | — | ~ | — | — | ~ | ✓ | ~ |
| External / public API | ✓ | ~ | — | — | ✓ | ✓ | — | — | — |
| Schema self-discovery | ~ | ✓ | — | ~ | ~ | — | — | ✓ | ~ |
| Replay / audit trail | — | — | — | ✓ | — | ~ | — | — | — |
| Multi-consumer fan-out | — | — | — | ✓ | — | ~ | — | — | — |
| Real-time server push | — | — | ~ | ~ | ~ | — | ✓ | — | — |
| Standard HTTP tooling | ✓ | ✓ | — | — | ✓ | — | ~ | ✓ | — |

✓ strong fit  ~  partial fit  — poor fit or not applicable

---

## Rules

### R-1: Selection Authority

Style selection decisions are classified into three authority tiers:

| Tier | Criteria | Authority |
|---|---|---|
| **Team Discretion** | Choosing from the approved catalogue; companion standards followed; bounded context scope | Owning team |
| **Architecture Review** | Introducing new broker or shared infrastructure; cross-domain event schema; consumer count >10; new MCP server exposed org-wide | IAB review required |
| **Governance Approval** | Style not in the catalogue (R-6); A2A in production; cross-organisational integration boundary; style conflict between teams sharing a boundary | IAB approval + Architecture Decision Record |

Teams **MUST** document their style selection and the applied selection
criteria in the service integration manifest before first production
deployment.

### R-2: Companion Standards Obligation

These cross-cutting standards apply to **every** integration boundary
regardless of style:

- **INTG-STD-004** — naming conventions for all contracts, fields,
  event types, endpoint paths, and tool names
- **INTG-STD-006** — backward/forward compatibility; breaking change
  process
- **INTG-STD-029** — observability; W3C trace context propagation;
  structured logging

Style-specific companion standards are listed in each profile. Teams
**MUST** identify applicable standards at integration design time, not
retrospectively.

Every integration boundary **MUST** have a machine-readable schema
specification committed alongside its code:

| Style | Specification format |
|---|---|
| REST | OpenAPI 3.x |
| GraphQL | GraphQL SDL + introspection |
| gRPC | Protocol Buffers `.proto` |
| Event-Driven / Webhooks | AsyncAPI 3.x |
| MCP | MCP tool/resource schema per specification |
| A2A | A2A capability manifest |
| File / Batch | JSON Schema or Avro schema with file manifest |

### R-3: Style Coexistence Limits

- A single bounded context **SHOULD NOT** expose more than three
  distinct integration styles simultaneously. Beyond three, cognitive
  and operational overhead becomes disproportionate.
- When a boundary already uses three styles and a new one is proposed,
  the team **MUST** evaluate whether an existing style satisfies the
  requirement before requesting Architecture Review.
- Style inventory **MUST** be reviewed as part of quarterly architecture
  health checks.

### R-4: Lifecycle Governance

Every integration boundary (API, event stream, webhook channel, MCP
server, batch pipeline) **MUST** carry a documented lifecycle state:

| State | Meaning |
|---|---|
| **ACTIVE** | In production, actively maintained, consumers may depend on it |
| **DEPRECATED** | No new consumers permitted; existing consumers have a sunset date; INTG-STD-006 migration window active |
| **SUNSET** | No longer available; all consumers confirmed migrated |

The IAB **MUST** maintain a registry of all cross-domain integration
boundaries and their lifecycle states. Deprecation **MUST** follow the
Breaking Change Process in INTG-STD-006.

### R-5: AI and Agent Integration Governance

These rules apply when an AI model, LLM, or autonomous agent is a
producer or consumer of an integration boundary:

- **Semantic completeness:** Every MCP tool, A2A capability, and
  AI-consumed API **MUST** include natural-language descriptions of:
  purpose, side effects (or explicit "none"), parameter semantics, and
  conditions under which the operation must not be called.
- **Idempotency:** All operations invokable by AI agents **MUST** be
  idempotent **OR** **MUST** explicitly document non-idempotency and
  provide idempotency key support (INTG-STD-034 R-6).
- **Audit immutability:** All AI agent actions **MUST** produce immutable
  log entries including: agent identity, capability invoked, parameters
  (redacted per INTG-STD-029 R-4), timestamp, and outcome.
- **Human oversight gates:** Any AI agent workflow producing real-world
  side effects **MUST** define a human oversight gate with a documented
  escalation path and override mechanism in the integration manifest.
- **Tool catalogue registration:** Every MCP server and A2A capability
  set **MUST** be registered in the tool catalogue before exposure to
  AI agents. Unregistered tools **MUST** be rejected at the gateway.

### R-6: New Style Adoption Process

A style not in the approved catalogue **MUST NOT** be adopted in
production without:

1. **Proposal** — Architecture Proposal documenting: the style, the
   problem existing styles cannot solve, known trade-offs, and a pilot
   scope (single team, ≤90 days, internal only)
2. **Evaluation** — IAB evaluates within 30 days. A pilot period
   **MAY** be approved for low-risk internal contexts
3. **Governance addition** — A successful pilot **MUST** produce a full
   style profile added to this document before broader adoption
4. **Status progression** — New styles start at `EXPERIMENTAL`;
   promotion to `APPROVED` requires documented production usage across
   two or more independent bounded contexts

### R-7: Bounded Context Alignment

- External-facing boundaries (public, partner, or cross-organisation)
  **MUST** use styles with strong backward-compatibility guarantees:
  REST, GraphQL, Webhooks, File/Batch, or CloudEvents (INTG-STD-015).
- Internal boundaries between tightly co-owned services **MAY** use
  gRPC, message queues, or SSE where the coupling trade-off is
  explicitly accepted and documented.
- Cross-domain boundaries within the same organisation **SHOULD** prefer
  event-driven styles to preserve domain autonomy and enable independent
  deployment.

---

## Cross-Cutting Requirements

| Requirement | Standard | Applies to |
|---|---|---|
| Naming conventions | INTG-STD-004 | All styles |
| Backward/forward compatibility | INTG-STD-006 | All styles |
| Observability and tracing | INTG-STD-029 | All styles |
| Resilience patterns | INTG-STD-033 | All outbound calls |
| Retry policy and DLQ | INTG-STD-034 | All styles |
| Timeout configuration | INTG-STD-035 | All styles |
| Machine-readable schema specification | See R-2 | All styles |

---

## Examples

### Compliant selection — Team Discretion (R-1 Tier 1)

A team building an internal order-management service needs synchronous
request-response between two services they own. They document in their
integration manifest:

```
Style: REST / HTTP API
Justification: synchronous response required; same-team services;
  CRUD operations on order resource
Companion standards: INTG-STD-008, INTG-STD-009, INTG-STD-004,
  INTG-STD-006, INTG-STD-033, INTG-STD-035
Schema spec: openapi/orders.v1.yaml committed at /api/openapi/
Lifecycle: ACTIVE
```

No IAB review required. The selection matches the Primary Decision Flow
and all companion standards are identified.

### Compliant selection — Architecture Review required (R-1 Tier 2)

A team wants to publish domain events consumed by 12 downstream services
across four bounded contexts:

```
Style: Event-Driven (Publish-Subscribe)
Justification: temporal decoupling required; 12 consumers cannot be
  tightly coupled to producer availability; audit trail needed
Consumer count: 12 (triggers Architecture Review per R-1)
New infrastructure: Kafka topic on shared cluster
Companion standards: INTG-STD-015, INTG-STD-004 (R-17–R-20),
  INTG-STD-034, INTG-STD-006 (FULL_TRANSITIVE schema registry)
Schema spec: asyncapi/order-events.v1.yaml
Lifecycle: ACTIVE
```

IAB review required because consumer count exceeds 10 and a shared
infrastructure component is involved. The review validates that the
CloudEvents envelope, schema registry compatibility mode, and DLQ
configuration are in place before production approval.

### Non-compliant selection (would be blocked)

A team proposes to expose internal service calls to an AI agent via a
REST API with no OpenAPI spec and no semantic descriptions:

```
Style: REST / HTTP API
Notes: "AI agent will call our existing endpoints directly"
```

**Blocked for three violations:**
1. No OpenAPI specification — violates R-2 (machine-readable schema
   specification required for all styles)
2. No semantic descriptions for AI consumption — violates R-5
   (semantic completeness required when AI agent is a consumer)
3. MCP should be evaluated — the Primary Decision Flow mandates
   evaluating MCP when the primary consumer is an AI agent

The team must either add a complete OpenAPI spec + semantic annotations
and justify why MCP is not appropriate, or adopt MCP with full tool
descriptions per R-5.

---



Architecture Reviews and integration manifests **MUST** verify:

1. Style justified against the Selection Framework or prior IAB decision
2. Style in the approved catalogue or Governance Approval obtained (R-1)
3. Companion standards listed and compliance acknowledged (R-2)
4. Machine-readable schema specification committed alongside code (R-2)
5. Semantic descriptions present for all AI-consumed boundaries (R-5)
6. Human oversight gate defined for AI agent side-effect workflows (R-5)
7. MCP/A2A tools registered in the tool catalogue before exposure (R-5)
8. Lifecycle state documented (R-4)

Non-compliance at Architecture Review blocks production progression.
Non-compliance discovered post-deployment **MUST** be remediated within
two sprints or documented with an accepted risk exception signed by the
domain owner and IAB.

---

## References

### Normative

- [INTG-STD-004 — Naming Conventions](../standards/foundational/naming-conventions.md)
- [INTG-STD-006 — Backward/Forward Compatibility](../standards/versioning/backward-forward-compatibility.md)
- [INTG-STD-015 — Event Envelope](../standards/events/event-envelope.md)
- [INTG-STD-029 — Integration Observability](../standards/observability/integration-observability.md)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [CloudEvents v1.0.2](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md)
- [AsyncAPI Specification v3](https://www.asyncapi.com/)
- [MCP Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)

### Informative

- [Enterprise Integration Patterns — Hohpe & Woolf](https://www.enterpriseintegrationpatterns.com/)
- [Building Microservices, 2nd Ed. — Sam Newman](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
- [What Do You Mean by "Event-Driven"? — Fowler (2017)](https://martinfowler.com/articles/201701-event-driven.html)
- [Data on the Outside vs. Inside — Pat Helland, ACM Queue](https://queue.acm.org/detail.cfm?id=3415014)
- [Google A2A Protocol (2025)](https://github.com/a2aproject/A2A)
- [OWASP Gen AI Security — LLM Top 10 (2025)](https://genai.owasp.org/)
- [ISO/IEC 42001:2023 — AI Management Systems](https://www.iso.org/standard/42001)
- [NIST AI Agent Standards Initiative (2026)](https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [Microsoft Agent Governance Toolkit (2026)](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents/)

---

## Rationale

**Why nine styles and not fewer?** Netflix, Uber, Zalando, and Google
all operate four or more integration styles simultaneously. Prescribing
fewer would create artificial restrictions that teams circumvent with
ungoverned custom protocols. Governing nine named styles with explicit
trade-offs is more effective than denying a heterogeneous reality.

**Why are MCP and A2A first-class styles?** Precepts is an AI-native
platform. The Semantic = Very High rating in the Coupling Taxonomy
captures a unique governance challenge: a technically correct tool that
an AI agent cannot correctly interpret is equivalent to a broken API.
Treating AI integration as an afterthought would contradict the
platform's core value proposition.

**Why coexistence rather than forced convergence?** Forced convergence
creates migration debt that frequently exceeds the governance overhead
of managed heterogeneity. R-3's three-style limit imposes discipline
without requiring costly rewrites; R-4's lifecycle governance enables
organic consolidation over time.

**On second-order effects examined during design:**

REST proliferation → mandatory OpenAPI specs → API catalogue emerges →
AI agents navigate the integration landscape → spec completeness becomes
self-reinforcing because tooling fails without it.

Event schema sprawl → FULL_TRANSITIVE registry (INTG-STD-006) + naming
discipline (INTG-STD-004) → schema governance moves to design time, not
consumer failure time.

MCP tool sprawl (documented at AWS, Dataiku, Gravitee as "agent sprawl")
→ R-5 tool catalogue registration + R-3 coexistence limit → primary and
secondary controls at different organisational layers.

**Governance paradox:** Clear selection criteria and pre-approved
patterns give teams more autonomy, not less. Most decisions remain at
Team Discretion (R-1 Tier 1); the IAB's role becomes exceptional.
The goal of this document is to make architecture governance fast for
standard cases and deliberate only for novel ones.

**Arguments examined and rejected:**

*"AI governance is speculative"* — NIST AI Agent Standards Initiative
(February 2026), ISO 42001, and OWASP LLM Top 10 are published,
citable standards. A2A's experimental status is the calibrated risk
mitigation; governance guidance with that marker is better than a
governance vacuum.

*"Coupling taxonomy is academic"* — GraphQL N+1, event schema leakage,
gRPC proxy incompatibility, and webhook topology failures all trace
directly to coupling type mismatches discovered late. The taxonomy moves
this decision to design time.

---

## Version History

| Version | Date       | Change |
| ------- | ---------- | ------ |
| 1.0.0   | 2026-02-17 | Initial Scaffold |
| 1.1.0   | 2026-04-12 | Full content: nine-style catalogue, coupling taxonomy, selection framework, governance rules R-1–R-7, AI/agent governance, cross-cutting requirements, multi-order validation |
