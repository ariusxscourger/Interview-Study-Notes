# SOFTWARE ARCHITECTURE FUNDAMENTALS
## Exam-Style Study Notes

**Roadmap Section:** Software Architecture → What is Software Architecture, What is a Software Architect, Levels of Architecture

---

## 1. WHAT IS SOFTWARE ARCHITECTURE?

### WHY? (Problem Statement)

**Software systems are complex.** Without architecture, you get:
- **Big ball of mud** — everything coupled to everything
- **Unmaintainable code** — changes break unrelated features
- **Scaling failures** — can't handle growth without rewrite
- **Team friction** — developers step on each other's toes
- **Technical debt compounding** — each shortcut makes next one harder

**Architecture solves this by making key decisions *early* and *explicitly*.**

> **Analogy:** Building a house without blueprints = software without architecture. You might get walls and a roof, but plumbing won't connect, load-bearing walls get removed during renovation, and the foundation cracks under the second floor.

---

### HOW? (Internal Mechanism)

**Software Architecture = Significant Decisions**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WHAT MAKES A DECISION "ARCHITECTURAL"           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  HIGH IMPACT              │  HARD TO CHANGE LATER                  │
│  ─────────────            │  ──────────────────                    │
│  • System-wide effects    │  • Requires coordinated migration      │
│  • Affects multiple teams │  • Data migration / schema changes     │
│  • Impacts NFRs           │  • Contract breaking (APIs, events)    │
│  • Cost of failure = high │  • Skill/organizational dependencies   │
│                           │                                        │
│  EXAMPLES:                │  EXAMPLES:                             │
│  • Database choice        │  • Monolith → Microservices            │
│  • Communication style    │  • Sync → Async messaging              │
│  • Deployment topology    │  • SQL → NoSQL                         │
│  • Security model         │  • Framework migration                 │
│                           │                                        │
│  NON-ARCHITECTURAL:       │                                        │
│  • Variable naming        │  Low impact, easy to change            │
│  • Library version        │  (unless it drives architecture)       │
│  • Code formatting        │                                        │
│                           │                                        │
└─────────────────────────────────────────────────────────────────────┘
```

**Architecture captures:**
1. **Structure** — Components, relationships, boundaries
2. **Behavior** — Communication patterns, data flow, control flow
3. **Properties** — Quality attributes (NFRs): performance, security, availability, scalability
4. **Principles** — Guidelines that govern future decisions
5. **Rationale** — *Why* each decision was made (critical for evolution)

---

### WHAT? (Key Concepts & Definitions)

| Concept | Definition |
|---------|------------|
| **Component** | Deployable/runnable unit with clear interface (service, library, module) |
| **Connector** | Mechanism for component interaction (HTTP, gRPC, message queue, shared DB) |
| **Configuration** | Arrangement of components + connectors at runtime |
| **Architecture Style** | Named pattern of organization (Layered, Microservices, Event-Driven, CQRS) |
| **Architecture Pattern** | Reusable solution to recurring problem (Circuit Breaker, Saga, Outbox) |
| **Quality Attribute (NFR)** | Non-functional requirement: latency, throughput, availability, modifiability |
| **Architecture Decision Record (ADR)** | Document capturing context, decision, consequences, alternatives considered |

---

### LEVELS OF ARCHITECTURE (Roadmap: "Levels of Architecture")

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ARCHITECTURE HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ENTERPRISE ARCHITECTURE (EA)                                              │
│   ─────────────────────────                                                 │
│   Scope:  Entire organization / portfolio                                   │
│   Focus:  Business-IT alignment, standards, governance, roadmap            │
│   Artifacts:  Business capability map, application portfolio, tech radar   │
│   Frameworks:  TOGAF, Zachman, FEAF                                         │
│   Decisions:  Cloud strategy, vendor selection, data governance,           │
│               platform standards, security baseline                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                   SOLUTION ARCHITECTURE                              │  │
│   │  ────────────────────────────────────                               │  │
│   │  Scope:  Single business problem / initiative                       │  │
│   │  Focus:  How components solve a specific business need             │  │
│   │  Artifacts:  Context diagram, component diagram, sequence diagrams │  │
│   │  Decisions:  Integration patterns, data flow, non-functional       │  │
│   │              requirements per capability, vendor/products           │  │
│   │                                                                     │  │
│   │   ┌─────────────────────────────────────────────────────────────┐  │  │
│   │   │                 APPLICATION ARCHITECTURE                    │  │  │
│   │   │  ──────────────────────────────────────                     │  │  │
│   │   │  Scope:  Single application / service                       │  │  │
│   │   │  Focus:  Internal structure, modules, layers, data model   │  │  │
│   │   │  Artifacts:  Module diagram, class diagram, ER diagram,    │  │  │
│   │   │              API contracts, deployment topology             │  │  │
│   │   │  Decisions:  Layering, DDD bounded contexts, DB schema,    │  │  │
│   │   │              caching strategy, internal APIs                │  │  │
│   │   │                                                             │  │  │
│   │   │   ┌─────────────────────────────────────────────────────┐  │  │  │
│   │   │   │              CODE / MODULE ARCHITECTURE              │  │  │  │
│   │   │   │  ─────────────────────────────────────────           │  │  │  │
│   │   │   │  Scope:  Single module / component                   │  │  │  │
│   │   │   │  Focus:  Classes, functions, algorithms, patterns    │  │  │  │
│   │   │   │  Artifacts:  Clean architecture layers, SOLID,       │  │  │  │
│   │   │   │              design patterns, test structure          │  │  │  │
│   │   │   └─────────────────────────────────────────────────────┘  │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### ARCHITECTURE VS DESIGN

| Architecture | Design |
|--------------|--------|
| **What** — High-level structure | **How** — Low-level implementation |
| **Strategic** — Hard to change | **Tactical** — Easier to change |
| **Cross-cutting** — Affects many | **Localized** — Within a component |
| **NFR-driven** — Performance, security | **FR-driven** — Features, algorithms |
| **Team/Org impact** — Conway's Law | **Developer impact** — Code quality |

> **Rule of thumb:** If changing it requires coordinating multiple teams, migrating data, or breaking contracts — it's architecture. If it's refactoring within a module — it's design.

---

## 2. WHAT IS A SOFTWARE ARCHITECT?

### WHY? (Problem Statement)

**Organizations need someone who:**
- Bridges business strategy ↔ technical execution
- Makes decisions with incomplete information
- Balances competing concerns (speed vs quality, cost vs scalability)
- Prevents "architecture by accident" (emergent chaos)
- Enables teams to work in parallel without constant sync

---

### HOW? (Role Mechanics)

**The Architect's Loop:**
```
┌────────────────────────────────────────────────────────────────────┐
│                    ARCHITECT'S CONTINUOUS LOOP                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   1. UNDERSTAND CONTEXT                                           │
│      ├─ Business goals, constraints, stakeholders                │
│      ├─ Technical landscape, existing systems, debt              │
│      ├─ Team topology, skills, culture                           │
│      └─ Regulatory, compliance, security requirements            │
│                        │                                          │
│                        ▼                                          │
│   2. MAKE DECISIONS                                               │
│      ├─ Identify architecturally significant requirements (ASRs) │
│      ├─ Generate alternatives (at least 3)                       │
│      ├─ Evaluate trade-offs (cost, risk, time, quality)          │
│      ├─ Document in ADR (context, decision, consequences)        │
│      └─ Communicate & get buy-in                                  │
│                        │                                          │
│                        ▼                                          │
│   3. GOVERN & EVOLVE                                              │
│      ├─ Review PRs/designs for architectural compliance          │
│      ├─ Mentor & coach developers                                │
│      ├─ Monitor drift (architecture erosion)                     │
│      ├─ Evolve architecture incrementally (Strangler Fig)        │
│      └─ Retire decisions when context changes                    │
│                        │                                          │
│                        ▼                                          │
│   ────────────────────────────────────────────────────────────    │
│   │                    FEEDBACK LOOP                              │
│   │  Production incidents → Architecture retrospectives          │
│   │  New requirements → Re-evaluate decisions                    │
│   │  Team growth → Adjust boundaries                              │
│   ────────────────────────────────────────────────────────────    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Responsibilities & Skills)

| Responsibility | Description | Key Activities |
|----------------|-------------|----------------|
| **Technical Strategy** | Define technical direction aligned with business | Tech radar, POCs, vendor eval, roadmap |
| **Architecture Decisions** | Make & document significant decisions | ADRs, trade-off analysis, alternatives |
| **Standards & Governance** | Establish patterns, conventions, guardrails | Coding standards, approved libraries, linting |
| **Quality Attributes** | Ensure NFRs are met (perf, security, availability) | SLIs/SLOs, load testing, threat modeling |
| **Stakeholder Communication** | Translate technical ↔ business | Diagrams, presentations, risk articulation |
| **Team Enablement** | Unblock teams, reduce cognitive load | Platform tools, shared libraries, templates |
| **Risk Management** | Identify & mitigate technical risks | Risk register, spike experiments, fallback plans |

**Anti-Patterns to Avoid:**
- ❌ **Ivory Tower Architect** — Decides in isolation, throws docs over wall
- ❌ **Bottleneck Architect** — Every decision goes through them
- ❌ **Resume-Driven Architect** — Chases shiny tech, ignores context
- ❌ **Diagram-Only Architect** — Produces pictures, no code or governance

**Effective Archetype:** **Hands-on Architect** — Codes (spikes, critical paths), reviews, pairs, owns production incidents.

---

## 3. LEVELS OF ARCHITECTURE (Deep Dive)

### 3.1 APPLICATION ARCHITECTURE

**Scope:** Single deployable unit (service, monolith, mobile app)

#### Core Decisions:

| Decision Area | Options | Trade-offs |
|---------------|---------|------------|
| **Layering** | Clean Arch, Onion, Hexagonal, Layered | Testability vs complexity, coupling direction |
| **Modularity** | Modular monolith, feature folders, vertical slices | Deployability vs coupling, team autonomy |
| **Data Access** | EF Core, Dapper, raw SQL, Repository pattern | Abstraction vs performance, testability |
| **Communication** | MediatR (in-process), HTTP, gRPC, messages | Coupling, latency, operational complexity |
| **State Management** | Stateful vs stateless, session storage | Scalability, affinity, consistency |

#### Application Architecture Diagram Template:

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                       │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │  REST API   │ │  GraphQL    │ │  gRPC       │ │ SignalR   │ │
│  │  (Controllers)│ │  (Schema)   │ │  (Protobuf) │ │  (WebSocket)│
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └─────┬────┘ │
│         │               │               │             │       │
├─────────┼───────────────┼───────────────┼─────────────┼───────┤
│         ▼               ▼               ▼             ▼       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    APPLICATION LAYER                     │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │  │
│  │  │  Commands   │ │  Queries    │ │  Events     │        │  │
│  │  │  (Handlers) │ │  (Handlers) │ │  (Handlers) │        │  │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘        │  │
│  └─────────┼───────────────┼───────────────┼────────────────┘  │
│            │               │               │                   │
│            ▼               ▼               ▼                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                      DOMAIN LAYER                        │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐    │  │
│  │  │Entities │ │Value Obj│ │Aggregates│ │Domain Events│    │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └──────┬──────┘    │  │
│  └───────┼────────────┼───────────┼─────────────┼───────────┘  │
│           │            │           │             │              │
├───────────┼────────────┼───────────┼─────────────┼──────────────┤
│           ▼            ▼           ▼             ▼              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  INFRASTRUCTURE LAYER                    │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │  │
│  │  │EF Core   │ │ Redis    │ │ HTTP     │ │ Message    │  │  │
│  │  │(Postgres)│ │ Cache    │ │ Clients  │ │ Bus (Kafka)│  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.2 SOLUTION ARCHITECTURE

**Scope:** End-to-end solution for a business capability (e.g., "Order Management", "Customer Onboarding")

#### Core Concerns:

```
┌────────────────────────────────────────────────────────────────────┐
│                    SOLUTION ARCHITECTURE CONCERNS                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │   CONTEXT       │  │   COMPONENTS    │  │   CONNECTORS    │   │
│  │   ────────────  │  │   ────────────  │  │   ────────────  │   │
│  │  • Business     │  │  • Services     │  │  • Sync (REST,  │   │
│  │    capability   │  │  • Databases    │  │    gRPC)        │   │
│  │  • Actors/Users │  │  • Queues/Topics│  │  • Async (Kafka,│   │
│  │  • External     │  │  • Caches       │  │    RabbitMQ)    │   │
│  │    systems      │  │  • Event stores │  │  • Events       │   │
│  │  • Boundaries   │  │  • UI/Apps      │  │  • File transfer│   │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘   │
│           │                    │                    │             │
│           ▼                    ▼                    ▼             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    DATA FLOW & INTEGRATION                  │  │
│  │  • Master data management    • Event choreography          │  │
│  │  • Reference data sync       • Saga orchestration          │  │
│  │  • CDC / replication         • API gateway routing         │  │
│  │  • Data ownership            • Contract testing            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    QUALITY ATTRIBUTES                       │  │
│  │  • Latency budgets per flow    • Availability targets      │  │
│  │  • Throughput requirements     • Disaster recovery (RPO/RTO)│  │
│  │  • Consistency boundaries      • Security zones            │  │
│  │  • Observability (logs/metrics/traces)                     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

#### Solution Architecture Artifacts:

| Artifact | Purpose | Audience |
|----------|---------|----------|
| **Context Diagram (C4 Level 1)** | System + external actors | Business stakeholders |
| **Container Diagram (C4 Level 2)** | Deployable units + tech | Architects, Tech Leads |
| **Component Diagram (C4 Level 3)** | Internal modules | Developers |
| **Sequence Diagrams** | Key flows (happy path + errors) | All technical |
| **Deployment Diagram** | Infrastructure, networks, regions | DevOps, Infra |
| **Data Flow Diagram** | PII, sensitive data, trust boundaries | Security, Compliance |
| **ADR Log** | Decision history | Architects, Future maintainers |

---

### 3.3 ENTERPRISE ARCHITECTURE

**Scope:** Organization-wide technology portfolio, strategy, governance

#### EA Frameworks:

| Framework | Focus | Best For |
|-----------|-------|----------|
| **TOGAF** | Process (ADM), deliverables, governance | Large enterprises, formal governance |
| **Zachman** | Classification matrix (What/How/Where/Who/When/Why) | Taxonomy, completeness checking |
| **FEAF** | US Federal government reference model: Federal govt specific | Govt agencies |
| **ArchiMate** | Modeling language (not process) | Visualization, tooling |
| **SABSA** | Security architecture | Security-first organizations |

#### EA Domains (TOGAF):

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE ARCHITECTURE DOMAINS                  │
├──────────────────────┬──────────────────────┬──────────────────────┤
│   BUSINESS ARCH      │  INFORMATION ARCH    │  APPLICATION ARCH    │
│   ──────────────     │  ─────────────────   │  ─────────────────   │
│   • Strategy         │   • Data governance  │   • App portfolio    │
│   • Capabilities     │   • Master data      │   • Integration      │
│   • Processes        │   • Analytics/BI     │   • Standards        │
│   • Organization     │   • Privacy/GDPR     │   • Lifecycle mgmt   │
│   • Value streams    │   • Metadata         │   • Rationalization  │
├──────────────────────┼──────────────────────┼──────────────────────┤
│   TECHNOLOGY ARCH    │                      │                      │
│   ────────────────   │   CROSS-CUTTING      │                      │
│   • Infrastructure   │   ───────────────    │                      │
│   • Platforms        │   • Security         │                      │
│   • Networks         │   • Compliance       │                      │
│   • Cloud strategy   │   • Observability    │                      │
│   • Vendor mgmt      │   • Cost optimization│                      │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

#### EA Deliverables:

| Deliverable | Cadence | Owner |
|-------------|---------|-------|
| **Technology Radar** | Quarterly | EA Team |
| **Application Portfolio** | Annual + updates | EA + App Owners |
| **Architecture Principles** | Review annually | CTO / EA Lead |
| **Standards & Patterns** | As needed | Architecture Board |
| **Roadmap & Target State** | Annual + initiative-based | EA + Business |
| **Compliance Reports** | Quarterly / audit-driven | EA + Security |

---

## PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Document Architecture Decisions (ADRs)** | Always | Verbal decisions / tribal knowledge | Decisions lost, can't evaluate trade-offs later |
| **Architecture Fitness Functions** | CI/CD pipeline | Manual architecture reviews only | Drift undetected until production incident |
| **Incremental Architecture (Strangler Fig)** | Legacy modernization | Big-bang rewrite | Risk, cost, time, business disruption |
| **Architecture as Code** | Platform teams | Diagrams in Confluence only | Diagrams rot, code is source of truth |
| **Architectural Guardrails** | Large orgs, many teams | "Guidelines" without enforcement | Inconsistent implementations, security gaps |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. Conceptual: "What is software architecture? How is it different from design?"

**Answer:**
```
Architecture = Significant decisions that are hard to change + affect system-wide properties
Design = Lower-level decisions within architectural boundaries

Key distinction: 
- Architecture: "We use event-driven microservices with async communication" 
  (affects scaling, teams, data consistency, ops, contracts)
- Design: "OrderService uses MediatR for commands and Entity Framework for persistence"
  (changeable within service, doesn't affect other services)

Test: "If I change this, do I need to coordinate with other teams / migrate data / break contracts?"
  Yes → Architecture. No → Design.
```

---

### 2. Conceptual: "Explain the three levels of architecture with examples."

**Answer:**
```
1. APPLICATION ARCHITECTURE (Single deployable)
   Example: Order Service internal structure
   - Clean Architecture layers (Domain, Application, Infrastructure)
   - Module boundaries: Orders, Payments, Inventory within service
   - Database schema, caching strategy, internal APIs
   
2. SOLUTION ARCHITECTURE (Business capability)
   Example: "Customer places order" end-to-end
   - Services: API Gateway → Order → Payment → Inventory → Shipping → Notification
   - Data flow: OrderCreated event → PaymentReserved → StockReserved → ShipmentScheduled
   - Integration: Sync for payment auth, async for fulfillment
   - NFRs: Order latency <500ms, payment 99.99% availability, eventual consistency OK for shipping
   
3. ENTERPRISE ARCHITECTURE (Organization portfolio)
   Example: Retail company technology strategy
   - Business capabilities: Merchandising, Supply Chain, E-commerce, Stores, Loyalty
   - Platform: Kubernetes on Azure, Kafka event backbone, PostgreSQL/CosmosDB standards
   - Governance: API standards, security baseline, observability stack (OpenTelemetry)
   - Roadmap: Decompose monolith → domain services → platform services
```

---

### 3. Role: "What does a software architect actually do day-to-day?"

**Answer:**
```
NOT: Sit in ivory tower drawing diagrams all day.

YES:
├── 30% — Technical decisions & ADRs (evaluate alternatives, document trade-offs)
├── 25% — Code & technical spikes (prove architectures, unblock teams)
├── 20% — Reviews & governance (PR reviews, design reviews, fitness functions)
├── 15% — Stakeholder communication (translate tech ↔ business, risk articulation)
├── 10% — Mentoring & enabling (pair programming, patterns, platform tools)

Key principle: "Architects who don't code become administrators."
```

---

### 4. Decision Making: "How do you make an architectural decision?"

**Answer:**
```
1. IDENTIFY ASRs (Architecturally Significant Requirements)
   - Which NFRs drive this? (latency, consistency, throughput, team autonomy)

2. GENERATE ALTERNATIVES (minimum 3)
   - Don't jump to first idea. Document: Option A, B, C with pros/cons

3. EVALUATE TRADE-OFFS
   - Use decision matrix: Cost, Risk, Time, Operational complexity, Team skills
   - Explicitly state what you're giving up

4. DOCUMENT IN ADR
   - Title, Status, Context, Decision, Consequences, Alternatives considered
   - Link to related ADRs

5. SOCIALIZE & GET BUY-IN
   - Present to affected teams, address concerns, iterate

6. IMPLEMENT & GOVERN
   - Fitness functions in CI, code reviews, monitoring alerts
```

---

### 5. Evolution: "How do you evolve architecture without big-bang rewrites?"

**Answer:**
```
STRANGLER FIG PATTERN:
1. IDENTIFY bounded context to extract
2. CREATE new service alongside monolith
3. ROUTE new traffic to new service (feature flags / API gateway)
4. MIGRATE data incrementally (dual write → read from new → delete old)
5. DEPRECATE old code

PRINCIPLES:
- Deploy independently from day 1
- Observability parity (logs, metrics, traces)
- Contract tests between old/new
- Rollback plan for each increment
- Measure: error rate, latency, business metrics

NEVER: "Stop the world" rewrite. It fails 90% of time.
```

---

### 6. Quality Attributes: "How do you handle NFRs in architecture?"

**Answer:**
```
MAKE NFRs EXPLICIT AND MEASURABLE:

| NFR | Metric (SLI) | Target (SLO) | Architecture Lever |
|-----|--------------|--------------|-------------------|
| Latency | p99 response time | < 200ms | Caching, read replicas, CDN, async |
| Availability | Uptime % | 99.95% | Multi-AZ, circuit breaker, retry, graceful degradation |
| Throughput | req/sec sustained | 10K RPS | Horizontal scaling, partitioning, queue buffering |
| Consistency | Staleness window | < 1s (eventual) | Event sourcing, CDC, saga |
| Security | Time to detect/respond | < 15 min | Zero-trust, mTLS, secrets rotation, audit logs |
| Modifiability | Lead time for change | < 1 day | Modular boundaries, feature flags, contract tests |

ARCHITECTURE DECISIONS ARE NFR TRADE-OFFS.
Example: "We chose eventual consistency (AP) for shopping cart 
because availability during partition > strong consistency. 
Compensation: reconciliation job runs hourly."
```

---

### 7. Communication: "How do you communicate architecture to non-technical stakeholders?"

**Answer:**
```
USE THE RIGHT ARTIFACT FOR THE AUDIENCE:

Executive/Business:
├── Business capability map (what we do)
├── Roadmap with business outcomes (not tech tasks)
├── Risk register (business impact, likelihood, mitigation)
└── Investment ask (cost, ROI, timeline)

Product/Management:
├── Context diagram (C4 Level 1) — system + actors
├── Sequence diagrams for key user journeys
├── NFR commitments (SLA/SLO)
└── Dependency map (what blocks what)

Engineering:
├── Container diagram (C4 Level 2) — services, DBs, queues
├── Component diagram (C4 Level 3) — modules, interfaces
├── ADRs (decision log)
├── API contracts (OpenAPI/Protobuf)
├── Deployment topology
└── Runbooks / incident response

NEVER show: Class diagrams, implementation details to non-engineers.
```

---

### 8. Governance: "How do you prevent architecture drift?"

**Answer:**
```
ARCHITECTURE FITNESS FUNCTIONS (automated guardrails in CI/CD):

STATIC ANALYSIS:
├── ArchUnit / NetArchTest / go-arch-lint — enforce layer rules
├── Dependency direction — Domain doesn't reference Infrastructure
├── Naming conventions — Commands end with Command, Events past tense
├── Banned dependencies — No Newtonsoft.Json (use System.Text.Json)

CONTRACT TESTS:
├── Consumer-driven contracts (Pact) — API compatibility
├── Schema registry (Confluent) — Event compatibility (AVRO/Protobuf)
├── Database migration tests — Backward compatibility

RUNTIME:
├── SLO alerts — Latency, error rate, saturation
├── Distributed trace sampling — Verify call paths
├── Chaos engineering — Verify resilience (Litmus, Chaos Mesh)

PROCESS:
├── Architecture review gate for new services
├── Quarterly architecture retrospective
├── Tech debt registry with owners & due dates
```

---

### 9. Microservices: "When do you NOT use microservices?"

**Answer:**
```
AVOID MICROSERVICES WHEN:

✓ Team < 10 developers (communication overhead > benefit)
✓ Domain not well understood (boundaries will be wrong)
✓ Strong consistency required across boundaries (distributed transactions = pain)
✓ Low scaling requirements (vertical scaling simpler)
✓ Simple CRUD application (no complex domain logic)
✓ No DevOps maturity (can't operate 20+ services)
✓ Regulatory constraints (data residency, audit — harder distributed)

START WITH: Modular Monolith
├── Clear module boundaries (namespaces, assemblies)
├── Shared database, separate schemas
├── In-process communication (MediatR)
├── Independent deployability via feature flags
└── Extract service ONLY when:
    - Team autonomy needed
    - Different scaling profile
    - Different technology stack
    - Different release cadence
```

---

### 10. Cloud: "How do you design for cloud vs on-prem?"

**Answer:**
```
CLOUD-NATIVE PRINCIPLES (12-Factor + Cloud Design Patterns):

ASSUME FAILURE:
├── Design for: instance death, AZ outage, region outage
├── Stateless services (any instance can handle any request)
├── Externalize state (DB, cache, blob storage, config)
├── Health endpoints + readiness/liveness probes

ELASTICITY:
├── Horizontal scaling (add instances, not bigger VMs)
├── Queue-based load leveling (absorb spikes)
├── Autoscale on custom metrics (queue depth, business KPIs)

OBSERVABILITY BY DEFAULT:
├── Structured logging (JSON, correlation IDs)
├── Metrics (RED: Rate, Errors, Duration)
├── Distributed tracing (W3C TraceContext)

SECURITY:
├── Zero-trust network (mTLS via service mesh)
├── Secrets in vault (not config files)
├── Least privilege (IRSA, Workload Identity)

COST AWARENESS:
├── Right-sizing (vertical pod autoscaler)
├── Spot instances for batch/async
├── Data tiering (hot/warm/cold storage)
```

---

### 11. Data: "How do you handle data consistency in distributed systems?"

**Answer:**
```
CONSISTENCY SPECTRUM (strongest → weakest):

1. STRONG CONSISTENCY (ACID)
   - Single database, distributed transactions (2PC) — AVOID at scale
   - Use: Financial ledgers, inventory reservation
   
2. EVENTUAL CONSISTENCY (BASE)
   - Write to one service, propagate via events
   - Use: Most business operations (order → payment → inventory)
   - Pattern: Saga (choreography or orchestration)
   
3. CAUSAL CONSISTENCY
   - Preserve cause-effect ordering (vector clocks, version vectors)
   - Use: Collaborative editing, social feeds

4. READ-YOUR-WRITES
   - After write, subsequent reads see it
   - Implementation: Route writer's reads to primary, or sync replication

PATTERNS:
├── SAGA (choreography) — Events trigger next step, compensating events on failure
├── SAGA (orchestration) — Central coordinator tells participants what to do
├── OUTBOX — DB write + event in same transaction → reliable publishing
├── CDC (Debezium) — Capture DB changes → publish events
├── EVENT SOURCING — Store events, not state → full audit, temporal queries
└── CQRS — Separate read/write models → optimize each independently

DECISION TREE:
Need ACID across services? → Saga (orchestrated for complex, choreographed for simple)
High throughput, eventual OK? → Event-driven + Outbox
Audit required? → Event Sourcing
Complex read queries? → CQRS + Read Models
```

---

### 12. Security: "How do you architect for security?"

**Answer:**
```
SECURITY BY DESIGN — THREAT MODELING (STRIDE):

┌─────────────────────────────────────────────────────────────────┐
│  S - Spoofing         │  AuthN: mTLS, JWT, OAuth2, OIDC        │
│  T - Tampering        │  Integrity: Signing, checksums, HMAC   │
│  R - Repudiation      │  Audit: Immutable logs, non-repudiation│
│  I - Info Disclosure  │  Encryption: TLS 1.3, AES-256 at rest  │
│  D - DoS              │  Rate limit, circuit breaker, quotas   │
│  E - Elevation        │  AuthZ: RBAC/ABAC, least privilege     │
└─────────────────────────────────────────────────────────────────┘

ARCHITECTURE PATTERNS:
├── ZERO TRUST — No implicit trust, verify every request
├── SERVICE MESH — mTLS, authZ, observability at infra layer
├── API GATEWAY — Single entry point, auth, rate limit, validation
├── SECRETS MANAGEMENT — Vault, AWS Secrets Manager, Azure Key Vault
├── SHIFT LEFT — SAST/DAST/SCA in CI, threat modeling in design
└── INCIDENT RESPONSE — Runbooks, forensics, rotation procedures
```

---

### 13. Legacy: "How do you modernize a legacy monolith?"

**Answer:**
```
ASSESSMENT FIRST:
├── Codebase analysis (coupling, complexity, test coverage)
├── Business criticality & change frequency per module
├── Team structure & ownership
├── Technical debt hotspots (SonarQube, NDepend)

STRATEGY (choose per module):

1. STRANGLER FIG (extract service)
   - Best for: High-change, well-bounded, different scale needs
   
2. REFACTOR IN PLACE (modular monolith)
   - Best for: Stable, low-change, tightly coupled domain
   - Enforce: Module boundaries, internal APIs, tests
   
3. REWRITE (last resort)
   - Best for: Throwaway prototype, unsalvageable tech stack
   - Rule: Incremental delivery, not big bang

4. REPLATFORM (lift & shift + optimize)
   - Best for: COBOL/mainframe, vendor lock-in
   - Automate: Tests, deployment, parity checks

KEY SUCCESS FACTORS:
✅ Executive sponsorship & funding
✅ Dedicated team (not "side project")
✅ Comprehensive test suite BEFORE changes
✅ Feature flags for gradual rollout
✅ Observability parity old ↔ new
✅ Rollback plan for each increment
✅ Celebrate deletions (code removed = debt paid)
```

---

### 14. Team Topology: "How does architecture relate to team structure?"

**Answer:**
```
CONWAY'S LAW: "Organizations design systems that mirror their communication structures."

TEAM TOPOLOGIES (Matthew Skelton & Manuel Pais):

┌──────────────────┬─────────────────────────────────────────────────────┐
│ TEAM TYPE        │ PURPOSE                                            │
├──────────────────┼─────────────────────────────────────────────────────┤
│ STREAM-ALIGNED   │ Owns a business capability end-to-end              │
│ (Product Team)   │ Long-lived, cross-functional, autonomous           │
├──────────────────┼─────────────────────────────────────────────────────┤
│ PLATFORM         │ Builds internal developer platform (IDP)           │
│                  │ Reduces cognitive load for stream teams            │
├──────────────────┼─────────────────────────────────────────────────────┤
│ ENABLING         │ Helps teams adopt new tech/practices (coaching)    │
│                  │ Temporary, expert knowledge transfer               │
├──────────────────┼─────────────────────────────────────────────────────┤
│ COMPLICATED-     │ Specialized subsystem requiring deep expertise     │
│ SUBSYSTEM        │ (e.g., ML platform, real-time trading engine)      │
└──────────────────┴─────────────────────────────────────────────────────┘

INTERACTION MODES:
├── COLLABORATION — Close work, shared goals (short-term)
├── X-AS-A-SERVICE — Platform provides self-service APIs (long-term)
└── FACILITATING — Enabling team coaches/upskills (temporary)

ARCHITECTURE IMPLICATION:
- Service boundaries ≈ Team boundaries
- Platform = shared infrastructure (k8s, CI/CD, observability, auth)
- API contracts = Team interfaces (consumer-driven contracts)
- Cognitive load = Architecture complexity per team
```

---

### 15. Metrics: "How do you measure architecture health?"

**Answer:**
```
LEADING INDICATORS (predictive):
├── Architecture Fitness Function pass rate (CI)
├── ADR coverage (% significant decisions documented)
├── Cyclomatic complexity per module
├── Coupling metrics (afferent/efferent, instability)
├── Test coverage (unit + contract + integration)
├── Dependency freshness (CVE count, version lag)
├── Cognitive load (files/module, lines of code, domains/team)
└── Technical debt ratio (SonarQube)

LAGGING INDICATORS (outcome):
├── Deployment frequency & lead time (DORA)
├── Change failure rate & MTTR (DORA)
├── Availability (SLO compliance)
├── Incident count by severity
├── Performance (p50/p95/p99 latency)
├── Security incidents / vuln time-to-fix
├── Developer satisfaction (survey)
└── Time to onboard new developer

ARCHITECTURE HEALTH DASHBOARD:
- Green: All fitness functions passing, debt decreasing
- Yellow: Some fitness functions failing, debt stable
- Red: Critical fitness functions failing, debt increasing

REVIEW CADENCE: Weekly (team), Monthly (architecture board), Quarterly (executive)
```