# FRAMEWORKS & METHODOLOGIES
## Exam-Style Study Notes

**Roadmap Section:** Frameworks → PMI, ITIL, RUP, BABOK, Prince2, ESB/SOAP, REST, GraphQL, IAF, BPM/BPEL, UML, Messaging Queues, Datawarehouse Principles, Infrastructure as Code, Certifications

---

## 1. PMI (PROJECT MANAGEMENT INSTITUTE)

### WHY? (Problem Statement)

**Projects fail without structure.** PMI provides standardized framework for project delivery.

---

### HOW? (PMBOK Guide - 7th Edition Principles)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PMI PROJECT MANAGEMENT PRINCIPLES                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. BE A DILIGENT, RESPECTFUL, CARING STEWARD                              │
│  2. CREATE A COLLABORATIVE PROJECT TEAM ENVIRONMENT                        │
│  3. EFFECTIVELY ENGAGE WITH STAKEHOLDERS                                   │
│  4. FOCUS ON VALUE                                                          │
│  5. RECOGNIZE, EVALUATE, AND RESPOND TO SYSTEM INTERACTIONS               │
│  6. DEMONSTRATE LEADERSHIP BEHAVIORS                                       │
│  7. TAILOR BASED ON CONTEXT                                                │
│  8. BUILD QUALITY INTO PROCESSES AND DELIVERABLES                          │
│  9. NAVIGATE COMPLEXITY                                                    │
│  10. OPTIMIZE RISK RESPONSES                                               │
│  11. EMBRACE ADAPTABILITY AND RESILIENCE                                   │
│  12. ENABLE CHANGE TO ACHIEVE ENVISIONED FUTURE STATE                      │
│                                                                             │
│  PERFORMANCE DOMAINS (vs old Knowledge Areas):                             │
│  ┌────────────────┬────────────────┬────────────────┬────────────────────┐ │
│  │ Stakeholder    │ Team           │ Development    │ Planning           │ │
│  │ Performance    │ Performance    │ Approach &     │ Performance        │ │
│  │ Domain         │ Domain         │ Life Cycle     │ Domain             │ │
│  ├────────────────┼────────────────┼────────────────┼────────────────────┤ │
│  │ Project Work   │ Delivery       │ Measurement    │ Uncertainty        │ │
│  │ Domain         │ Domain         │ Domain         │ Domain             │ │
│  └────────────────┴────────────────┴────────────────┴────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Artifacts for Architects)

| Artifact | Architect's Role |
|----------|------------------|
| **Project Charter** | Define technical scope, constraints, success criteria |
| **Stakeholder Register** | Identify technical stakeholders, communication needs |
| **WBS (Work Breakdown Structure)** | Decompose architecture into deliverables |
| **Risk Register** | Technical risks (tech debt, scalability, security) |
| **Change Log** | Architecture changes, ADR references |
| **Lessons Learned** | Architecture retrospectives |

---

## 2. ITIL (IT INFRASTRUCTURE LIBRARY)

### WHY? (Problem Statement)

**IT services need standardized management.** ITIL = best practices for IT service management (ITSM).

---

### HOW? (ITIL 4 - Service Value System)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ITIL 4 SERVICE VALUE SYSTEM                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GUIDING PRINCIPLES:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Focus on Value                                                   │   │
│  │ 2. Start Where You Are                                              │   │
│  │ 3. Progress Iteratively with Feedback                               │   │
│  │ 4. Collaborate and Promote Visibility                               │   │
│  │ 5. Think and Work Holistically                                      │   │
│  │ 6. Keep It Simple and Practical                                     │   │
│  │ 7. Optimize and Automate                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SERVICE VALUE CHAIN:                                                      │
│  Plan → Improve → Engage → Design & Transition → Obtain/Build → Deliver & Support │
│                                                                             │
│  PRACTICES (34 total - Key for Architects):                               │
│  ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐ │
│  │ General Mgmt    │ Service Mgmt    │ Technical Mgmt  │ Architect Role  │ │
│  ├─────────────────┼─────────────────┼─────────────────┼─────────────────┤ │
│  │ • Architecture  │ • Incident      │ • Deployment    │ • Architecture  │ │
│  │   Management    │   Management    │   Management    │   Management    │ │
│  │ • Continual     │ • Problem       │ • Infrastructure│ • Service       │ │
│  │   Improvement   │   Management    │   & Platform    │   Design        │ │
│  │ • Information   │ • Change        │   Management    │ • Change        │ │
│  │   Security      │   Enablement    │ • Software      │   Enablement    │ │
│  │ • Knowledge     │ • Service       │   Development   │ • Deployment    │ │
│  │   Management    │   Request       │   & Management  │   Management    │ │
│  │ • Measurement   │   Management    │                 │ • Capacity/Perf │ │
│  │ • Org Change    │ • Service Level │                 │   Management    │ │
│  │   Management    │   Management    │                 │ • Availability  │ │
│  │ • Portfolio     │ • Availability  │                 │   Management    │ │
│  │   Management    │   Management    │                 │                 │ │
│  │ • Project       │ • Capacity &    │                 │                 │ │
│  │   Management    │   Performance   │                 │                 │ │
│  │ • Relationship  │   Management    │                 │                 │ │
│  │   Management    │ • Continuity    │                 │                 │ │
│  │ • Risk          │   Management    │                 │                 │ │
│  │   Management    │ • Release       │                 │                 │ │
│  │ • Strategy      │   Management    │                 │                 │ │
│  │   Management    │ • Deployment    │                 │                 │ │
│  │ • Supplier      │   Management    │                 │                 │ │
│  │   Management    │                 │                 │                 │ │
│  │ • Workforce &   │                 │                 │                 │ │
│  │   Talent Mgmt   │                 │                 │                 │ │
│  └─────────────────┴─────────────────┴─────────────────┴─────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Architect's ITIL Touchpoints)

| Practice | Architect Deliverable |
|----------|----------------------|
| **Architecture Management** | Architecture repository, principles, standards, patterns |
| **Service Design** | Service catalog entries, SLAs, capacity plans |
| **Change Enablement** | Architecture review board, RFC/ADR process |
| **Deployment Management** | Deployment architecture, rollback procedures |
| **Release Management** | Versioning strategy, compatibility matrix |
| **Availability Management** | HA architecture, SLO definitions, DR plans |
| **Capacity & Performance** | Scaling architecture, performance budgets |
| **Service Continuity** | DR architecture, RPO/RTO, backup strategy |

---

## 3. RUP (RATIONAL UNIFIED PROCESS)

### WHY? (Problem Statement)

**Heavyweight process for large, complex systems.** RUP = iterative, architecture-centric, risk-driven.

---

### HOW? (RUP Phases & Disciplines)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RUP LIFECYCLE PHASES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INCEPTION                    ELABORATION                  CONSTRUCTION    │
│  ─────────────                ──────────────               ─────────────   │
│  • Define scope               • Baseline architecture      • Build product │
│  • Business case              • Mitigate top risks         • Iterate       │
│  • Identify risks             • Detailed plan              • Integrate     │
│  • Candidate architecture     • Executable architecture    • Test          │
│  • Go/No-Go decision          • Go/No-Go decision          • Go/No-Go     │
│                                                                             │
│  TRANSITION                                                          │
│  ────────────                                                           │
│  • Beta test / UAT                                                     │
│  • Deploy / Migrate                                                    │
│  • Train users                                                         │
│  • Go/No-Go (Production)                                               │
│                                                                             │
│  ITERATIONS WITHIN EACH PHASE (time-boxed, typically 2-6 weeks)        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                    RUP DISCIPLINES (Workflows)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ENGINEERING DISCIPLINES:                                                 │
│  • Business Modeling     • Requirements     • Analysis & Design          │
│  • Implementation        • Test             • Deployment                 │
│  • Configuration & Change Mgmt  • Project Management  • Environment     │
│                                                                             │
│  SUPPORTING DISCIPLINES:                                                  │
│  • Environment           • Configuration & Change Mgmt  • Project Mgmt   │
│                                                                             │
│  BEST PRACTICES (RUP Core):                                              │
│  1. Develop Iteratively       2. Manage Requirements                    │
│  3. Use Component Architecture  4. Model Visually (UML)                 │
│  5. Verify Quality Continuously 6. Control Changes                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. BABOK (BUSINESS ANALYSIS BODY OF KNOWLEDGE)

### WHY? (Problem Statement)

**Business analysis bridges business needs and technical solutions.** BABOK = standard for BA practice.

---

### HOW? (BABOK Knowledge Areas)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BABOK v3 KNOWLEDGE AREAS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. BUSINESS ANALYSIS PLANNING & MONITORING                                │
│     • Plan BA approach, governance, performance                            │
│                                                                             │
│  2. ELICITATION & COLLABORATION                                            │
│     • Interviews, workshops, observation, prototypes                       │
│     • Event Storming, Domain Storytelling (Architect collaboration)        │
│                                                                             │
│  3. REQUIREMENTS LIFE CYCLE MANAGEMENT                                     │
│     • Traceability, prioritization, change management, approval            │
│                                                                             │
│  4. STRATEGY ANALYSIS                                                      │
│     • Current state, future state, gap analysis, business case             │
│                                                                             │
│  5. REQUIREMENTS ANALYSIS & DESIGN DEFINITION                              │
│     • Models (process, data, use case), solution options, trade-offs       │
│     • Architect collaboration: non-functional requirements, constraints    │
│                                                                             │
│  6. SOLUTION EVALUATION                                                    │
│     • Measure value, assess limitations, recommend improvements            │
│                                                                             │
│  UNDERLYING COMPETENCIES:                                                  │
│  • Analytical Thinking    • Behavioral Characteristics    • Business Knowledge│
│  • Communication Skills   • Interaction Skills          • Software Knowledge│
│  • Tools & Technology                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Architect-BA Collaboration)

| BA Deliverable | Architect Input |
|----------------|-----------------|
| Business Requirements | Technical feasibility, constraints |
| Functional Requirements | API contracts, data models, integration patterns |
| Non-Functional Requirements | SLOs, architecture patterns, technology choices |
| Solution Options | Trade-off analysis, cost estimation |
| Traceability Matrix | ADR references, architecture decisions |

---

## 5. PRINCE2 (PROJECTS IN CONTROLLED ENVIRONMENTS)

### WHY? (Problem Statement)

**Structured project management with defined roles.** PRINCE2 = process-based, product-focused.

---

### HOW? (PRINCE2 7 Principles, Themes, Processes)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRINCE2 STRUCTURE                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  7 PRINCIPLES:                                                             │
│  1. Continued Business Justification                                       │
│  2. Learn from Experience                                                  │
│  3. Defined Roles and Responsibilities                                     │
│  4. Manage by Stages                                                       │
│  5. Manage by Exception                                                    │
│  6. Focus on Products                                                      │
│  7. Tailor to Suit the Project Environment                                 │
│                                                                             │
│  7 THEMES:                                                                 │
│  • Business Case    • Organization    • Quality    • Plans                │
│  • Risk             • Change          • Progress                          │
│                                                                             │
│  7 PROCESSES:                                                              │
│  1. Starting Up a Project (SU)    2. Directing a Project (DP)            │
│  3. Initiating a Project (IP)     4. Controlling a Stage (CS)            │
│  5. Managing Product Delivery (MP) 6. Managing a Stage Boundary (SB)     │
│  7. Closing a Project (CP)                                                 │
│                                                                             │
│  KEY ROLES:                                                                │
│  • Project Board (Executive, Senior User, Senior Supplier)               │
│  • Project Manager                                                       │
│  • Team Manager                                                          │
│  • Project Assurance                                                     │
│  • Project Support                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. ESB / SOAP

### WHY? (Problem Statement)

**Enterprise integration before microservices.** ESB = centralized integration backbone.

---

### HOW? (ESB vs Modern Alternatives)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ESB ARCHITECTURE                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ESB CAPABILITIES:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Message Routing (Content-based, Header-based)                    │   │
│  │ • Protocol Translation (SOAP ↔ REST, JMS ↔ HTTP, FTP ↔ SFTP)       │   │
│  │ • Data Transformation (XSLT, Mapping, Enrichment)                  │   │
│  │ • Service Orchestration (BPEL)                                     │   │
│  │ • Security (WS-Security, Authentication, Authorization)            │   │
│  │ • Monitoring & Logging                                             │   │
│  │ • Error Handling & Retry                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SOAP (Simple Object Access Protocol):                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • XML-based messaging protocol                                     │   │
│  │ • WS-* Standards (Security, Reliable Messaging, Transactions)      │   │
│  │ • WSDL (Web Services Description Language) for contracts           │   │
│  │ • Heavyweight, verbose, complex                                    │   │
│  │ • Still used in: Banking, Healthcare, Government, Legacy ERP       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODERN REPLACEMENT:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • API Gateway (Kong, Apigee, AWS API GW) — North/South traffic    │   │
│  │ • Service Mesh (Istio, Linkerd) — East/West traffic               │   │
│  │ • Event Streaming (Kafka) — Async integration                      │   │
│  │ • iPaaS (MuleSoft, Dell Boomi, Azure Logic Apps) — SaaS integration│   │
│  │ • GraphQL Federation — Unified API layer                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MIGRATION PATH (Strangler Fig):                                           │
│  1. New services → API Gateway / Service Mesh                              │
│  2. ESB → Proxy legacy endpoints through Gateway                           │
│  3. Migrate orchestration → Saga / Choreography                            │
│  4. Decommission ESB                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. REST (REPRESENTATIONAL STATE TRANSFER)

### WHY? (Problem Statement)

**Web-scale architectural style.** REST = constraints for scalable, evolvable APIs.

---

### HOW? (REST Constraints & Maturity)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REST CONSTRAINTS (Fielding)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. CLIENT-SERVER — Separation of concerns                                │
│  2. STATELESS — No session state on server                                │
│  3. CACHEABLE — Responses explicitly cacheable                            │
│  4. UNIFORM INTERFACE — Resource identification, manipulation via         │
│     representations, self-descriptive messages, HATEOAS                   │
│  5. LAYERED SYSTEM — Intermediaries (proxy, gateway, load balancer)       │
│  6. CODE ON DEMAND (optional) — Executable code (JavaScript, WASM)        │
│                                                                             │
│  RICHARDSON MATURITY MODEL:                                               │
│  ┌──────────┬──────────────────────────────────────────────────────────┐  │
│  │ Level 0  │ RPC over HTTP (SOAP, XML-RPC) — Single endpoint, POST  │  │
│  ├──────────┼──────────────────────────────────────────────────────────┤  │
│  │ Level 1  │ Resources — Multiple endpoints (/orders, /customers)   │  │
│  ├──────────┼──────────────────────────────────────────────────────────┤  │
│  │ Level 2  │ HTTP Verbs — GET/POST/PUT/DELETE, Status codes         │  │
│  ├──────────┼──────────────────────────────────────────────────────────┤  │
│  │ Level 3  │ HATEOAS — Hypermedia links for discoverability         │  │
│  └──────────┴──────────────────────────────────────────────────────────┘  │
│                                                                             │
│  REST BEST PRACTICES:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Nouns for resources (/orders, not /getOrders)                    │   │
│  │ • Plural nouns (/orders/123, not /order/123)                       │   │
│  │ • HTTP verbs for semantics (GET, POST, PUT, PATCH, DELETE)        │   │
│  │ • Proper status codes (200, 201, 204, 400, 401, 404, 409, 429)   │   │
│  │ • Versioning in URL (/v1/orders) or Header (Accept: vnd.api+json) │   │
│  │ • Pagination (cursor-based, not offset)                            │   │
│  │ • Filtering, sorting, field selection (?fields=id,name)           │   │
│  │ • Idempotency keys for POST/PUT                                    │   │
│  │ • RFC 7807 Problem Details for errors                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. GRAPHQL

### WHY? (Problem Statement)

**REST over-fetching/under-fetching.** GraphQL = query language for precise data fetching.

---

### HOW? (GraphQL Core Concepts)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GRAPHQL ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CORE CONCEPTS:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Schema (SDL) — Strongly typed contract                          │   │
│  │ • Query — Read data (GET equivalent)                               │   │
│  │ • Mutation — Write data (POST/PUT/PATCH/DELETE equivalent)         │   │
│  │ • Subscription — Real-time updates (WebSocket)                     │   │
│  │ • Resolvers — Functions that fetch data for fields                 │   │
│  │ • DataLoader — Batching + Caching for N+1 prevention              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SCHEMA EXAMPLE:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ type Order {                                                         │   │
│  │   id: ID!                                                            │   │
│  │   customer: Customer!                                                │   │
│  │   items: [OrderItem!]!                                               │   │
│  │   total: Money!                                                      │   │
│  │   status: OrderStatus!                                               │   │
│  │   createdAt: DateTime!                                               │   │
│  │ }                                                                    │   │
│  │                                                                      │   │
│  │ type Query {                                                         │   │
│  │   order(id: ID!): Order                                              │   │
│  │   orders(customerId: ID!, status: OrderStatus): [Order!]!           │   │
│  │ }                                                                    │   │
│  │                                                                      │   │
│  │ type Mutation {                                                      │   │
│  │   createOrder(input: CreateOrderInput!): Order!                     │   │
│  │   submitOrder(id: ID!): Order!                                       │   │
│  │ }                                                                    │   │
│  │                                                                      │   │
│  │ type Subscription {                                                  │   │
│  │   orderUpdated(customerId: ID!): Order!                              │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GRAPHQL vs REST:                                                          │
│  ┌─────────────────┬─────────────────┬─────────────────────────────────┐  │
│  │ Aspect          │ REST            │ GraphQL                         │  │
│  ├─────────────────┼─────────────────┼─────────────────────────────────┤  │
│  │ Fetching        │ Fixed endpoints │ Client-specified fields         │  │
│  │ Over-fetching   │ Common          │ Eliminated                      │  │
│  │ Under-fetching  │ Multiple calls  │ Single request                  │  │
│  │ Caching         │ HTTP cache      │ Complex (normalized cache)      │  │
│  │ Versioning      │ URL/Header      │ Schema evolution (deprecation)  │  │
│  │ Learning curve  │ Low             │ Higher                          │  │
│  │ Tooling         │ Universal       │ Rich (Apollo, Relay, GraphiQL)  │  │
│  └─────────────────┴─────────────────┴─────────────────────────────────┘  │
│                                                                             │
│  FEDERATION (Micro-frontends / Microservices):                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Gateway composes subgraphs from services                         │   │
│  │ • @key directive for entity resolution                            │   │
│  │ • @external, @requires, @provides for cross-service fields        │   │
│  │ • Tools: Apollo Federation, GraphQL Mesh, Tailcall                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. IAF (INTEGRATED ARCHITECTURE FRAMEWORK)

### WHY? (Problem Statement)

**Enterprise architecture methodology for defense/government.** IAF = comprehensive EA framework.

---

### HOW? (IAF Structure)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IAF (Integrated Architecture Framework)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  IAF CORE PHILOSOPHY: "Architecture as a means to an end"                 │
│                                                                             │
│  FOUR VIEWS (based on ISO/IEC 42010):                                     │
│  ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐ │
│  │ BUSINESS VIEW   │ INFORMATION VIEW│ TECHNOLOGY VIEW │ SERVICE VIEW    │ │
│  │ ─────────────── │ ─────────────── │ ─────────────── │ ─────────────── │ │
│  │ • Strategy      │ • Data models   │ • Infrastructure│ • Capabilities  │ │
│  │ • Processes     │ • Info flows    │ • Platforms     │ • Contracts     │ │
│  │ • Organization  │ • Knowledge     │ • Standards     │ • Interfaces    │ │
│  │ • Governance    │ • Security      │ • Networks      │ • SLAs          │ │
│  └─────────────────┴─────────────────┴─────────────────┴─────────────────┘ │
│                                                                             │
│  CROSS-CUTTING ASPECTS:                                                   │
│  • Security       • Standards      • Compliance      • Governance        │
│  • Risk           • Quality        • Sustainability  • Cost              │
│                                                                             │
│  METHODOLOGY:                                                              │
│  1. Establish Architecture Vision                                         │
│  2. Develop Baseline Architecture                                         │
│  3. Define Target Architecture                                            │
│  4. Perform Gap Analysis                                                  │
│  5. Define Migration Plan                                                 │
│  6. Govern Implementation                                                 │
│  7. Manage Architecture Change                                            │
│                                                                             │
│  WHEN TO USE: Defense, Government, Large enterprises with strict          │
│  governance, compliance, multi-stakeholder environments                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. BPM / BPEL (BUSINESS PROCESS MANAGEMENT / EXECUTION LANGUAGE)

### WHY? (Problem Statement)

**Automate and orchestrate business processes.** BPM = discipline, BPEL = execution language.

---

### HOW? (BPM Lifecycle & BPEL)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BPM LIFECYCLE                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DESIGN → MODEL → EXECUTE → MONITOR → OPTIMIZE                            │
│     │        │        │         │          │                               │
│     ▼        ▼        ▼         ▼          ▼                               │
│  • Process   • BPMN    • BPEL    • KPIs     • Continuous                  │
│    discovery   2.0      engine   dashboards  improvement                   │
│  • Requirements                            (Six Sigma, Lean)               │
│  • Stakeholder                            │                               │
│    alignment                              │                               │
│                                                                             │
│  BPMN 2.0 (Business Process Model and Notation):                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Flow Objects: Events (Start/End/Intermediate), Activities,      │   │
│  │   Gateways (Exclusive/Parallel/Inclusive/Event-based)            │   │
│  │ • Connecting Objects: Sequence Flow, Message Flow, Association   │   │
│  │ • Swimlanes: Pools (participants), Lanes (roles)                │   │
│  │ • Artifacts: Data Objects, Groups, Annotations                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BPEL (Business Process Execution Language):                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • XML-based language for executable processes                     │   │
│  │ • Partner Links (services), Variables, Correlation Sets           │   │
│  │ • Activities: Invoke, Receive, Reply, Assign, Wait, Flow,        │   │
│  │   While, If, Pick, Scope, Compensate                              │   │
│  │ • Fault Handling, Compensation, Transactions                      │   │
│  │ • Used in: Oracle BPEL PM, IBM WebSphere, Apache ODE (legacy)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODERN APPROACH:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • BPMN for modeling (Camunda, Zeebe, Flowable)                    │   │
│  │ • BPEL largely replaced by:                                        │   │
│  │   - Saga/Choreography (Microservices)                              │   │
│  │   - Workflow Engines (Temporal, Cadence, AWS Step Functions)      │   │
│  │   - Stateful Functions (Flink, Akka)                              │   │
│  │   - Low-code (Power Automate, Zapier, n8n)                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. UML (UNIFIED MODELING LANGUAGE)

### WHY? (Problem Statement)

**Standard visual language for software architecture.** UML = communication, not documentation.

---

### HOW? (UML Diagram Types - Essential for Architects)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    UML DIAGRAMS (Essential Subset)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STRUCTURAL DIAGRAMS:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. CLASS DIAGRAM — Static structure (classes, attributes, ops,     │   │
│  │    relationships: inheritance, association, aggregation,          │   │
│  │    composition, dependency, realization)                          │   │
│  │                                                                    │   │
│  │ 2. COMPONENT DIAGRAM — Physical deployment (components,           │   │
│  │    interfaces, ports, dependencies) — Maps to C4 Container       │   │
│  │                                                                    │   │
│  │ 3. DEPLOYMENT DIAGRAM — Nodes (servers, devices), artifacts,     │   │
│  │    execution environments — Maps to C4 Deployment               │   │
│  │                                                                    │   │
│  │ 4. PACKAGE DIAGRAM — Namespaces, dependencies between packages   │   │
│  │                                                                    │   │
│  │ 5. OBJECT DIAGRAM — Instance snapshot of class diagram           │   │
│  │                                                                    │   │
│  │ 6. COMPOSITE STRUCTURE — Internal structure of classifiers       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BEHAVIORAL DIAGRAMS:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 7. USE CASE DIAGRAM — Actors, use cases, relationships            │   │
│  │                                                                    │   │
│  │ 8. SEQUENCE DIAGRAM — Time-ordered message passing (MUST for      │   │
│  │    architects — API flows, distributed transactions)              │   │
│  │                                                                    │   │
│  │ 9. COMMUNICATION DIAGRAM — Object interactions (structure focus)  │   │
│  │                                                                    │   │
│  │ 10. TIMING DIAGRAM — State changes over time                     │   │
│  │                                                                    │   │
│  │ 11. ACTIVITY DIAGRAM — Flow of control/data (business processes)  │   │
│  │                                                                    │   │
│  │ 12. STATE MACHINE DIAGRAM — State transitions (entity lifecycle)  │   │
│  │                                                                    │   │
│  │ 13. INTERACTION OVERVIEW — High-level flow of interactions        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ARCHITECT'S UML TOOLKIT (Minimal Effective Set):                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✅ Sequence Diagram — API flows, distributed transactions          │   │
│  │ ✅ Component Diagram — Service boundaries, interfaces              │   │
│  │ ✅ Deployment Diagram — Infrastructure, topology                   │   │
│  │ ✅ Class Diagram — Domain model (DDD aggregates, entities)        │   │
│  │ ✅ State Machine — Entity lifecycle (Order, Payment, Shipment)    │   │
│  │ ✅ Activity Diagram — Complex business workflows                   │   │
│  │ ❌ Use Case — Better: User Story Mapping / Event Storming         │   │
│  │ ❌ Package/Object/Timing/Communication — Rarely needed             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TOOLING: PlantUML, Mermaid, Structurizr, Enterprise Architect,           │
│  Draw.io, Lucidchart, Miro                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. MESSAGING QUEUES

### WHY? (Problem Statement)

**Async communication backbone.** Queues = decoupling, resilience, scaling.

---

### HOW? (Patterns & Technologies)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGING PATTERNS & TECHNOLOGIES                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MESSAGING PATTERNS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. POINT-TO-POINT (Queue) — One consumer per message              │   │
│  │    • Work distribution, load leveling                              │   │
│  │    • Competing consumers pattern                                   │   │
│  │                                                                    │   │
│  │ 2. PUB/SUB (Topic) — Multiple consumers per message              │   │
│  │    • Event broadcasting, fan-out                                  │   │
│  │    • Durable subscriptions (persistent)                           │   │
│  │                                                                    │   │
│  │ 3. REQUEST-REPLY — Sync over async (correlation ID)              │   │
│  │    • Temporary reply queues                                       │   │
│  │                                                                    │   │
│  │ 4. DEAD LETTER QUEUE — Failed messages for inspection            │   │
│  │    • Poison message handling, retry exhausted                     │   │
│  │                                                                    │   │
│  │ 5. PRIORITY QUEUES — Urgent messages first                       │   │
│  │    • Multiple queues or priority property                         │   │
│  │                                                                    │   │
│  │ 6. DELAYED / SCHEDULED MESSAGES — Future delivery                │   │
│  │    • Retry with backoff, scheduled jobs                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DELIVERY GUARANTEES:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • AT MOST ONCE — Fire and forget (may lose)                        │   │
│  │ • AT LEAST ONCE — Ack + retry (may duplicate) → IDEMPOTENCY!      │   │
│  │ • EXACTLY ONCE — Idempotency + transactions (hard, rare)          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TECHNOLOGY COMPARISON:                                                    │
│  ┌──────────────┬────────────┬──────────┬────────┬────────┬────────────┐  │
│  │ Feature      │ Kafka      │ RabbitMQ │ NATS   │ AWS SQS│ Azure SB   │  │
│  ├──────────────┼────────────┼──────────┼────────┼────────┼────────────┤  │
│  │ Model        │ Log        │ Broker   │ Pub/Sub│ Queue  │ Queue/Topic│  │
│  │ Ordering     │ Per partition│ Per queue│ Per subj│ FIFO  │ Per session│  │
│  │ Replay       │ ✅ Native  │ ❌       │ ❌     │ ❌    │ ❌         │  │
│  │ Throughput   │ Very High  │ High     │ Very High│ High | High       │  │
│  │ Latency      │ Low        │ Low      │ Ultra Low│ Med  │ Med        │  │
│  │ Persistence  │ Disk       │ Disk/Mem │ Mem    │ Disk  │ Disk       │  │
│  │ Protocol     │ TCP        │ AMQP     │ NATS   │ HTTP   │ AMQP/HTTP  │  │
│  │ Multi-tenant │ Topics     │ VHosts   │ Accounts│ Queues │ Namespaces │  │
│  └──────────────┴────────────┴──────────┴────────┴────────┴────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. DATAWAREHOUSE PRINCIPLES

### WHY? (Problem Statement)

**Analytics ≠ Transactions.** OLAP vs OLTP requires different architecture.

---

### HOW? (Data Warehouse Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA WAREHOUSE ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  KIMBALL (Dimensional) vs INMON (Normalized):                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ KIMBALL (Bottom-up):                                              │   │
│  │ • Business process → Dimensional models (Star schema)             │   │
│  │ • Conformed dimensions across data marts                          │   │
│  │ • Fast query performance, business-user friendly                 │   │
│  │                                                                   │   │
│  │ INMON (Top-down):                                               │   │
│  │ • Enterprise data model (3NF) → Data marts                       │   │
│  │ • Single source of truth, normalized                             │   │
│  │ • Slower queries, complex ETL                                    │   │
│  │                                                                   │   │
│  │ MODERN: DATA LAKEHOUSE (Best of both)                           │   │
│  │ • Raw → Bronze → Silver → Gold (Medallion)                      │   │
│  │ • Iceberg/Delta/Hudi for ACID on data lake                      │   │
│  │ • dbt for transformation, SQL for modeling                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  STAR SCHEMA (Kimball):                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Fact Table (center): Measures (additive, semi-additive, non-)   │   │
│  │   Foreign keys to dimensions                                       │   │
│  │   Grain: What does one row represent?                             │   │
│  │                                                                    │   │
│  │ • Dimension Tables: Descriptive attributes                        │   │
│  │   Surrogate keys (not natural keys)                               │   │
│  │   Slowly Changing Dimensions (SCD Type 0/1/2/3/4/6)              │   │
│  │   Conformed dimensions (shared across facts)                      │   │
│  │                                                                    │   │
│  │ • Benefits: Query performance, understandability, BI tool support │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODERN STACK:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Ingestion: Airbyte, Fivetran, Kafka Connect, Debezium           │   │
│  │ • Storage: S3 + Iceberg/Delta/Hudi, Snowflake, BigQuery, Redshift │   │
│  │ • Transform: dbt (SQL), Spark/Flink (complex)                     │   │
│  │ • Orchestration: Airflow, Dagster, Prefect, Temporal              │   │
│  │ • BI: Tableau, Power BI, Looker, Metabase, Superset, Evidence     │   │
│  │ • Reverse ETL: Census, Hightouch                                  │   │
│  │ • Catalog: DataHub, Amundsen, Unity Catalog, Atlas               │   │
│  │ • Quality: Great Expectations, Soda, dbt tests                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. INFRASTRUCTURE AS CODE (IaC)

### WHY? (Problem Statement)

**Manual infrastructure = drift, errors, not reproducible.** IaC = versioned, testable, automated.

---

### HOW? (IaC Tools & Patterns)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE AS CODE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TOOL COMPARISON:                                                          │
│  ┌────────────────┬─────────────┬────────────┬─────────┬────────────────┐  │
│  │ Tool           │ Language    │ Approach   │ State   │ Best For       │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ Terraform      │ HCL         │ Declarative│ Remote  │ Multi-cloud,   │  │
│  │                │             │            │ (S3,    │ mature         │  │
│  │                │             │            │ Consul) │ ecosystem      │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ OpenTofu       │ HCL         │ Declarative│ Remote  │ Terraform fork,│  │
│  │ (fork)         │             │            │         │ MPL license    │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ Pulumi         │ TS, Python, │ Imperative │ Remote  │ App devs,      │  │
│  │                │ Go, C#,     │ (looks     │         │ testing,       │  │
│  │                │ Java, YAML  │ like code) │         │ reuse          │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ AWS CDK        │ TS, Python, │ Imperative │ Cloud   │ AWS-heavy,     │  │
│  │                │ Java, Go    │ (synth →   │Formation│ constructs     │  │
│  │                │             │ CFN)       │         │                │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ Bicep          │ Bicep       │ Declarative│ Azure   │ Azure-native   │  │
│  │                │ (DSL)       │            │ Resource│                │  │
│  │                │             │            │ Manager │                │  │
│  ├────────────────┼─────────────┼────────────┼─────────┼────────────────┤  │
│  │ Crossplane     │ YAML/Go     │ Declarative│ K8s     │ K8s-native,    │  │
│  │                │             │ (CRDs)     │ etcd    │ control plane  │  │
│  └────────────────┴─────────────┴────────────┴─────────┴────────────────┘  │
│                                                                             │
│  IaC BEST PRACTICES:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Version control everything (Git)                                 │   │
│  │ • Modular design (modules, stacks, layers)                         │   │
│  │ • Remote state with locking (S3 + DynamoDB, Consul, Azure Blob)   │   │
│  │ • CI/CD pipeline: fmt → validate → plan → apply                    │   │
│  │ • Policy as Code (OPA, Sentinel, Checkov, tfsec)                  │   │
│  │ • Drift detection (periodic plan, Terraform Cloud)                │   │
│  │ • Secrets management (Vault, AWS Secrets Manager, SOPS)           │   │
│  │ • Testing: Terratest, Kitchen-Terraform, unit tests               │   │
│  │ • Cost estimation (Infracost in PR)                                │   │
│  │ • Documentation (terraform-docs, README per module)               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GITOPS (IaC + Git):                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Git = Single source of truth (desired state)                    │   │
│  │ • ArgoCD / Flux sync cluster to Git                               │   │
│  │ • PR-based changes, audit trail, rollback = git revert            │   │
│  │ • Separation: App code repo vs Infra repo vs Config repo          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 15. CERTIFICATIONS (FOR ARCHITECTS)

### WHY? (Problem Statement)

**Certifications validate knowledge, open doors, demonstrate commitment.**

---

### HOW? (Certification Landscape)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECT CERTIFICATION ROADMAP                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CLOUD ARCHITECT (Major Providers):                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AWS:                                                                │   │
│  │ • Solutions Architect Associate (SAA) — Entry                      │   │
│  │ • Solutions Architect Professional (SAP) — Advanced               │   │
│  │ • Specialty: Security, Networking, Database, ML, SAP              │   │
│  │                                                                     │   │
│  │ AZURE:                                                              │   │
│  │ • AZ-104 (Admin Associate) — Prereq                                │   │
│  │ • AZ-305 (Solutions Architect Expert) — Target                    │   │
│  │ • Specialty: Security, AI, Data, Cosmos DB                        │   │
│  │                                                                     │   │
│  │ GCP:                                                                │   │
│  │ • Cloud Architect Professional — Target                            │   │
│  │ • Specialty: Data Engineer, ML Engineer, Security                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ENTERPRISE ARCHITECTURE:                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • TOGAF 9/10 (Foundation + Certified) — EA standard               │   │
│  │ • ArchiMate 3 (Foundation + Practitioner) — Modeling language    │   │
│  │ • Zachman (awareness) — Framework taxonomy                        │   │
│  │ • FEAF (Federal) — Government                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SOFTWARE ARCHITECTURE:                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • iSAQB CPSA-F (Foundation) — Software architecture               │   │
│  │ • iSAQB CPSA-A (Advanced) — Specialization modules               │   │
│  │ • SEI (CMU) — Architecture-centric engineering                   │   │
│  │ • OMG CABA — Business architecture                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DOMAIN-SPECIFIC:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Security: CISSP, CCSP, CISM, SABSA                                │   │
│  │ • Data: CDMP, AWS Data Analytics, Azure Data Engineer             │   │
│  │ • DevOps: Docker/K8s (CKA, CKAD, CKS), HashiCorp (Terraform)     │   │
│  │ • Agile: PSM, PSPO, SAFe (SAPC, SP, SA), Scrum.org               │   │
│  │ • Project Mgmt: PMP, PRINCE2 Practitioner                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  VENDOR-SPECIFIC (Platform):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Red Hat: RHCA (Certified Architect)                              │   │
│  │ • VMware: VCDX (Design Expert) — Elite                            │   │
│  │ • Oracle: OCM (Master)                                             │   │
│  │ • Salesforce: CTA (Technical Architect) — Elite                   │   │
│  │ • Kubernetes: CKA, CKAD, CKS                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RECOMMENDED PATH FOR SOFTWARE ARCHITECT:                                  │
│  1. Cloud Architect (AWS/Azure/GCP) — Foundation                         │
│  2. TOGAF / iSAQB CPSA-F — EA + Software Architecture                    │
│  3. Specialty (Security, Data, DevOps) — Differentiation                │
│  4. Advanced (CPSA-A, Cloud Pro, SAFe) — Leadership                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PATTERNS & ANTI-PATTERNS

| Framework/Methodology | When to Use | Anti-Pattern | Why Avoid |
|----------------------|-------------|--------------|-----------|
| **PMI/PMBOK** | Formal project governance | Ad-hoc project management | Scope creep, no accountability |
| **ITIL** | IT service delivery/operations | Firefighting culture | Recurring incidents, no improvement |
| **RUP** | Large, complex, regulated systems | Waterfall in disguise | Big design up front, no iteration |
| **BABOK** | Complex requirements elicitation | "Just code it" | Wrong solution, rework |
| **PRINCE2** | Structured projects, clear stages | No project controls | Budget/schedule overruns |
| **ESB/SOAP** | Legacy integration, strict contracts | ESB for everything | Central bottleneck, vendor lock-in |
| **REST** | Public APIs, caching, simplicity | REST for everything (chatty) | Over-fetching, N+1, tight coupling |
| **GraphQL** | Flexible queries, federation | GraphQL for simple CRUD | Complexity, caching difficulty |
| **IAF** | Defense/Gov, strict governance | IAF for startup | Bureaucracy, slow delivery |
| **BPM/BPEL** | Long-running human workflows | BPEL for microservices | Centralized orchestration, coupling |
| **UML** | Architecture communication | UML for everything | Diagram rot, no code alignment |
| **Messaging** | Async, decoupling, resilience | Sync for everything | Cascade failures, temporal coupling |
| **Data Warehouse** | Analytics, reporting | OLTP for analytics | Lock contention, slow queries |
| **IaC** | All environments | ClickOps (manual) | Drift, snowflakes, bus factor |
| **Certifications** | Career validation, knowledge | Certificate collection | Theory without practice |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. ITIL: "How does ITIL relate to modern DevOps/SRE?"

**Answer:**
```
ITIL 4 ALIGNMENT WITH DEVOPS/SRE:

ITIL 4 PRINCIPLES MAP TO DEVOPS:
• "Focus on Value" → Customer-centric delivery
• "Start Where You Are" → Incremental adoption
• "Progress Iteratively with Feedback" → CI/CD, small batches
• "Collaborate and Promote Visibility" → Cross-functional teams
• "Think and Work Holistically" → End-to-end ownership
• "Keep It Simple and Practical" → Minimal viable process
• "Optimize and Automate" → Infrastructure as Code, GitOps

ITIL PRACTICES → SRE MAPPING:
• Incident Management → SRE Incident Response (blameless postmortems)
• Problem Management → Error Budget, Reliability Engineering
• Change Enablement → Progressive Delivery (Canary, Feature Flags)
• Service Level Management → SLO/SLI/Error Budgets
• Availability/Capacity Management → Capacity Planning, Autoscaling
• Release/Deployment Management → GitOps, Progressive Delivery
• Service Continuity → Disaster Recovery, Chaos Engineering
• Knowledge Management → Runbooks, Playbooks, Documentation

KEY DIFFERENCE: ITIL = Process framework, DevOps/SRE = Culture + Engineering
COMPLEMENTARY: ITIL provides governance structure; DevOps/SRE provides speed.
```

---

### 2. REST vs GraphQL: "When would you choose GraphQL over REST?"

**Answer:**
```
CHOOSE GRAPHQL WHEN:
✓ Multiple clients with different data needs (Web, Mobile, Partner)
✓ Complex object graphs with deep nesting (over-fetching in REST)
✓ Rapid frontend iteration (backend changes not needed for new fields)
✓ Federation across microservices (unified API gateway)
✓ Real-time subscriptions needed (WebSocket built-in)
✓ Strong typing + introspection for developer experience

CHOOSE REST WHEN:
✓ Simple CRUD resources (overhead not justified)
✓ Caching critical (HTTP caching, CDN)
✓ Public APIs with diverse consumers (universal HTTP)
✓ Team lacks GraphQL expertise (learning curve)
✓ Regulatory requires simple audit trails (HTTP logs)
✓ Binary file upload/download (multipart better)

HYBRID: GraphQL for complex queries, REST for simple resources + file ops
```

---

### 3. Microservices: "Saga vs Orchestration for distributed transactions?"

**Answer:**
```
SAGA CHOREOGRAPHY (Event-driven):
✓ 2-3 services, simple flow
✓ Services truly autonomous
✓ No central coordinator
✗ Implicit flow (hard to trace)
✗ Cyclic dependency risk
✗ Compensation scattered

SAGA ORCHESTRATION (Central coordinator):
✓ 4+ services, complex/conditional logic
✓ Explicit flow visibility
✓ Centralized compensation
✓ Easier testing (mock participants)
✗ Coordinator becomes smart (domain leak risk)
✗ Single point of failure (mitigate: stateless, replicated)

DECISION TREE:
Flow complexity > 3 steps OR conditional logic? → Orchestration
Services need to evolve independently? → Choreography
Need human-in-the-loop? → Orchestration (state machine)
Team autonomy paramount? → Choreography

HYBRID: Orchestrator for core flow, choreography for notifications
```

---

### 4. IaC: "Terraform vs Pulumi vs CDK — how do you choose?"

**Answer:**
```
TERRAFORM (HCL, Declarative):
✓ Multi-cloud, mature, huge provider ecosystem
✓ Plan/Apply workflow, drift detection
✓ HCL learning curve, limited logic (no loops/conditionals easily)
✓ Best for: Platform teams, multi-cloud, operations-focused

PULUMI (TypeScript/Python/Go/C#, Imperative):
✓ Real programming language (loops, functions, classes, packages)
✓ IDE support, testing, linting, reuse
✓ Same providers as Terraform (bridged)
✓ Best for: App developers, complex logic, unit testing IaC

AWS CDK (TypeScript/Python, Imperative → CloudFormation):
✓ Native AWS constructs (L2/L3), best AWS experience
✓ Synthesizes to CloudFormation (CFN drift issues)
✓ Best for: AWS-only, CDK constructs accelerate development

CROSSPLANE (Kubernetes-native, Declarative):
✓ Manages external resources via K8s CRDs
✓ GitOps native, control plane = K8s
✓ Best for: K8s-platform teams, self-service infra

DECISION:
Team skills + Cloud strategy + Testing needs
• Ops team, multi-cloud → Terraform
• Dev team, complex logic → Pulumi
• AWS-only, speed → CDK
• K8s platform → Crossplane
```

---

### 5. Messaging: "Kafka vs RabbitMQ vs NATS — when to use each?"

**Answer:**
```
KAFKA (Distributed Log):
✓ High throughput (millions/sec), replay, ordering per partition
✓ Event sourcing, audit trails, CDC, stream processing (KSQL, Flink)
✓ Long retention (days/weeks), consumer groups
✗ Operational complexity (ZooKeeper/KRaft, brokers, partitions)
✗ Higher latency (~5ms) vs queues

RABBITMQ (Traditional Broker):
✓ Flexible routing (exchanges, bindings), priority queues, TTL
✓ Lower latency (~1-2ms), per-message ack, dead lettering
✓ Mature, flexible protocols (AMQP, MQTT, STOMP)
✗ No native replay, horizontal scaling harder
✗ Memory pressure with large queues

NATS (Lightweight Pub/Sub):
✓ Ultra-low latency (~0.5ms), JetStream for persistence
✓ Simple, fast, subject-based routing, wildcards
✓ JetStream: streams, consumer groups, ack, replay
✗ Younger ecosystem, fewer integrations
✗ Less mature tooling vs Kafka/RabbitMQ

DECISION:
Event sourcing / audit / replay / high throughput → Kafka
Traditional queuing / complex routing / low latency → RabbitMQ
Ultra-low latency / simple pub-sub / cloud-native → NATS
```

---

### 6. Architecture Frameworks: "TOGAF vs Zachman vs ArchiMate — differences?"

**Answer:**
```
TOGAF (The Open Group Architecture Framework):
• PROCESS (ADM - Architecture Development Method)
• Phases: Preliminary → A (Vision) → B (Business) → C (Info Systems) → D (Tech) → E (Opportunities) → F (Migration) → G (Gov) → H (Change)
• Deliverables: Principles, Building Blocks, Gap Analysis, Roadmap
• Best for: Large enterprises needing governance process

ZACHMAN FRAMEWORK:
• TAXONOMY (Classification Matrix)
• 6×6 Matrix: What/How/Where/Who/When/Why × Planner/Owner/Designer/Builder/Subcontractor/User
• No process, just "fill the cells"
• Best for: Completeness checking, classification, ontology

ARCHIMATE:
• MODELING LANGUAGE (Notation)
• Layers: Business, Application, Technology, Physical, Motivation, Implementation
• Relationships: Composition, Aggregation, Assignment, Realization, Serving, Access, Influence, Association
• Tool support: Archi, Sparx EA, BiZZdesign, MEGA
• Best for: Visualization, communication, tool interoperability

USAGE TOGETHER:
TOGAF (Process) + ArchiMate (Modeling) + Zachman (Completeness check)
```

---

### 7. Data Architecture: "Kimball vs Inmon vs Data Mesh — when to use?"

**Answer:**
```
KIMBALL (Dimensional/Bottom-up):
• Business process → Star schema data marts → Conformed dimensions
• Fast queries, BI-friendly, incremental
• Best for: Departmental analytics, self-service BI

INMON (Normalized/Top-down):
• Enterprise 3NF model → Data marts
• Single source of truth, normalized
• Best for: Regulatory, master data management, enterprise reporting

DATA MESH (Decentralized, Domain-oriented):
• Domain-oriented ownership (data as product)
• Self-serve data infrastructure
• Federated governance
• Best for: Large orgs, diverse domains, scale

MODERN RECOMMENDATION:
DATA LAKEHOUSE (Unified):
• Raw (Bronze) → Cleaned (Silver) → Business (Gold)
• Iceberg/Delta/Hudi for ACID on object storage
• dbt for SQL transformations, Spark for complex
• Supports both BI (Star) and ML (Raw)

DECISION:
Small/medium → Kimball + Lakehouse
Large regulated → Inmon + Lakehouse
Large decentralized → Data Mesh + Lakehouse
```

---

### 8. BPM: "How do you model a long-running business process with compensation?"

**Answer:"
```
BPMN 2.0 MODELING FOR SAGA:

1. PROCESS STRUCTURE:
   Pool: Order Fulfillment
   Lanes: Customer, Order Service, Payment, Inventory, Shipping

2. COMPENSATION BOUNDARY EVENTS:
   - Attach to activity: "Reserve Payment"
   - Compensation activity: "Refund Payment"
   - Trigger: Intermediate Event (Error/Escalation) on later step

3. TRANSACTION SUBPROCESS (Double-lined):
   - Contains: Reserve Payment → Reserve Inventory → Schedule Shipment
   - Success: End Event (thin) → Confirm Order
   - Failure: Cancel End Event (thick) → Triggers compensation

4. COMPENSATION FLOW:
   On "Payment Failed":
   → Cancel "Reserve Inventory" (compensate)
   → Cancel "Reserve Payment" (compensate)
   → Send "Order Cancelled" notification

5. CORRELATION:
   - Process Instance ID = Order ID
   - Message correlation on Order ID for async responses

IMPLEMENTATION:
- Camunda/Zeebe/Flowable execute BPMN directly
- Or translate to Saga Orchestrator (state machine)
- BPEL largely replaced by BPMN + Engine
```

---

### 9. UML: "Which UML diagrams do you actually use as an architect?"

**Answer:**
```
MINIMAL EFFECTIVE SET (5 diagrams):

1. SEQUENCE DIAGRAM — #1 for architects
   • API flows, distributed transactions, error paths
   • Lifelines: Client, API GW, Services, DB, Kafka
   • Async: <<create>>, <<destroy>>, found messages

2. COMPONENT DIAGRAM — Service boundaries
   • Components: Services, DBs, Queues, External systems
   • Interfaces: Provided (ball) / Required (socket)
   • Maps to C4 Container

3. DEPLOYMENT DIAGRAM — Infrastructure
   • Nodes: K8s clusters, VMs, Serverless, Databases
   • Artifacts: Containers, ConfigMaps, Secrets
   • Networks: VPCs, Subnets, Peering, Service Mesh

4. CLASS DIAGRAM — Domain model (DDD)
   • Aggregates (root), Entities, Value Objects
   • Associations, Multiplicity, Composition
   • Domain Events, Repository Interfaces

5. STATE MACHINE — Entity lifecycle
   • Order: Draft → Submitted → Paid → Shipped → Delivered
   • Transitions: Events (Submit, Pay, Ship, Deliver)
   • Guards: [paymentSuccess], [inventoryAvailable]

RARELY USE:
❌ Use Case → User Story Mapping better
❌ Package/Object/Timing/Communication/Interaction Overview → Overkill
❌ Activity → BPMN better for business processes
```

---

### 10. ESB: "How do you migrate from ESB to modern architecture?"

**Answer:**
```
STRANGLER FIG MIGRATION:

PHASE 1: API GATEWAY IN FRONT
• Deploy Kong/Apigee/AWS API GW in front of ESB
• Route new consumers to Gateway, not ESB directly
• Implement auth, rate limit, logging at Gateway

PHASE 2: EXTRACT ORCHESTRATION
• Identify BPEL orchestrations
• For each: 
  - Simple → Saga Choreography (events)
  - Complex → Saga Orchestrator (state machine: Camunda/Zeebe/Temporal)
• New services own their workflows

PHASE 3: REPLACE TRANSFORMATIONS
• XSLT/Mapping in ESB → 
  - Service-owned serializers (JSON/Protobuf)
  - Anti-corruption layers at boundaries
• Schema Registry (Confluent/Apicurio) for contracts

PHASE 4: REPLACE ROUTING
• ESB content-based routing → 
  - API Gateway path/host/header routing
  - Service Mesh (Istio) for east-west
  - Kafka topic routing for events

PHASE 5: DECOMMISSION
• Run parallel (shadow traffic) for validation
• Monitor: Latency, error rate, functionality
• Cutover: DNS switch, remove ESB
• Archive ESB configs for reference

KEY PRINCIPLE: One integration at a time, never big bang.
```

---

### 11. SOAP: "When would you still use SOAP in 2024?"

**Answer:**
```
LEGITIMATE SOAP USE CASES (2024):

1. LEGACY INTEGRATION (Cannot change):
   • Mainframe banking (ISO 20022, SWIFT)
   • Healthcare (HL7 v3, FHIR R4 still has SOAP bindings)
   • Government (Tax, Customs, Regulatory reporting)

2. WS-* STANDARDS REQUIRED:
   • WS-Security (XML Encryption, Signature, SAML tokens)
   • WS-ReliableMessaging (Guaranteed delivery, ordering)
   • WS-AtomicTransaction (Distributed transactions)
   • WS-Policy (Security assertions)

3. CONTRACT-FIRST WITH STRICT VALIDATION:
   • WSDL + XSD = Unambiguous contract
   • Code generation in 20+ languages
   • Schema validation at boundary

4. ENTERPRISE PLATFORMS:
   • SAP (SOAP primary, OData secondary)
   • Oracle EBS, Salesforce (SOAP API still supported)
   • IBM WebSphere, Oracle SOA Suite

MODERN APPROACH:
• SOAP Adapter at boundary (API Gateway / Service Mesh)
• Internal: REST/gRPC/Async
• Anti-corruption layer translates SOAP ↔ Internal model
• Never expose SOAP directly to new consumers
```

---

### 12. Certifications: "Which certifications matter for a Software Architect?"

**Answer:**
```
TIER 1 (Foundation - Get First):
☐ Cloud Architect (AWS SAA → SAP, or AZ-305, or GCP Architect)
☐ TOGAF 9/10 Foundation + Certified
☐ iSAQB CPSA-F (Software Architecture Foundation)

TIER 2 (Specialization - Differentiate):
☐ Cloud Specialty (Security, Data, ML, Networking)
☐ iSAQB CPSA-A (Advanced: DDD, Security, Performance, etc.)
☐ SAFe (SP, SASM, SA) if org uses SAFe
☐ Security: CISSP or CCSP (if security focus)

TIER 3 (Leadership/Elite):
☐ AWS Solutions Architect Professional / Azure Expert / GCP Architect
☐ TOGAF Certified + ArchiMate Practitioner
☐ iSAQB CPSA-A (multiple modules)
☐ Vendor Elite: VCDX (VMware), CTA (Salesforce), RHCA (Red Hat)

WHAT EMPLOYERS VALUE:
1. Cloud Architect (hands-on cloud) — Mandatory for most roles
2. TOGAF / iSAQB — Shows EA + Software Architecture breadth
3. Specialty certs — Proves depth in domain
4. Vendor elites — Rare, high prestige

ANTI-PATTERN: Collecting certs without practice.
RULE: One cert per year max, apply learning to current role.
```

---

### 13. Data Mesh: "Explain Data Mesh principles and when to adopt."

**Answer:"
```
DATA MESH 4 PRINCIPLES (Zhamak Dehghani):

1. DOMAIN-ORIENTED OWNERSHIP
   • Data owned by domain team (not central data team)
   • Team that produces data owns its quality, schema, SLAs
   • "Data as a Product" mindset

2. DATA AS A PRODUCT
   • Discoverable, Addressable, Trustworthy, Self-describing
   • Schema, Documentation, SLAs, Samples, Lineage
   • Versioned, Deprecation policy, Support channel

3. SELF-SERVE DATA PLATFORM
   • Platform team builds infrastructure (not pipelines)
   • Domain teams self-serve: ingest, transform, serve
   • Tools: dbt, Airflow, Kafka, Iceberg, Catalog, Quality

4. FEDERATED GOVERNANCE
   • Global policies (security, privacy, standards)
   • Domain autonomy within guardrails
   • Computational governance (policy as code)

WHEN TO ADOPT:
✓ 50+ data sources, 10+ domains
✓ Central team bottleneck (months for new pipeline)
✓ Domain expertise required for data quality
✓ Organizational maturity (DevOps, Cloud, IaC)

WHEN NOT TO ADOPT:
✗ Small team (< 10 data engineers)
✗ Simple analytics needs
✗ No domain ownership culture
✗ Regulatory requires central control

IMPLEMENTATION PATH:
1. Pilot: 1-2 domains, build platform capabilities
2. Platform: Self-serve infra (ingestion, transform, catalog)
3. Governance: Computational policies, data contracts
4. Scale: Onboard domains incrementally
```

---

### 14. Architecture Decision Records: "How do you implement ADRs in a team?"

**Answer:"
```
ADR IMPLEMENTATION PLAYBOOK:

1. TEMPLATE (Markdown in /docs/adr/):
# ADR-XXXX: Title
## Status: Proposed | Accepted | Rejected | Superseded
## Context: What forces drove this?
## Decision: What are we doing?
## Consequences: Positive, Negative, Risks
## Alternatives: What else considered?
## Related: ADR-XXXX, RFC-XXXX

2. PROCESS:
• Any structural decision → ADR (in PR with code)
• Author writes draft → 3-day async review → Decider (Architect) decides
• Accepted → Merge → Communicate in #arch-decisions
• Superseded → New ADR links to old, old status updated

3. TOOLING:
• ADR Tools (adr-tools, adr-log) for numbering
• GitHub Actions: Validate template, check links
• Index page auto-generated (adr-index)

4. GOVERNANCE:
• Quarterly review: Any superseded? Any missing?
• Architecture Board: Reviews controversial ADRs
• Exception process: Time-boxed, documented in ADR

5. METRICS:
• % decisions with ADR (target 100%)
• Time from proposal to decision (< 1 week)
• Supersession rate (healthy = some)
• Team awareness (survey quarterly)
```

---

### 15. Microservices Decomposition: "How do you identify service boundaries?"

**Answer:"
```
SERVICE DECOMPOSITION FRAMEWORK:

1. BUSINESS CAPABILITY MAPPING:
   • Value stream map → Business capabilities
   • Each capability = Candidate services
   • Capabilities stable, processes change

2. DDD BOUNDED CONTEXTS:
   • Ubiquitous Language differences
   • Team ownership boundaries
   • Data ownership (Who owns Customer?)
   • Transactional boundaries (What MUST be ACID?)

3. ANALYZE COUPLING:
   • High cohesion within, low coupling between
   • Data coupling (shared tables = bad)
   • Temporal coupling (must deploy together = bad)
   • Behavioral coupling (chatty calls = bad)

4. VALIDATE WITH TEAM TOPOLOGY:
   • One team per service (or service per team)
   • Team has all skills to own end-to-end
   • Cognitive load manageable (< 7 services/team)

ANTI-PATTERNS:
❌ Nanoservices (too small, chatty)
❌ Shared database (defeats independence)
❌ Distributed monolith (deploy together, fail together)
❌ Anemic services (no behavior, just CRUD)

EVOLUTION:
Start Modular Monolith → Extract when pain > cost
Pain signals: Team friction, scaling needs, tech heterogeneity, deploy coupling
```

---

## QUICK REFERENCE: FRAMEWORK SELECTION GUIDE

| Need | Framework | Alternative |
|------|-----------|-------------|
| Project Governance | PMI/PMBOK | PRINCE2 |
| IT Service Management | ITIL 4 | COBIT |
| Large System Development | RUP | Scaled Agile (SAFe/LeSS) |
| Requirements Analysis | BABOK | Event Storming + User Story Mapping |
| Structured Project Delivery | PRINCE2 | PMI |
| Legacy Integration | ESB/SOAP | API Gateway + Strangler Fig |
| Public/Web APIs | REST | GraphQL, gRPC |
| Flexible Queries/Federation | GraphQL | REST + BFF |
| Defense/Gov EA | IAF | TOGAF |
| Business Process Automation | BPMN + Camunda/Zeebe | Temporal/Step Functions |
| Architecture Communication | UML (Sequence, Component, Deployment) | C4 + Mermaid |
| Async Communication | Kafka (events) / RabbitMQ (commands) | NATS, Cloud Pub/Sub |
| Analytics Platform | Data Lakehouse (Iceberg/Delta + dbt) | Traditional DW |
| Infrastructure Automation | Terraform / Pulumi / CDK | Crossplane, CloudFormation |
| Career Validation | Cloud Architect + TOGAF/iSAQB | Vendor specialties |

---

*This document covers all topics from the Frameworks section of the roadmap.sh Software Architect roadmap with exam-style depth.*