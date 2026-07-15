# DESIGN & ARCHITECTURE
## Exam-Style Study Notes

**Roadmap Section:** Design & Architecture → Design & Architecture Decisions, Decision Making, Requirements Elicitation

---

## 1. DESIGN & ARCHITECTURE DECISIONS

### WHY? (Problem Statement)

**Every system has an architecture** — the only question is whether it's **intentional** or **accidental**.

- **Accidental architecture** emerges from thousands of uncoordinated micro-decisions
- **Intentional architecture** results from explicit, documented, reviewed decisions
- **Architecture decisions are the most expensive to change** — they affect structure, data, contracts, teams

> **Rule:** If a decision requires coordination across teams, migrates data, or breaks contracts — it's an architecture decision. Document it.

---

### HOW? (Architecture Decision Records — ADRs)

#### ADR Template (MADR / Nygard format)

```markdown
# ADR-0042: Use Event-Driven Architecture for Order Fulfillment

## Status
Accepted (2025-01-15)

## Context
Order fulfillment involves Payment, Inventory, Shipping, Notification services.
Current synchronous REST calls create:
- Temporal coupling (all services must be up)
- Latency accumulation (500ms + 300ms + 200ms + 100ms)
- Cascade failures (Shipping down → Order stuck)
- No audit trail of fulfillment steps

**Architecturally Significant Requirements (ASRs):**
- Availability: 99.95% (fulfillment must survive partial outages)
- Latency: Order submission < 200ms (async acceptable for fulfillment)
- Auditability: Full trace of every fulfillment step for compliance
- Scalability: Handle 10x Black Friday load

## Decision
Adopt **event-driven choreography** for order fulfillment:

1. Order Service emits `OrderCreated` event
2. Payment Service consumes → emits `PaymentReserved` / `PaymentFailed`
3. Inventory Service consumes → emits `StockReserved` / `StockUnavailable`
4. Shipping Service consumes → emits `ShipmentScheduled`
5. Notification Service consumes all → sends customer updates
6. Order Service consumes fulfillment events → updates order status

**Technology:** Kafka (ordered, durable, replayable), Avro schemas in Schema Registry

## Consequences

### Positive
- ✅ Services decoupled temporally — can deploy independently
- ✅ Failure isolation — one service down doesn't block others
- ✅ Natural audit trail — event log = compliance record
- ✅ Scalability — consumers scale independently, replay for new consumers
- ✅ Latency — Order submission returns in ~50ms (just event publish)

### Negative
- ❌ **Eventual consistency** — order status not immediately final
- ❌ **Complexity** — distributed tracing, idempotency, schema evolution required
- ❌ **Debugging harder** — no single call stack, need correlation IDs
- ❌ **Operational burden** — Kafka cluster, schema registry, monitoring
- ❌ **Testing** — integration tests need full event pipeline

### Risks & Mitigations
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Event loss | Low | High | Kafka acks=all, min.insync.replicas=2, outbox pattern |
| Duplicate processing | Medium | Medium | Idempotent consumers (dedupe by event ID) |
| Schema incompatibility | Medium | High | Schema Registry (BACKWARD compatibility), contract tests |
| Ordering violations | Low | High | Single partition per order ID, exactly-once semantics |

## Alternatives Considered

### 1. Synchronous Orchestration (Orchestrator Service)
- **Pros:** Simple mental model, immediate consistency, easier debugging
- **Cons:** Centralized coupling, orchestrator = SPOF, scaling bottleneck
- **Verdict:** Rejected — violates availability ASR

### 2. Saga with Orchestrator (State Machine)
- **Pros:** Explicit flow control, compensation handling, visibility
- **Cons:** Orchestrator becomes domain-aware, coupling
- **Verdict:** Consider for complex compensation (future ADR)

### 3. CDC (Change Data Capture) from Order DB
- **Pros:** No code changes to Order Service, captures all changes
- **Cons:** Leaky abstraction (DB schema = contract), no business semantics
- **Verdict:** Rejected — violates encapsulation

## Implementation Notes
- Outbox pattern in Order Service (DB transaction + event in same tx)
- Consumer idempotency keys: `orderId:eventType:eventId`
- Correlation ID propagated via Kafka headers (W3C TraceContext)
- Dead letter topics for poison messages + alerting
- Consumer lag monitoring (Prometheus + Alertmanager)

## Related ADRs
- ADR-0012: Kafka as Event Backbone
- ADR-0018: Avro + Schema Registry for Contracts
- ADR-0023: Outbox Pattern for Reliable Events
- ADR-0031: Correlation ID Standard
```

---

#### ADR Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ADR LIFECYCLE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   PROPOSED ──► IN REVIEW ──► ACCEPTED ──► SUPERSEDED ──► DEPRECATED        │
│      │            │            │            │              │               │
│      │            │            │            │              │               │
│      ▼            ▼            ▼            ▼              ▼               │
│  Author      Architecture  Architecture  New ADR      No longer           │
│  drafts      Board reviews Board         references   recommended,        │
│  with        with team,   approves,     this ADR,    kept for            │
│  context,    stakeholders, updates       with link   historical          │
│  alternatives  revises      tech radar    to new     context             │
│                                                                             │
│   STATE TRANSITIONS:                                                        │
│   ─────────────────                                                         │
│   • PROPOSED → IN REVIEW: Author submits for review                        │
│   • IN REVIEW → ACCEPTED: Architecture Board approves                      │
│   • IN REVIEW → REJECTED: Board rejects (stays in log with reason)         │
│   • ACCEPTED → SUPERSEDED: New ADR replaces it (explicit link)             │
│   • ACCEPTED → DEPRECATED: No longer valid, no replacement yet             │
│   • SUPERSEDED/DEPRECATED → (terminal)                                     │
│                                                                             │
│   NEVER DELETE ADRs — they are the institutional memory                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### ADR Best Practices

| Practice | Why |
|----------|-----|
| **One ADR per significant decision** | Granular, linkable, reviewable |
| **Write in present tense** | "We use..." not "We will use..." |
| **Include alternatives considered** | Shows due diligence, prevents re-litigation |
| **Quantify ASRs** | "Low latency" → "p99 < 200ms at 10K RPS" |
| **Link related ADRs** | Builds decision graph, shows dependencies |
| **Store in Git (markdown)** | Versioned, reviewable via PR, near code |
| **Number sequentially** | ADR-0001, ADR-0002 — sortable, referenceable |
| **Review quarterly** | Catch drift, supersede outdated decisions |

---

### WHAT? (Decision Categories & Templates)

#### Architecture Decision Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE DECISION CATEGORIES                         │
├──────────────────────┬──────────────────────────────────────────────────────┤
│ CATEGORY             │ EXAMPLES                                             │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ STRUCTURAL           │ Monolith vs Microservices, Module boundaries,        │
│ (System Organization)│ Layering strategy, Plugin architecture               │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ COMMUNICATION        │ Sync vs Async, REST vs gRPC vs GraphQL,              │
│ (Integration Style)  │ Message broker choice, Event format (Avro/Protobuf)  │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ DATA MANAGEMENT      │ Database per service vs shared, SQL vs NoSQL,        │
│ (Persistence)        │ Sharding strategy, Event sourcing, CQRS, CDC         │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ INFRASTRUCTURE       │ Cloud provider, Kubernetes vs Serverless,            │
│ (Platform)           │ Service mesh, API Gateway, Observability stack       │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ SECURITY & COMPLIANCE│ Auth model (OAuth2/OIDC), mTLS, Secrets management,  │
│                      │ Data residency, Encryption standards                 │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ OPERATIONAL          │ Deployment strategy (Blue/Green, Canary),            │
│ (Runtime)            │ Circuit breaker config, Retry policies, SLOs         │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ TECHNOLOGY STANDARDS │ Language/runtime versions, Framework choices,        │
│ (Conventions)        │ Library approvals, Code style, Testing standards     │
└──────────────────────┴──────────────────────────────────────────────────────┘
```

---

## 2. DECISION MAKING

### WHY? (Problem Statement)

**Architecture decisions are made under uncertainty** with incomplete information, competing stakeholders, and irreversible consequences. Bad process leads to:
- **Analysis paralysis** — too many options, no decision
- **HiPPO decisions** — Highest Paid Person's Opinion wins
- **Consensus theater** — everyone agrees but no one owns it
- **Decision amnesia** — six months later, no one remembers why

---

### HOW? (Decision-Making Frameworks)

#### 1. RAPID Decision Model (Bain & Company)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            RAPID ROLES                                      │
├─────────┬───────────────────────────────────────────────────────────────────┤
│ R       │ RECOMMEND — Does the analysis, proposes options, writes ADR      │
│         │ (Architect / Tech Lead)                                          │
├─────────┼───────────────────────────────────────────────────────────────────┤
│ A       │ AGREE — Must concur; can veto. Usually Security, Legal,          │
│         │ Compliance, Platform. If they disagree, escalate.                │
├─────────┼───────────────────────────────────────────────────────────────────┤
│ P       │ PERFORM — Implements the decision. Development teams.            │
├─────────┼───────────────────────────────────────────────────────────────────┤
│ I       │ INPUT — Provides data, constraints, perspectives.                │
│         │ Product, Ops, Data, UX, other architects. No veto power.         │
├─────────┼───────────────────────────────────────────────────────────────────┤
│ D       │ DECIDE — Single person accountable for final call.               │
│         │ CTO / Chief Architect / Architecture Board Chair.                │
│         │ "The buck stops here."                                           │
└─────────┴───────────────────────────────────────────────────────────────────┘

KEY PRINCIPLE: Only ONE "D". Multiple "A"s possible. Many "I"s.
```

#### 2. Architecture Trade-off Analysis Matrix (ATAM-inspired)

```
DECISION: Choose Event Broker for Order Fulfillment

┌─────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ CRITERION       │ WEIGHT   │ KAFKA    │ RABBITMQ │ AWS      │ AZURE    │
│                 │ (1-5)    │          │          │ EVENT-   │ SERVICE  │
│                 │          │          │          │ BRIDGE   │ BUS      │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Throughput      │ 5        │ 5 ★      │ 3        │ 4        │ 3        │
│ Latency         │ 4        │ 4        │ 5 ★      │ 3        │ 3        │
│ Durability      │ 5        │ 5 ★      │ 4        │ 4        │ 4        │
│ Ordering        │ 4        │ 5 ★      │ 2        │ 3        │ 3        │
│ Replayability   │ 5        │ 5 ★      │ 2        │ 3        │ 3        │
│ Operational     │ 3        │ 2        │ 4        │ 5 ★      │ 5 ★      │
│ Complexity      │          │          │          │          │          │
│ Cloud Native    │ 4        │ 4        │ 3        │ 5 ★      │ 5 ★      │
│ Cost            │ 3        │ 3        │ 4        │ 2        │ 2        │
│ Team Expertise  │ 4        │ 4        │ 3        │ 2        │ 2        │
│ Vendor Lock-in  │ 2        │ 5 ★      │ 5 ★      │ 1        │ 1        │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ WEIGHTED SCORE  │          │ 4.3 ★    │ 3.3      │ 3.4      │ 3.2      │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

★ = Selected (Kafka) — highest weighted score
```

#### 3. Decision Quality Checklist (Before Deciding)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DECISION QUALITY CHECKLIST                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FRAMING                                                                    │
│  ☐ Is the decision statement clear and scoped?                             │
│  ☐ Are the architecturally significant requirements (ASRs) explicit?       │
│  ☐ Is the decision owner (D) identified?                                   │
│                                                                             │
│  ALTERNATIVES                                                               │
│  ☐ At least 3 viable alternatives generated?                               │
│  ☐ Status quo included as an alternative?                                  │
│  ☐ "Do nothing" / "Defer" considered?                                      │
│                                                                             │
│  INFORMATION                                                                │
│  ☐ Key uncertainties identified?                                           │
│  ☐ Spikes/POCs planned for high-uncertainty areas?                         │
│  ☐ Stakeholder input (I) gathered?                                         │
│                                                                             │
│  REASONING                                                                  │
│  ☐ Trade-offs explicit for each alternative?                               │
│  ☐ Criteria weighted by importance (not all equal)?                        │
│  ☐ Biases checked (confirmation, anchoring, sunk cost)?                    │
│                                                                             │
│  COMMITMENT                                                                 │
│  ☐ Agree (A) parties consulted and documented?                             │
│  ☐ Perform (P) teams aware and resourced?                                  │
│  ☐ Decision (D) recorded with rationale?                                   │
│                                                                             │
│  FOLLOW-THROUGH                                                             │
│  ☐ ADR written and merged?                                                 │
│  ☐ Fitness functions added to CI?                                          │
│  ☐ Communication plan for affected teams?                                  │
│  ☐ Review date set (quarterly)?                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### Cognitive Biases in Architecture Decisions

| Bias | Symptom | Countermeasure |
|------|---------|----------------|
| **Confirmation Bias** | Only seeking data that supports preferred option | Assign "Red Team" to challenge |
| **Anchoring** | First option becomes reference point | Generate alternatives *before* evaluating |
| **Sunk Cost** | "We've invested so much in X" | Ignore past investment; evaluate forward |
| **Availability Heuristic** | Recent vivid example drives choice | Use structured criteria, not anecdotes |
| **Groupthink** | No dissent in review | Require written objections; anonymous input |
| **Not Invented Here** | Rejecting external solutions | Explicit "build vs buy" criteria |
| **Resume-Driven Development** | Choosing tech for career not need | Require business justification in ADR |

---

### WHAT? (Decision-Making Artifacts)

#### Decision Log (Lightweight ADR Index)

| ID | Title | Status | Date | Owner | Supersedes |
|----|-------|--------|------|-------|------------|
| ADR-0001 | Use .NET 8 for all new services | Accepted | 2024-01 | C. Architect | — |
| ADR-0002 | Kafka for event backbone | Accepted | 2024-01 | C. Architect | — |
| ADR-0003 | PostgreSQL as default relational DB | Accepted | 2024-02 | D. Architect | — |
| ADR-0004 | Modular monolith for Order domain | Accepted | 2024-03 | T. Lead | — |
| ADR-0005 | Extract Payment Service | Superseded | 2024-06 | T. Lead | ADR-0004 |
| ADR-0006 | Event-driven fulfillment (choreography) | Accepted | 2025-01 | C. Architect | — |

---

## 3. REQUIREMENTS ELICITATION

### WHY? (Problem Statement)

**"The hardest single part of building a software system is deciding precisely what to build."** — Fred Brooks

Most architecture failures trace to **requirements failures**:
- Building the wrong thing (solving wrong problem)
- Missing critical NFRs (performance, security, compliance)
- Ambiguous requirements → different interpretations
- Stakeholder misalignment → scope creep, rework

---

### HOW? (Elicitation Techniques)

#### 1. Stakeholder Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STAKEHOLDER MAP                                     │
├──────────────────────┬──────────────────────────────────────────────────────┤
│ PRIMARY (Decide)     │ • Product Owner / GM                                 │
│                      │ • Business Sponsor (funds it)                        │
│                      │ • Compliance / Legal (constraints)                   │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ SECONDARY (Influence)│ • End Users (actual workflow)                        │
│                      │ • Operations / SRE (run it)                          │
│                      │ • Security (threat model)                            │
│                      │ • Data / Analytics (consume it)                      │
│                      │ • Support (debug it)                                 │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ TERTIARY (Inform)    │ • Sales / Marketing (sell it)                        │
│                      │ • Finance (cost model)                               │
│                      │ • Executives (strategic alignment)                   │
│                      │ • Partners / Vendors (integrate)                     │
└──────────────────────┴──────────────────────────────────────────────────────┘
```

#### 2. Requirements Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REQUIREMENTS TAXONOMY                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FUNCTIONAL REQUIREMENTS (What the system DOES)                            │
│  ─────────────────────────────────────────────────                         │
│  • User stories / Use cases                                                │
│  • Business rules & calculations                                           │
│  • Data entry, validation, transformation                                  │
│  • Workflows & state transitions                                           │
│  • Integrations (APIs, file feeds, events)                                 │
│  • Reporting & analytics                                                   │
│                                                                             │
│  NON-FUNCTIONAL REQUIREMENTS (How the system BEHAVES) — QUALITY ATTRIBUTES │
│  ───────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  ┌─────────────┬──────────────────┬────────────────────────────────────┐  │
│  │ CATEGORY    │ KEY METRICS      │ ARCHITECTURE IMPLICATIONS          │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ PERFORMANCE │ Latency (p50/95/99)│ Caching, CDN, read replicas,     │  │
│  │             │ Throughput (RPS)   │ async, partitioning, indexing    │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ SCALABILITY │ Horizontal scale   │ Stateless, sharding, queue       │  │
│  │             │ limits, elasticity │ buffering, auto-scaling          │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ AVAILABILITY│ Uptime %, RPO/RTO  │ Multi-AZ, replication, failover, │  │
│  │             │ MTBF/MTTR          │ circuit breaker, graceful deg.   │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ CONSISTENCY │ Strong/Eventual/   │ Consensus, quorum, saga,         │  │
│  │             │ Causal             │ event sourcing, CRDTs            │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ SECURITY    │ AuthZ/AuthN,       │ Zero trust, mTLS, secrets,       │  │
│  │             │ Encryption, Audit  │ WAF, DDoS protection, pen test   │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ MAINTAIN-   │ Deploy freq,       │ Modularity, tests, observability,│  │
│  │ ABILITY     │ Lead time, MTTC    │ feature flags, docs              │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ OPERATIONAL │ Observability,     │ Logs, metrics, traces, alerts,   │  │
│  │ EXCELLENCE  │ Debuggability      │ runbooks, chaos engineering      │  │
│  ├─────────────┼──────────────────┼────────────────────────────────────┤  │
│  │ COMPLIANCE  │ GDPR, HIPAA,       │ Data residency, retention,       │  │
│  │             │ PCI-DSS, SOX       │ audit trails, encryption         │  │
│  └─────────────┴──────────────────┴────────────────────────────────────┘  │
│                                                                             │
│  CONSTRAINTS (Hard boundaries)                                             │
│  ────────────────────────────                                              │
│  • Budget, Timeline, Team size/skills                                      │
│  • Technology standards (approved languages, clouds, vendors)              │
│  • Regulatory (data residency, certifications)                             │
│  • Legacy system integration (can't change)                                │
│  • Organizational (team topology, approval gates)                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### 3. Elicitation Techniques by Phase

| Phase | Techniques | Output |
|-------|------------|--------|
| **DISCOVERY** | Stakeholder interviews, Contextual inquiry, Job shadowing, Survey | Problem statement, Stakeholder map, High-level goals |
| **EXPLORATION** | Workshop (Event Storming, Domain Storytelling), User journey mapping, Process mining | Domain model, Business capabilities, Pain points |
| **SPECIFICATION** | User story mapping, Example mapping (BDD), Acceptance criteria, Decision tables | Prioritized backlog, NFR specifications, Constraints |
| **VALIDATION** | Prototype/Spike, Architecture review, Threat modeling, Cost estimation | Feasibility confirmation, Risk register, ADRs |

---

#### 4. Event Storming (Domain-Driven Design Workshop)

```
EVENT STORMING PROCESS:
────────────────────────

1. CHAOTIC EXPLORATION (30-60 min)
   ┌────────────────────────────────────────────────────────────┐
   │  Participants write DOMAIN EVENTS (past tense) on ORANGE   │
   │  stickies: "Order Placed", "Payment Authorized",           │
   │  "Inventory Reserved", "Shipment Created"                  │
   │  Place on timeline. No discussion yet — just dump.         │
   └────────────────────────────────────────────────────────────┘
                    │
                                    │
                                    ▼
2. ENFORCE TIMELINE (15 min)
   ┌────────────────────────────────────────────────────────────┐
   │  Order events chronologically. Identify:                   │
   │  • Gaps (missing events)                                   │
   │  • Parallel tracks (swimlanes)                             │
   │  • Pivotal events (major state changes)                    │
   └────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
3. ADD COMMANDS (BLUE) & ACTORS (YELLOW) (20 min)
   ┌────────────────────────────────────────────────────────────┐
   │  For each event: What TRIGGERED it? → COMMAND              │
   │  Who INITIATED it? → ACTOR (User, System, Timer)           │
   │  Commands: "Place Order", "Authorize Payment"              │
   └────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
4. IDENTIFY AGGREGATES (YELLOW) & BOUNDED CONTEXTS (20 min)
   ┌────────────────────────────────────────────────────────────┐
   │  Group commands/events that belong together → AGGREGATE    │
   │  Order, Payment, Inventory, Shipping                       │
   │  Draw boundaries → BOUNDED CONTEXTS (candidate services)   │
   └────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
5. ADD POLICIES (LILAC) & READ MODELS (GREEN) (15 min)
   ┌────────────────────────────────────────────────────────────┐
   │  Policies: "When Order Placed → Reserve Payment"           │
   │  Read Models: "Order Summary", "Customer Dashboard"        │
   │  External systems (PINK): Payment Gateway, ERP, Email      │
   └────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
6. HOT SPOTS & QUESTIONS (RED) (10 min)
   ┌────────────────────────────────────────────────────────────┐
   │  Mark ambiguities, conflicts, technical questions          │
   │  "What if payment fails after inventory reserved?"         │
   │  These become spikes, ADRs, or backlog items               │
   └────────────────────────────────────────────────────────────┘

OUTPUT: Shared domain understanding, Service boundaries, Event catalog, Risk list
```

---

### WHAT? (Requirements Artifacts)

#### 1. Architecture Significant Requirements (ASR) Document

```markdown
# ASR-001: Order Management System

## Functional Requirements
| ID | Requirement | Priority | Source |
|----|-------------|----------|--------|
| FR-001 | Customer places order with multiple items | Must | PO |
| FR-002 | Support guest checkout (no account) | Must | PO |
| FR-003 | Apply promo codes, loyalty points | Should | Marketing |
| FR-004 | Split shipment across warehouses | Could | Ops |
| FR-005 | Order modification within 15 min | Must | Support |

## Non-Functional Requirements (Quantified)
| ID | Quality Attribute | Metric | Target | Rationale |
|----|-------------------|--------|--------|-----------|
| NFR-01 | Latency (Order Submit) | p99 | < 200ms | Conversion impact |
| NFR-02 | Latency (Order History) | p95 | < 500ms | UX |
| NFR-03 | Throughput | Peak RPS | 5,000 | Black Friday |
| NFR-04 | Availability | Uptime | 99.95% | Revenue loss/min |
| NFR-05 | Data Durability | RPO | 0 (sync replica) | Financial data |
| NFR-06 | Disaster Recovery | RTO | < 1 hour | Business continuity |
| NFR-07 | Consistency | Order State | Strong | Financial accuracy |
| NFR-08 | Consistency | Inventory | Eventual (<5s) | Performance |
| NFR-09 | Security | PCI-DSS | Level 1 | Credit cards |
| NFR-10 | Security | PII | GDPR Art. 25 | EU customers |
| NFR-11 | Auditability | Order Changes | 100% traceable | Compliance |
| NFR-12 | Scalability | Horizontal | 10x peak | Growth headroom |

## Constraints
| ID | Constraint | Type | Impact |
|----|------------|------|--------|
| CON-01 | Deploy to Azure (corporate standard) | Platform | Limits cloud services |
| CON-02 | .NET 8 / C# (team skills) | Technology | Language/framework fixed |
| CON-03 | PostgreSQL (approved DB) | Technology | No SQL Server, no MongoDB |
| CON-04 | 6-month deadline | Schedule | Phased delivery required |
| CON-05 | Team: 2 squads (8 devs each) | Org | Limits parallel workstreams |
| CON-06 | Integrate with legacy ERP (SOAP) | Integration | Adapter pattern required |

## Assumptions
| ID | Assumption | Validation Plan |
|----|------------|-----------------|
| ASM-01 | Payment gateway supports async webhooks | Spike by Sprint 2 |
| ASM-02 | Inventory service exposes reservation API | Confirm with Inventory team |
| ASM-03 | Customer data in CRM (single source) | Verify with Data team |
```

---

#### 2. Quality Attribute Workshop (QAW) Template

```
QUALITY ATTRIBUTE WORKSHOP — ORDER MANAGEMENT

SCENARIO 1: BLACK FRIDAY TRAFFIC SPIKE
────────────────────────────────────────
Stimulus: 10x normal traffic (50K → 500K RPS) for 4 hours
Environment: Normal operation
Response: System processes orders without data loss
Response Measure: 
  - 99.9% orders accepted
  - p99 latency < 500ms (degraded from 200ms)
  - Zero data loss (durability)
  - Auto-scale within 3 minutes

ARCHITECTURAL TACTICS:
☐ Load balancing (L7, least connections)
☐ Horizontal pod autoscaler (CPU + custom queue metric)
☐ Queue-based load leveling (Kafka buffer)
☐ Read replicas for queries (CQRS)
☐ Circuit breaker on downstream (Payment, Inventory)
☐ Graceful degradation: disable non-critical (recommendations, analytics)
☐ Rate limiting at API Gateway (per customer, per IP)

SCENARIO 2: PAYMENT GATEWAY OUTAGE
────────────────────────────────────────
Stimulus: Payment provider returns 503 for 30 minutes
Environment: Peak hours
Response: Orders queued, customers notified, no data loss
Response Measure:
  - 100% orders persisted (outbox pattern)
  - Customer sees "Processing" not "Failed"
  - Automatic retry when gateway recovers
  - Manual intervention < 1% of orders

ARCHITECTURAL TACTICS:
☐ Async command processing (Order → Kafka → Payment Worker)
☐ Outbox pattern (DB + Event in same transaction)
☐ Idempotent payment requests (dedupe key)
☐ Exponential backoff retry (1m, 5m, 15m, 1h, 6h)
☐ Dead letter queue + alerting after max retries
☐ Customer notification via event (OrderPaymentPending)

SCENARIO 3: DATA BREACH ATTEMPT
────────────────────────────────────────
Stimulus: Attacker attempts SQL injection via order notes field
Environment: Production
Response: Attack blocked, logged, alerted, no data exposed
Response Measure:
  - 0 successful injections
  - Alert to SOC within 1 minute
  - WAF blocks IP automatically
  - Audit trail complete

ARCHITECTURAL TACTICS:
☐ Parameterized queries (EF Core / Dapper)
☐ Input validation (FluentValidation, allowlist)
☐ WAF rules (OWASP Core Rule Set)
☐ Structured logging with correlation ID
☐ SIEM integration (Splunk/Sentinel)
☐ Penetration test quarterly
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Context | Anti-Pattern | Why Avoid |
|---------|---------|--------------|-----------|
| **ADR for Every Significant Decision** | Always | Verbal decisions, Confluence pages | Decisions lost, no rationale, can't revisit |
| **Quantified NFRs (SLIs/SLOs)** | Requirements | "Fast", "Scalable", "Secure" | Unmeasurable, untestable, unimplementable |
| **Event Storming for Domain Discovery** | Greenfield / Re-architecture | Solo architect designs in isolation | Missed domain complexity, wrong boundaries |
| **Trade-off Matrix (Weighted Criteria)** | Technology Selection | Gut feel / Resume-driven / Vendor pitch | Biased, unexplainable, fragile |
| **RAPID Decision Model** | Org with multiple stakeholders | Consensus (everyone must agree) | Slow, watered-down, no ownership |
| **Spike/POC for High Uncertainty** | New Tech / Complex Integration | Assume it works / Big design up front | Late surprises, sunk cost |
| **Requirements as Executable Specs (BDD)** | Complex Business Logic | Word docs / Jira tickets only | Ambiguity, misalignment, late defects |
| **Architecture Decision Backlog** | Ongoing Governance | No tracking | Decisions made ad-hoc, inconsistent |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. ADRs: "Why use ADRs? What's in a good one?"

**Answer:**
```
ADRs capture *context*, *decision*, *consequences*, *alternatives* — not just the choice.

GOOD ADR INCLUDES:
├── Title + Number + Date + Status
├── Context (what forces drove this?)
├── Decision (what are we doing?)
├── Consequences (positive, negative, risks)
├── Alternatives Considered (at least 2, with why rejected)
├── Related ADRs (dependencies, supersedes)
└── Implementation Notes (fitness functions, migration)

BAD ADR: "We chose Kafka." (No context, no trade-offs, no alternatives)
GOOD ADR: "ADR-0012: Kafka for event backbone — chosen for throughput, 
         durability, replay; accepted operational complexity; 
         rejected RabbitMQ (no replay), Event Bridge (vendor lock-in)."
```

---

### 2. Decision Making: "How do you choose between Kafka and RabbitMQ?"

**Answer:**
```
USE STRUCTURED TRADE-OFF ANALYSIS, NOT GUT FEEL:

1. DEFINE CRITERIA FROM ASRs:
   - Throughput (50K+ events/sec) → Weight: 5
   - Replayability (audit, new consumers) → Weight: 5
   - Ordering per entity (orderId) → Weight: 4
   - Operational simplicity → Weight: 3
   - Team expertise → Weight: 4
   - Cost → Weight: 2

2. SCORE EACH OPTION (1-5):
   | Criterion      | Weight | Kafka | RabbitMQ |
   |----------------|--------|-------|----------|
   | Throughput     | 5      | 5     | 3        |
   | Replayability  | 5      | 5     | 2        |
   | Ordering       | 4      | 5     | 2        |
   | Ops Simplicity | 3      | 2     | 4        |
   | Team Expertise | 4      | 4     | 3        |
   | Cost           | 2      | 3     | 4        |
   | WEIGHTED TOTAL |        | 4.4   | 2.7      |

3. DOCUMENT IN ADR with scores, rationale, risks.

RESULT: Kafka wins. But if throughput was 1K/sec and team knew RabbitMQ — 
        RabbitMQ might win. CONTEXT MATTERS.
```

---

### 3. Requirements: "How do you elicit NFRs from stakeholders who only talk features?"

**Answer:**
```
STAKEHOLDERS DON'T SPEAK NFR — TRANSLATE:

TECHNIQUE 1: SCENARIO-BASED QUESTIONS
"Walk me through Black Friday. What happens if the site slows down?"
→ "We lose $X/min" → Availability & Latency targets

TECHNIQUE 2: FEAR QUESTIONS
"What keeps you up at night about this system?"
→ "Data breach" → Security & Compliance
→ "Wrong prices charged" → Consistency & Audit

TECHNIQUE 3: CONSTRAINT QUESTIONS
"What CAN'T we change?" (Regulations, contracts, legacy)
→ Hard constraints for architecture

TECHNIQUE 4: QUANTIFY FEATURES
"Real-time dashboard" → "What's 'real-time'? 1s? 100ms? Who sees it?"
→ Latency, Consistency, Throughput targets

TECHNIQUE 5: SLO WORKSHOP
Collaborative session: "For each user journey, what's acceptable?"
Output: SLI/SLO table signed by Product + Engineering

TEMPLATE:
| User Journey | SLI | SLO | Consequence of Miss |
|--------------|-----|-----|---------------------|
| Place Order  | p99 latency | <200ms | Cart abandonment ↑ |
| Order History| p95 latency | <500ms | Support tickets ↑ |
| Payment      | Success rate | 99.95% | Revenue loss, trust |
```

---

### 4. Architecture Decisions: "How do you handle disagreement on a major architectural decision?"

**Answer:**
```
RAPID MODEL — CLEAR ROLES PREVENT STALEMATES:

1. SINGLE DECIDER (D) — CTO / Chief Architect
   Accountable for outcome. Makes final call.

2. RECOMMENDER (R) — Lead Architect
   Does analysis, writes ADR, presents options.

3. AGREERS (A) — Security, Legal, Platform, Data
   Must concur. Can veto (with written reason).
   If veto → escalate to D.

4. INPUT (I) — Product, UX, Ops, other teams
   Provide data, perspectives. NO veto.

5. PERFORMERS (P) — Dev Teams
   Implement. Know decision early for planning.

PROCESS:
1. R writes draft ADR with alternatives
2. Circulate to A + I for 1-week async review
3. Sync meeting for objections (time-boxed 60 min)
4. R incorporates feedback, updates ADR
5. D decides: Accept / Reject / Defer (with reason)
6. ADR merged, communicated, fitness functions added

KEY: Disagreement is EXPECTED and DOCUMENTED in ADR.
     "Security objected due to X; D accepted risk because Y."
```

---

### 5. Event Storming: "When would you use Event Storming vs traditional requirements gathering?"

**Answer:**
```
EVENT STORMING WHEN:
✓ Complex domain with multiple experts (no single person knows it all)
✓ Domain boundaries unclear (bounded context discovery)
✓ Cross-functional alignment needed (dev + product + ops)
✓ Legacy system modernization (reverse engineer domain)
✓ New product exploration (discover unknown unknowns)

TRADITIONAL (Interviews, User Stories) WHEN:
✓ Well-understood domain (CRUD, standard patterns)
✓ Simple scope (single team, few stakeholders)
✓ Regulatory requirements (need traceable specs)
✓ Time-boxed discovery (Event Storming = 2-4 hours minimum)

HYBRID APPROACH (RECOMMENDED):
1. Event Storming (2-4 hrs) → Domain model, boundaries, events
2. User Story Mapping (1-2 hrs) → Prioritized backlog, MVP slice
3. Example Mapping (per story) → Acceptance criteria, edge cases
4. Architecture Decision Records → Document structural choices
```

---

### 6. NFRs: "How do you make NFRs testable and enforceable?"

**Answer:**
```
CONVERT NFRs TO SLIs/SLOs + FITNESS FUNCTIONS:

NFR: "System must be highly available"
    ↓
SLI: Availability = (Successful Requests / Total Requests) * 100
SLO: Availability ≥ 99.95% over 30-day rolling window
    ↓
FITNESS FUNCTION (CI/CD):
- Synthetic monitoring (every 30s) → Alert if SLO breach predicted
- Chaos experiment (weekly): Kill pod, verify recovery < 30s
- Deployment gate: Error budget burn rate < 2x

NFR: "Low latency"
    ↓
SLI: p99 latency of PlaceOrder endpoint
SLO: p99 < 200ms at 5K RPS
    ↓
FITNESS FUNCTION:
- Load test in CI (k6/Gatling) — fail build if p99 > 200ms
- Production: Continuous profiling (Pyroscope) — alert on regression
- Canary deploy: Compare latency vs baseline — auto-rollback if +10%

NFR: "Secure"
    ↓
SLI: % Critical vulnerabilities in dependencies
SLO: 0 critical CVEs > 7 days old
    ↓
FITNESS FUNCTION:
- SCA scan (Trivy/Snyk) on every PR — fail on critical
- SAST in pipeline — fail on high severity
- Container scan — fail on critical
- Pen test report — zero critical findings
```

---

### 7. Stakeholders: "How do you handle conflicting requirements from different stakeholders?"

**Answer:**
```
CONFLICT RESOLUTION FRAMEWORK:

1. MAKE CONFLICTS EXPLICIT
   Create trade-off matrix:
   | Stakeholder | Requirement | Conflicts With | Impact if Met | Impact if Not |
   |-------------|-------------|----------------|---------------|---------------|
   | Security    | Encrypt all | Performance    | +Security     | -Latency 20%  |
   | Product     | Sub-100ms   | Encryption     | +Conversion   | -Compliance   |

2. QUANTIFY IMPACT
   - A/B test, spike, model, estimate
   - "Encryption adds 15ms p99 → 2% conversion drop = $X/month"

3. FIND CREATIVE OPTIONS (not just A vs B)
   - Field-level encryption (only PII)
   - Hardware acceleration (AES-NI)
   - Async encryption (fire-and-forget for audit logs)
   - Edge encryption (Cloudflare Workers)

4. DECIDE WITH RAPID
   - R (Architect) presents options with data
   - A (Security, Product) agree or veto
   - D (CTO) decides based on business priority

5. DOCUMENT IN ADR
   "Chose field-level encryption (Option C). 
   Security agreed — PII protected. 
   Product agreed — latency impact <5ms. 
   D accepted — compliance met, revenue protected."

PRINCIPLE: Conflicts resolved by DATA + DECISION RIGHTS, not hierarchy or loudness.
```

---

### 8. Requirements: "What's the difference between functional and non-functional requirements in architecture?"

**Answer:**
```
FUNCTIONAL = WHAT the system DOES
├── User Stories: "As a customer, I want to place an order"
├── Use Cases: Place Order, Cancel Order, View History
├── Business Rules: "Orders > $100 get free shipping"
├── Data Transformations: CSV import → normalized DB
├── Integrations: POST /orders → ERP SOAP endpoint
└── Reports: Daily sales by region

NON-FUNCTIONAL = HOW the system BEHAVES (Quality Attributes)
├── Performance: p99 < 200ms, 10K RPS
├── Scalability: Horizontal to 100 pods, auto-scale < 3min
├── Availability: 99.95%, RPO=0, RTO<1hr
├── Consistency: Strong for orders, Eventual for analytics
├── Security: OAuth2, mTLS, PCI-DSS Level 1
├── Maintainability: Deploy daily, <1hr lead time, 80% coverage
├── Operability: Logs/metrics/traces, <5min MTTR
├── Compliance: GDPR, SOX, Data residency EU
└── Cost: <$0.01/order at peak

ARCHITECTURE IMPACT:
- Functional → Domain model, API contracts, Data schema
- Non-Functional → Infrastructure, Patterns, Technology choices, Topology

EXAMPLE:
Functional: "Customer views order history"
Non-Functional: "p95 < 500ms, available 99.9%, last 2 years data"
Architecture: CQRS read model (denormalized), Redis cache, 
              Read replicas, Archive cold data to blob storage
```

---

### 9. Decision Making: "How do you avoid 'Analysis Paralysis' in architecture?"

**Answer:**
```
ANALYSIS PARALYSIS = Infinite options, no decision, no progress.

PREVENTION STRATEGIES:

1. TIME-BOX DECISIONS
   "We decide by Friday. Best option with current info wins."
   Defer reversible decisions. Make irreversible ones deliberately.

2. REVERSIBLE VS IRREVERSIBLE (Jeff Bezos)
   ┌─────────────────────┬────────────────────────────────────┐
   │ TYPE 1: IRREVERSIBLE│ TYPE 2: REVERSIBLE                 │
   │ (One-way doors)     │ (Two-way doors)                    │
   ├─────────────────────┼────────────────────────────────────┤
   │ • Database choice   │ • Library version                  │
   │ • Cloud provider    │ • Cache TTL                        │
   │ • Event format      │ • Feature flag default             │
   │ • Service boundaries│ • Circuit breaker threshold        │
   │ • Auth model        │ • Retry policy                     │
   │                     │                                    │
   │ DECIDE CAREFULLY    │ DECIDE QUICKLY, ITERATE            │
   │ Full ADR, spikes,   │ Default to action, measure, adjust │
   │ Board review        │                                    │
   └─────────────────────┴────────────────────────────────────┘

3. SATISFICING (Herbert Simon)
   "Good enough" > "Optimal" when cost of delay > cost of suboptimal.
   Set minimum criteria. Pick first option that meets them.

4. SPIKE & DECIDE
   Time-boxed experiment (2-3 days max):
   - Build minimal prototype
   - Measure key criteria
   - Decide with data

5. ARCHITECTURE DECISION BACKLOG
   Prioritize decisions by:
   - Irreversibility (Type 1 first)
   - Dependency (what unblocks others?)
   - Risk (what fails catastrophically if wrong?)
   
   Work top-down. One decision per week cadence.

6. DEFAULT STANDARDS (Reduce Decision Surface)
   - "Default to PostgreSQL, Kafka, .NET 8, Azure, Kubernetes"
   - Only decide when default doesn't fit ASRs
   - Technology Radar: Adopt/Trial/Assess/Hold
```

---

### 10. Elicitation: "How do you validate requirements before committing to architecture?"

**Answer:**
```
VALIDATION TECHNIQUES (Progressive Fidelity):

LEVEL 1: PAPER VALIDATION (Hours)
├── Stakeholder walkthrough of user journeys
├── Event Storming → Shared understanding check
├── Architecture Decision Records review with stakeholders
└── "Pre-mortem": "It's 6 months later, project failed. Why?"

LEVEL 2: PROTOTYPE / SPIKE (Days)
├── Riskiest assumption → Code spike (2-3 days max)
├── Example: "Can Kafka handle our ordering guarantees?"
├── Measure: Throughput, latency, ordering, failure modes
├── Decision: Proceed / Pivot / Abandon

LEVEL 3: ARCHITECTURE PROTOTYPE (1-2 Sprints)
├── Walking Skeleton: End-to-end thin slice
├── Deploy to staging, run load test, chaos test
├── Validate: NFRs, deployment, observability, team velocity
├── Go/No-Go for full investment

LEVEL 4: MVP IN PRODUCTION (Weeks)
├── Release to subset of users (canary / feature flag)
├── Measure real SLIs, user feedback, operational burden
├── Iterate or pivot

VALIDATION CHECKLIST PER LEVEL:
☐ Functional correctness (happy path + 2 error paths)
☐ NFR targets met (latency, throughput, availability)
☐ Operational readiness (logs, metrics, alerts, runbooks)
☐ Team confidence (can they own it?)
☐ Cost within budget
☐ Security review passed
☐ Compliance check passed

RULE: No architecture commitment without Level 2+ validation for Type 1 decisions.
```

---

### 11. Trade-offs: "Explain a time you made a significant architectural trade-off."

**Answer:**
```
SITUATION: Order Service needed strong consistency for financial accuracy
           but high availability for Black Friday (99.99% target).

TRADE-OFF: CONSISTENCY vs AVAILABILITY (CAP / PACELC)

OPTIONS:
A. Single-leader PostgreSQL (Strong Consistency, CP)
   - Pros: ACID, simple, familiar
   - Cons: Leader = write bottleneck, failover = downtime (10-30s)
   
B. Multi-leader / Leaderless (Eventual Consistency, AP)
   - Pros: Write anywhere, high availability
   - Cons: Conflicts, lost updates, complex reconciliation

C. Single-leader + Sync Replica (Strong Consistency, HA)
   - Pros: ACID, fast failover (<5s with Patroni/Patron)
   - Cons: Cross-AZ latency (2ms), write throughput limited by sync

DECISION: Option C (PostgreSQL + Patroni + Sync Replica)

RATIONALE:
- Financial data REQUIRES strong consistency (regulatory)
- Availability target achievable with sync replica + fast failover
- Cross-AZ latency (2ms) acceptable for order write path
- Team expertise: PostgreSQL strong, distributed DB weak

MITIGATIONS FOR AVAILABILITY:
- Read replicas for queries (offload leader)
- Circuit breaker on write path → graceful degradation (queue orders)
- Multi-AZ deployment (leader in AZ1, sync replica in AZ2)
- Chaos engineering: Monthly failover drills

RESULT: 99.97% availability achieved, zero data loss incidents in 2 years.
        Write latency p99: 45ms (within 200ms budget).

ADR DOCUMENTED: ADR-0042 "PostgreSQL HA for Order Service"
```

---

### 12. Stakeholders: "How do you communicate architecture to non-technical stakeholders?"

**Answer:**
```
USE THE RIGHT ARTIFACT FOR THE AUDIENCE:

EXECUTIVES (CEO, CFO, VP Product)
├── Business Capability Map — What we're building & why
├── Roadmap — Phases, milestones, value delivery dates
├── Investment Ask — Cost, ROI, risk, alternatives
├── Risk Register — Top 5 risks, likelihood, impact, mitigation
└── One-Pager — "In 3 sentences: What, Why, When"

PRODUCT / PROGRAM MANAGEMENT
├── Context Diagram (C4 Level 1) — System + Actors
├── Sequence Diagrams — Key user journeys
├── NFR Commitments (SLOs) — Latency, Availability, Throughput
├── Dependency Map — What blocks what, critical path
└── Release Plan — Increments, features per increment

ENGINEERING LEADERSHIP (EMs, Tech Leads)
├── Container Diagram (C4 Level 2) — Services, DBs, Queues
├── ADR Log — Decisions, rationale, fitness functions
├── Technology Radar — Adopt/Trial/Assess/Hold
├── Team Topology — Ownership boundaries, cognitive load
└── Technical Debt Register — Items, severity, remediation plan

DEVELOPERS (Implementation)
├── Component Diagram (C4 Level 3) — Modules, Interfaces
├── API Contracts (OpenAPI/Protobuf) — Request/Response
├── Data Model (ER/Schema) — Tables, Indexes, Migrations
├── Coding Standards — Linting, Patterns, Libraries
├── Fitness Functions — CI gates (ArchUnit, Contract Tests)
└── Runbooks — Deploy, Rollback, Debug, Scale

SECURITY / COMPLIANCE / AUDIT
├── Data Flow Diagram — PII, Encryption, Trust Boundaries
├── Threat Model (STRIDE) — Threats, Mitigations, Residual Risk
├── Compliance Matrix — Requirement → Control → Evidence
├── Incident Response Plan — Roles, Playbooks, Contacts
└── Pen Test Reports — Findings, Remediation, Retest

RULE: Never show C4 Level 3 to executives. Never show roadmap to developers as "architecture."
      Match artifact to audience's decision-making needs.
```

---

### 13. Requirements: "How do you handle requirements that change during architecture phase?"

**Answer:**
```
REQUIREMENTS CHANGE IS INEVITABLE. DESIGN FOR CHANGE.

1. CLASSIFY CHANGE TYPE
   ┌──────────────────┬────────────────────────────────────────────┐
   │ TYPE             │ RESPONSE                                   │
   ├──────────────────┼────────────────────────────────────────────┤
   │ CLARIFICATION    │ Update spec, no architecture impact        │
   │ (Detail added)   │                                            │
   ├──────────────────┼────────────────────────────────────────────┤
   │ SCOPE CHANGE     │ Impact analysis → ADR if structural        │
   │ (New feature)    │ Update roadmap, negotiate trade-offs       │
   ├──────────────────┼────────────────────────────────────────────┤
   │ NFR CHANGE       │ Re-evaluate patterns, infra, SLAs          │
   │ (Scale target)   │ Update fitness functions                   │
   ├──────────────────┼────────────────────────────────────────────┤
   │ CONSTRAINT CHANGE│ Major — may invalidate decisions           │
   │ (New regulation) │ Architecture Review Board emergency session│
   └──────────────────┴────────────────────────────────────────────┘

2. CHANGE CONTROL PROCESS
   - All changes via ADR (even small ones if structural)
   - Architecture Review Board meets bi-weekly
   - Emergency changes: D (Decider) can approve, ratify next meeting

3. ARCHITECTURE FOR CHANGE (Principles)
   ┌─────────────────────────────────────────────────────────────────┐
   │ PRINCIPLE              │ TACTIC                                 │
   ├────────────────────────┼────────────────────────────────────────┤
   │ Defer Decisions        │ Feature flags, Strategy pattern,       │
   │ (Last Responsible      │ Config over code, Plugin architecture  │
   │  Moment)               │                                        │
   ├────────────────────────┼────────────────────────────────────────┤
   │ Encapsulate Variation  │ Interface segregation,                 │
   │                        │ Anti-corruption layer, Adapters        │
   ├────────────────────────┼────────────────────────────────────────┤
   │ Design for Evolution   │ Versioned APIs, Schema registry,       │
   │                        │ Backward compatibility, Strangler Fig  │
   ├────────────────────────┼────────────────────────────────────────┤
   │ Modular Boundaries     │ Bounded contexts, Module contracts,    │
   │                        │ Independent deployability              │
   ├────────────────────────┼────────────────────────────────────────┤
   │ Observability First    │ Structured logs, Metrics, Traces       │
   │                        │ (Detect impact of changes fast)        │
   └────────────────────────┴────────────────────────────────────────┘

4. COMMUNICATION
   - Architecture Decision Log visible to all
   - "What changed and why" in sprint review
   - Impact assessment shared with affected teams
```

---

### 14. Decision Making: "How do you evaluate 'Build vs Buy' for a platform capability?"

**Answer:**
```
BUILD vs BUY DECISION FRAMEWORK:

STEP 1: DEFINE REQUIREMENTS PRECISELY
   What capability? (Auth, Search, Queue, Analytics, Payments)
   What scale? (RPS, data volume, latency)
   What customization? (Standard vs unique)

STEP 2: TOTAL COST OF OWNERSHIP (3-Year Horizon)

| COST CATEGORY        │ BUILD (Est.)     │ BUY (Vendor)       │
├──────────────────────┼──────────────────┼────────────────────┤
| Initial Development  │ $500K (6 mo)     │ $0                 |
| Infrastructure       │ $50K/yr          │ Included / $20K/yr |
| Licensing            │ $0               │ $100K/yr           |
| Maintenance (FTE)    │ 2 FTE = $300K/yr │ 0.5 FTE = $75K/yr  |
| Customization        │ Included         │ $50K/yr (if API)   |
| Opportunity Cost     │ High (core team) │ Low                |
| Switching Cost       │ Low              │ High (vendor lock) |
| 3-YEAR TOTAL         │ ~$1.55M          │ ~$0.68M            |

STEP 3: STRATEGIC FACTORS (Non-Financial)
| FACTOR               │ BUILD ADVANTAGE  │ BUY ADVANTAGE      |
├──────────────────────┼──────────────────┼────────────────────┤
| Differentiation      │ ✅ Core IP       │ ❌ Commodity       |
| Time to Market       │ ❌ 6-12 months   │ ✅ Weeks           |
| Control/Flexibility  │ ✅ Full          │ ❌ Vendor roadmap  |
| Team Growth          │ ✅ Deep skills   │ ❌ Vendor dependent│
| Compliance           │ ✅ Full control  │ ⚠️ Shared responsibility|
| Risk                 │ ❌ Technical     │ ❌ Vendor viability|

STEP 4: DECISION MATRIX (Weighted)
| CRITERION          │ WEIGHT │ BUILD | BUY  │
|--------------------|--------|-------|------|
| 3-Year TCO         │ 30%    | 2     | 5    |
| Time to Market     │ 25%    | 2     | 5    |
| Differentiation    │ 20%    | 5     | 1    |
| Control            │ 15%    | 5     | 2    |
| Team Capability    │ 10%    | 3     | 4    |
| WEIGHTED SCORE     │        | 3.1   | 3.6  │

STEP 5: HYBRID OPTIONS
- Build core, buy commodity (e.g., Build recommendation engine, buy Elasticsearch)
- Start with buy, migrate to build when scale justifies (Strangler Fig)
- Open source + managed service (e.g., Self-hosted Kafka → Confluent Cloud)

STEP 6: DOCUMENT IN ADR
"Chose BUY (Algolia) for search — TCO 55% lower, 
  3 months faster, search not differentiator. 
  Re-evaluate at $10M ARR or custom ranking needs."

RULE: Default to BUY for commodities. BUILD only for strategic differentiators.
```

---

### 15. Elicitation: "What's your process for going from vague business idea to architectural blueprint?"

**Answer:**
```
END-TO-END PROCESS (4-6 Weeks for Medium System):

WEEK 1: DISCOVERY & FRAMING
├── Stakeholder Interviews (5-10) — Goals, Pain, Constraints
├── Stakeholder Map & RAPID Roles
├── Problem Statement (1 paragraph)
├── Success Metrics (Business KPIs)
└── Initial Constraints (Budget, Timeline, Tech, Team)

WEEK 2: DOMAIN EXPLORATION
├── Event Storming Workshop (2-4 hrs) — Domain Events, Commands, Aggregates
├── Bounded Context Identification — Candidate Service Boundaries
├── Context Map — Relationships, Integration Patterns
├── Ubiquitous Language Glossary
└── Domain Model Sketch (Aggregates, Entities, VOs)

WEEK 3: REQUIREMENTS SPECIFICATION
├── Functional Requirements (User Stories, Use Cases)
├── NFR Workshop (QAW) — Quantified SLIs/SLOs
├── Constraints & Assumptions Log
├── Priority Matrix (MoSCoW + Business Value)
└── Risk Register (Technical, Schedule, Organizational)

WEEK 4: ARCHITECTURE SYNTHESIS
├── Candidate Architectures (2-3 options)
├── Trade-off Analysis (Weighted Criteria Matrix)
├── ADRs for Structural Decisions
├── C4 Diagrams (Level 1 Context, Level 2 Container)
├── Data Flow Diagram (Security/Compliance)
├── Deployment Topology (Environments, Regions)
├── Technology Selections (with Radar position)
├── Team Topology Alignment (Conway's Law)
└── Fitness Function Definitions

WEEK 5: VALIDATION & PLANNING
├── Spike Riskiest Assumptions (2-3 spikes, time-boxed)
├── Walking Skeleton Plan (End-to-end thin slice)
├── Cost Model (Infrastructure + Team)
├── Phased Roadmap (MVP → V1 → V2)
├── Architecture Review Board Presentation
└── Sign-off / Iterate

WEEK 6: HANDOFF TO DELIVERY
├── Sprint 0 Plan (Infra, CI/CD, Observability, Scaffolding)
├── Team Kickoff (Architecture vision, Decisions, Guardrails)
├── ADR Repository Access
├── Fitness Functions in CI Pipeline
└── Architecture Office Hours Schedule

KEY PRINCIPLES:
✅ Time-boxed (not open-ended)
✅ Collaborative (not solo architect)
✅ Decision-focused (produces ADRs, not just diagrams)
✅ Validated (spikes, prototypes before commitment)
✅ Handoff-ready (teams can execute immediately)
```