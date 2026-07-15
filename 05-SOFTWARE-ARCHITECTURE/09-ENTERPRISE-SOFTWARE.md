# ENTERPRISE SOFTWARE
## Exam-Style Study Notes

**Roadmap Section:** Enterprise Software → MS Dynamics, SAP ERP/HANA, EMC DMS, IBM BPM, Salesforce

---

## 1. MICROSOFT DYNAMICS 365

### WHY? (Problem Statement)

**Organizations need unified CRM + ERP.** Dynamics 365 = modular business applications on Microsoft cloud (Dataverse + Power Platform).

---

### HOW? (Dynamics 365 Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DYNAMICS 365 ECOSYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  APPLICATION MODULES:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ CRM (Customer Engagement):                                         │   │
│  │ • Sales — Pipeline, forecasting, AI insights                       │   │
│  │ • Customer Service — Omnichannel, knowledge, SLAs                 │   │
│  │ • Marketing — Journeys, segmentation, events                       │   │
│  │ • Field Service — Scheduling, IoT, remote assist                  │   │
│  │ • Project Operations — PSA, resource management                   │   │
│  │                                                                     │   │
│  │ ERP (Finance & Operations):                                        │   │
│  │ • Finance — GL, AP/AR, fixed assets, budgeting                    │   │
│  │ • Supply Chain — Inventory, warehouse, procurement, manufacturing │   │
│  │ • Commerce — Unified commerce, POS, e-commerce                    │   │
│  │ • Human Resources — Core HR, talent, payroll, benefits            │   │
│  │ • Project Operations — Project-based ERP                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PLATFORM (Power Platform):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Power Apps — Low-code/no-code apps (Canvas, Model-driven)       │   │
│  │ • Power Automate — Workflow automation (Cloud, Desktop, Process)  │   │
│  │ • Power BI — Analytics, dashboards, paginated reports             │   │
│  │ • Power Virtual Agents — Chatbots                                  │   │
│  │ • Power Pages — External-facing websites                           │   │
│  │ • Dataverse — Data platform (tables, security, logic)             │   │
│  │ • AI Builder — Prebuilt/custom AI models                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ARCHITECTURE:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Multi-tenant SaaS (Azure)                                        │   │
│  │ • Dataverse = Azure SQL + metadata + business logic               │   │
│  │ • Extensibility: Plugins (C#), JavaScript, Power FX, OData/Web API│   │
│  │ • Integration: Dual-write (F&O ↔ Dataverse), Virtual tables       │   │
│  │ • ALM: Solutions (managed/unmanaged), Pipelines, Git              │   │
│  │ • Security: Role-based, field-level, team/ownership, Azure AD     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Integration Patterns)

| Pattern | Use Case | Technology |
|---------|----------|------------|
| **Dual-write** | F&O ↔ Dataverse sync | Built-in, near real-time |
| **Virtual Tables** | External data in Dataverse | OData, SQL, custom |
| **Power Automate** | Workflow integration | 1000+ connectors |
| **OData/Web API** | Custom integration | REST, batch, $filter |
| **Message Queue** | High-volume async | Azure Service Bus, Event Hub |
| **BYOD** | Analytics export | Azure SQL, Data Lake |

---

## 2. SAP ERP / S/4HANA

### WHY? (Problem Statement)

**Enterprise standard for complex business processes.** S/4HANA = next-gen ERP on HANA in-memory database.

---

### HOW? (S/4HANA Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SAP S/4HANA ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HANA DATABASE (In-Memory):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Columnar + Row store                                             │   │
│  │ • Compression (5-10x)                                              │   │
│  │ • Parallel processing (multi-core)                                 │   │
│  │ • ACID, MVCC                                                       │   │
│  │ • Advanced Analytics (PAL, AFL)                                    │   │
│  │ • Tiered storage (Hot/Warm/Cold)                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  APPLICATION LAYER (ABAP + CDS):                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • ABAP RESTful Programming Model (RAP)                             │   │
│  │ • Core Data Services (CDS) — Semantic data models                 │   │
│  │ • OData V2/V4 services                                             │   │
│  │ • Business Events (Event-driven)                                   │   │
│  │ • Embedded Analytics (CDS Views + SAC)                            │   │
│  │ • Side-by-side Extensibility (BTP, Kyma, FaaS)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODULES (Lines of Business):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Finance (FI) — GL, AP/AR, Asset, Controlling (CO)               │   │
│  │ • Logistics (MM, SD, PP, QM, PM, WM, EWM)                         │   │
│  │ • Human Capital Management (HCM)                                   │   │
│  │ • Project Systems (PS)                                             │   │
│  │ • Industry Solutions (IS-Oil, IS-Retail, IS-Auto, etc.)           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INTEGRATION (SAP Integration Suite / BTP):                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • API Management (Gateway, Developer Portal)                       │   │
│  │ • Cloud Integration (iFlows, Adapters: OData, IDoc, SOAP, REST)   │   │
│  │ • Event Mesh (Async, Pub/Sub)                                      │   │
│  │ • Open Connectors (Prebuilt 3rd party)                             │   │
│  │ • Business Events (S/4HANA → Event Mesh)                           │   │
│  │ • IDoc / ALE (Legacy)                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPLOYMENT OPTIONS:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Public Cloud (SAP managed, quarterly updates)                    │   │
│  │ • Private Cloud (Customer/Partner managed, annual updates)         │   │
│  │ • On-Premise (Customer managed, annual updates)                    │   │
│  │ • RISE with SAP (Bundle: Infra + ERP + BTP)                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Technical Concepts)

| Concept | Description |
|---------|-------------|
| **CDS Views** | Semantic data models, push-down to HANA, annotations for UI/Analytics |
| **RAP** | ABAP RESTful Programming Model — OData services, draft handling, actions |
| **BTP** | Business Technology Platform — Extension, Integration, Analytics, App Dev |
| **Fiori** | UX design system — Role-based, responsive, Launchpad |
| **Embedded Analytics** | Real-time operational reporting on transactional data |
| **Side-by-side Extensibility** | Extend on BTP (CAP, Kyma, FaaS) without modifying core |
| **Key User Extensibility** | Low-code: custom fields, logic, UI via Fiori |

---

## 3. EMC DOCUMENTUM (DMS)

### WHY? (Problem Statement)

**Enterprise content management at scale.** Documentum = secure, compliant, scalable document management.

---

### HOW? (Documentum Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DOCUMENTUM (OpenText) ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CORE COMPONENTS:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • CONTENT SERVER — Core repository (Java, PostgreSQL/Oracle)       │   │
│  │ • DOCBROKER — Connection broker (load balancing, failover)         │   │
│  │ • INDEX SERVER (xPlore) — Full-text search (Solr/Elastic)          │   │
│  │ • CONTENT STORAGE — File systems, S3, Centera, cloud tiers         │   │
│  │ • TRUSTED CONTENT SERVICES — Compliance, retention, legal hold     │   │
│  │ • PROCESS ENGINE — BPM workflow (legacy)                           │   │
│  │ • APPLICATION CONNECTORS — SAP, Salesforce, Microsoft, Email       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA MODEL:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • OBJECT TYPES — dm_document, dm_folder, dm_sysobject, custom      │   │
│  │ • ATTRIBUTES — Single, repeating, encrypted                         │   │
│  │ • RELATIONSHIPS — Parent/child, annotations, versions              │   │
│  │ • ASPECTS — Cross-cutting behavior (versionable, auditable)        │   │
│  │ • ACLs — Access control (user, group, role, dynamic)               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODERN CAPABILITIES:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • REST API (JSON, OpenAPI)                                         │   │
│  │ • Docker/Kubernetes deployment (CS, xPlore, D2)                    │   │
│  │ • D2 (Web UI) — Modern, responsive, configurable                   │   │
│  │ • Smart View — Embedded viewer (Office, PDF, CAD)                  │   │
│  │ • InfoArchive — Long-term retention, compliance                    │   │
│  │ • Cloud-native (AWS, Azure, GCP)                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INTEGRATION PATTERNS:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • REST/JSON — Primary API                                          │   │
│  │ • CMIS 1.1 — Standard CMIS interface                               │   │
│  │ • DFS (Documentum Foundation Services) — SOAP (legacy)             │   │
│  │ • Event Server — Real-time notifications (JMS, Kafka)              │   │
│  │ • Application Connectors — Prebuilt for major apps                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Versioning & Retention)

| Feature | Implementation |
|---------|----------------|
| **Versioning** | Implicit (checkin/checkout) or Explicit (explicit version labels), Major/Minor, Branching |
| **Retention** | Policies (time-based, event-based), Legal Hold, Disposition workflows |
| **Archive** | InfoArchive, Cloud tiering (S3 Glacier, Azure Archive) |
| **Search** | xPlore (Solr/Elastic), faceted, metadata + full-text |

---

## 4. IBM BPM / BUSINESS AUTOMATION WORKFLOW

### WHY? (Problem Statement)

**Process automation for complex, long-running workflows.** IBM BAW = BPM + Case Management + Decision Management + RPA.

---

### HOW? (IBM BAW Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IBM BUSINESS AUTOMATION WORKFLOW                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  COMPONENTS:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PROCESS SERVER — BPMN 2.0 engine (WebSphere Liberty)             │   │
│  │ • PROCESS CENTER — Design-time repository, governance              │   │
│  │ • PERFORMANCE DATA WAREHOUSE — Analytics, KPIs                     │   │
│  │ • CASE MANAGEMENT — Ad-hoc, knowledge worker processes             │   │
│  │ • DECISION SERVER (ODM) — Business rules (DMN, DRL)                │   │
│  │ • RPA (Watson Orchestrate) — Bot automation                        │   │
│  │ • CONTENT NAVIGATOR — Content integration (FileNet, CMIS)          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PROCESS MODELING (BPMN 2.0):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Service Tasks (Java, Web Service, REST, Script)                  │   │
│  │ • Human Tasks (Coach UI, Coach Views, Responsive)                  │   │
│  │ • Events (Start, Intermediate, End, Message, Timer, Error)         │   │
│  │ • Gateways (Exclusive, Parallel, Inclusive, Event-based)           │   │
│  │ • Subprocesses (Embedded, Call Activity, Event Subprocess)         │   │
│  │ • Compensation (Saga pattern for rollback)                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INTEGRATION:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Service Integration (SOAP, REST, JMS, MQ, Database, File)        │   │
│  │ • Business Objects (XML, JSON, XSD mapping)                        │   │
│  │ • External Implementations (Java, JavaScript)                      │   │
│  │ • Event Emission (Kafka, MQ, HTTP)                                 │   │
│  │ • API Gateway (IBM API Connect)                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPLOYMENT:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • On-premises (WebSphere, DB2/Oracle)                              │   │
│  │ • Cloud Pak for Business Automation (OpenShift/K8s)                │   │
│  │ • SaaS (IBM Cloud)                                                 │   │
│  │ • Hybrid                                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Modern Alternatives)

| Legacy | Modern Alternative |
|--------|-------------------|
| IBM BPM (on-prem) | Cloud Pak for BA (OpenShift) |
| WebSphere | Liberty / Quarkus / Spring Boot |
| Process Center | Git + CI/CD + BPMN in Git |
| Coach UI | React/Angular + BPMN.js |
| ODM (Rules) | DMN (Camunda, Drools, DMN.js) |
| Custom Java Services | Spring Boot / Node.js Microservices |

---

## 5. SALESFORCE

### WHY? (Problem Statement)

**World's #1 CRM platform.** Salesforce = multi-tenant SaaS with extensive customization, ecosystem, and AI.

---

### HOW() (Salesforce Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SALESFORCE PLATFORM ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CORE CLOUDS:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Sales Cloud — Lead, Opportunity, Account, Contact, Forecasting   │   │
│  │ • Service Cloud — Case, Knowledge, Omni-Channel, Field Service     │   │
│  │ • Marketing Cloud — Journey Builder, Email, Mobile, Advertising    │   │
│  │ • Commerce Cloud — B2C/B2B, Headless, Order Management             │   │
│  │ • Platform (Force.com) — Custom apps, AppExchange                  │   │
│  │ • Industries — Health, Financial Services, Manufacturing, etc.     │   │
│  │ • Data Cloud (CDP) — Unified customer profile                      │   │
│  │ • Einstein AI — Predictive, Generative, Conversational            │   │
│  │ • Slack — Digital HQ, integration                                   │   │
│  │ • Tableau — Analytics                                              │   │
│  │ • MuleSoft — Integration (Anypoint Platform)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PLATFORM ARCHITECTURE (Multi-tenant):                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • METADATA-DRIVEN — Objects, Fields, Layouts, Flows as metadata    │   │
│  │ • MULTI-TENANCY — Shared infrastructure, logical isolation         │   │
│  │ • GOVERNOR LIMITS — SOQL, DML, CPU, Heap, API calls per transaction│   │
│  │ • SHARING MODEL — OWD, Role Hierarchy, Sharing Rules, Teams        │   │
│  │ • SECURITY — Profiles, Permission Sets, Field-level, Session       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEVELOPMENT (Declarative + Programmatic):                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ DECLARATIVE (No-code/Low-code):                                    │   │
│  │ • Objects, Fields, Relationships                                   │   │
│  │ • Page Layouts, Record Pages, Dynamic Forms                        │   │
│  │ • Flow (Record-triggered, Autolaunched, Screen, Scheduled)         │   │
│  │ • Approval Processes, Validation Rules, Assignment Rules           │   │
│  │ • Reports, Dashboards, List Views                                  │   │
│  │                                                                    │   │
│  │ PROGRAMMATIC (Code):                                               │   │
│  │ • Apex (Java-like, strongly typed, triggers, classes, batch)       │   │
│  │ • Lightning Web Components (LWC) — Modern, standards-based         │   │
│  │ • Aura Components (legacy)                                         │   │
│  │ • Visualforce (legacy, PDF rendering)                              │   │
│  │ • SOQL/SOSL — Query/Search languages                               │   │
│  │ • Async Apex (Future, Queueable, Batch, Schedulable)             │   │
│  │ • Platform Events / Change Data Capture (CDC)                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INTEGRATION:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • REST/SOAP APIs (Enterprise, Partner, Tooling, Metadata, Bulk)    │   │
│  │ • Platform Events / CDC — Event-driven (Kafka-like)                │   │
│  │ • External Services (OpenAPI → Flow invocable)                     │   │
│  │ • MuleSoft Anypoint — API-led connectivity                         │   │
│  │ • Heroku Connect — Postgres sync                                   │   │
│  │ • Salesforce Connect — OData, Cross-org, Custom adapters           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEVOPS / ALM:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Source-Driven Development (SFDX, VS Code)                        │   │
│  │ • Scratch Orgs (Ephemeral, CI/CD)                                  │   │
│  │ • Sandboxes (Developer, Pro, Full, Partial)                        │   │
│  │ • Unlocked Packages (Modular, versioned, dependencies)             │   │
│  │ • CI/CD (GitHub Actions, GitLab, Azure DevOps, Copado, Gearset)   │   │
│  │ • Code Coverage (75% minimum), Static Analysis (PMD, CodeAnalyzer) │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Technical Limits & Patterns)

| Limit | Value | Workaround |
|-------|-------|------------|
| **SOQL Queries** | 100 sync / 200 async | Bulkify, selective queries |
| **DML Statements** | 150 sync / 300 async | Bulk DML, composite API |
| **CPU Time** | 10s sync / 60s async | Optimize loops, async |
| **Heap Size** | 6MB sync / 12MB async | Limit collections, transient |
| **Callout Timeout** | 120s max | Async (Continuation) |
| **API Calls** | Varies by edition/licenses | Bulk API, Composite API |

| Pattern | Implementation |
|---------|---------------|
| **Trigger Framework** | Handler classes, recursion guard, bulk-safe |
| **Selector Pattern** | Separate query logic from triggers |
| **Service Layer** | Business logic in services, not triggers |
| **Domain Layer** | Object-oriented wrapper for sObjects |
| **Unit of Work** | Transaction management, rollback |
| **Async Patterns** | Queueable chaining, Platform Events, CDC |

---

## ENTERPRISE SOFTWARE COMPARISON

| Dimension | Dynamics 365 | SAP S/4HANA | Documentum | IBM BAW | Salesforce |
|-----------|--------------|-------------|------------|---------|------------|
| **Primary Focus** | CRM + ERP | ERP | ECM | BPM | CRM |
| **Deployment** | SaaS (Azure) | Cloud/On-prem | Cloud/On-prem | Cloud/On-prem | SaaS |
| **Extensibility** | Power Platform, Plugins | ABAP, BTP, CAP | REST, CMIS | Java, JS, Services | Apex, LWC, Flow |
| **Integration** | Power Automate, Dual-write | Integration Suite, BTP | REST, CMIS | Services, MQ, Events | MuleSoft, APIs, Events |
| **Low-code** | Power Apps, Flow | SAP Build, Fiori | D2, xCP | Coaches, Flow | Flow, LWC |
| **Analytics** | Power BI | SAC, Embedded | xPlore | PDW | Tableau, CRM Analytics |
| **AI** | Copilot, Einstein | Joule, AI Core | xPlore AI | Watson, ODM | Einstein, Data Cloud |
| **Licensing** | Per user/app | Named user/Engine | Named user/CPU | PVU/Authorized | Per user/edition |
| **Best For** | Microsoft shops, Mid-market | Large complex enterprises | Regulated content mgmt | Complex workflows | CRM-centric, Speed |

---

## PATTERNS & ANTI-PATTERNS

| Platform | Pattern | Anti-Pattern | Why Avoid |
|----------|---------|--------------|-----------|
| **Dynamics** | Power Platform ALM (Solutions, Pipelines) | Direct prod customization | No version control, no rollback |
| **SAP** | Side-by-side on BTP (Clean Core) | Heavy ABAP modifications | Upgrade nightmare, lock-in |
| **Documentum** | REST API + xPlore search | Direct DB access | Bypasses security, versioning |
| **IBM BAW** | BPMN + Decision Services (DMN) | Hardcoded logic in coaches | Inflexible, untestable |
| **Salesforce** | SFDX + Scratch Orgs + Unlocked Packages | Change Sets, direct prod | No versioning, no CI/CD |
| **All** | API-first integration | Point-to-point, shared DB | Coupling, fragility |
| **All** | Event-driven (CDC, Platform Events) | Polling, batch sync | Latency, complexity |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. Salesforce: "How do you handle governor limits in Apex?"

```
GOVERNOR LIMIT STRATEGIES:

1. BULKIFY EVERYTHING:
   • Process records in collections, not loops
   • Single SOQL/DML per transaction (use Maps)
   • Example: Map<Id, List<Contact>> contactsByAccountId

2. ASYNC PROCESSING:
   • Future methods (simple, fire-and-forget)
   • Queueable (chaining, state, higher limits)
   • Batch Apex (millions of records, 50M query rows)
   • Scheduled Apex (recurring)

3. QUERY OPTIMIZATION:
   • Selective WHERE clauses (indexed fields)
   • Avoid SELECT * — specify fields
   • Use LIMIT, OFFSET carefully
   • Consider Skinny Tables for huge objects

4. PLATFORM EVENTS / CDC:
   • Decouple processing from transaction
   • Near real-time, replayable, ordered
   • Subscriber: Trigger, Flow, CometD, Pub/Sub API

5. FLOW VS APEX:
   • Flow: Declarative, admin-maintainable, lower limits
   • Apex: Complex logic, recursion control, higher limits
   • Use Flow for orchestration, Apex for computation

5. MONITORING:
   • Limits class (Limits.getQueries(), Limits.getDMLRows())
   • Debug logs (Analysis perspective)
   • Event Monitoring (API, Apex, Report)
```

---

### 2. SAP: "Clean Core strategy — what does it mean?"

```
CLEAN CORE = Minimize modifications to SAP standard code.

PRINCIPLES:
1. NO MODIFICATIONS to SAP standard objects (tables, programs, screens)
2. NO CUSTOM CODE in SAP namespace (Z/Y programs)
3. EXTEND via Side-by-Side on BTP (CAP, Kyma, FaaS)
4. USE KEY USER EXTENSIBILITY (App Builder, Custom Fields, Logic)
5. ADOPT STANDARD PROCESSES (Fit-to-Standard workshops)

TECHNICAL IMPLEMENTATION:
• ABAP Cloud Development Model (ABAP for Cloud)
• CDS Views for data models (not custom tables)
• RAP for OData services (not hand-coded)
• Business Events for integration (not RFC/IDoc)
• BTP Extension Suite (CAP, Kyma, Functions, API Mgmt)

BENEFITS:
✅ Quarterly upgrades without regression testing
✅ Innovation adoption (AI, ML, new features)
✅ Reduced TCO (less custom code maintenance)
✅ Compliance (audit-ready standard processes)

MIGRATION PATH:
1. Inventory custom code (Code Inspector, ATC)
2. Classify: Retire, Replace with Standard, Extend on BTP
3. Refactor to ABAP Cloud / Move to BTP
4. Activate Clean Core checks in transport system
```

---

### 3. Dynamics: "How does Dual-write work and when does it fail?"

```
DUAL-WRITE ARCHITECTURE:
• Real-time sync between F&O (Finance & Operations) and Dataverse
• Entity maps define field mappings (bidirectional)
• Runs in F&O transaction scope (rollback on failure)
• Uses Azure Service Bus internally

WHEN IT FAILS:
1. SCHEMA MISMATCH — Field type/length changes not propagated
2. DATA QUALITY — Required fields missing, invalid references
3. PERFORMANCE — Large batches timeout (30s limit)
4. CONFLICTS — Concurrent updates in both systems
5. SERVICE BUS ISSUES — Throttling, dead letters, poison messages

BEST PRACTICES:
• Use Virtual Tables for read-only scenarios (no sync overhead)
• Monitor Dual-write runtime (Health, Error logs, Latency)
• Implement retry logic with exponential backoff
• Use Change Tracking for audit/reconciliation
• Consider Bring Your Own Database (BYOD) for analytics

ALTERNATIVES:
• Virtual Tables (read external data in Dataverse)
• Data Integrator (batch, scheduled)
• Custom Integration (Logic Apps, Power Automate, Azure Functions)
• BYOD (Export to Azure SQL for reporting)
```

---

### 4. Documentum: "How do you implement versioning and retention?"

```
VERSIONING:
• Implicit (checkin/checkout) or Explicit (explicit version labels)
• Major (1.0, 2.0) / Minor (1.1, 1.2) versions
• Branching (parallel versions) — use sparingly
• Freeze/Lock — Prevent further versions

RETENTION (Trusted Content Services):
• Retention Policies — Time-based (years), Event-based (case closed)
• Legal Hold — Override retention, prevent deletion
• Disposition — Review → Approve → Destroy/Archive
• Audit Trail — Immutable log of all actions

ARCHITECTURE:
• Policy Objects — Define rules (dm_retention_policy)
• Policy Application — Apply to folders/types (dm_policy_application)
• Retention Jobs — Scheduled evaluation, disposition
• Compliance Reports — Evidence for auditors

INTEGRATION:
• REST API — Apply policies programmatically
• Event Server — React to lifecycle events
• InfoArchive — Long-term archive tier
• Cloud Tiering — S3 Glacier, Azure Archive
```

---

### 5. IBM BAW: "How do you handle long-running process compensation?"

```
COMPENSATION IN BPMN (Saga Pattern):

1. TRANSACTION SUBPROCESS (Double-lined boundary):
   • Contains: Reserve Payment → Reserve Inventory → Schedule Shipment
   • Success: Normal end event
   • Failure: Cancel end event (thick circle) → Triggers compensation

2. COMPENSATION BOUNDARY EVENTS:
   • Attach to activity: "Reserve Payment"
   • Compensation Activity: "Refund Payment" (separate process)
   • Triggered automatically on transaction cancellation

3. COMPENSATION HANDLER:
   • Separate BPMN process (Reusable)
   • Input: Original process data
   • Logic: Reverse operations (refund, release, notify)
   • Must be idempotent (safe to retry)

4. PROGRAMMATIC COMPENSATION (Java/JS):
   • context.getCompensationHandler().register("activityName", handler)
   • Handler receives original data, performs reversal
   • Must be idempotent (check if already compensated)

4. STATE MANAGEMENT:
   • Process variables preserved for compensation
   • Business keys (Order ID) for correlation
   • Snapshot isolation (PDW for analytics)

TESTING:
• Simulate failures at each step
• Verify compensation executes correctly
• Test idempotency (re-run compensation)
• Load test compensation under peak
```

---

### 6. Enterprise Integration: "How do you choose between sync and async integration?"

```
DECISION FRAMEWORK:

USE SYNCHRONOUS (REST/gRPC) WHEN:
✓ Caller needs result NOW to proceed
✓ User-facing operation (latency critical)
✓ ACID transaction required (immediate consistency)
✓ Simple request/response (CRUD)
✓ Read-your-writes guarantee needed

USE ASYNCHRONOUS (Kafka/RabbitMQ/Events) WHEN:
✓ Work is slow/retryable (email, video, report generation)
✓ Spike absorption needed (traffic smoothing)
✓ Cross-service workflows (Saga)
✓ Loose coupling (services evolve independently)
✓ Eventual consistency acceptable
✓ High throughput, fire-and-forget

HYBRID PATTERN (Real-world):
• Order placement: Sync (validate + create order) + Async (fulfillment events)
• Payment: Sync (authorization) + Async (capture, settlement)
• Search/Read: Sync (CQRS read model)
• Analytics/Reporting: Async (CDC → Data Lake)

TECHNOLOGY SELECTION:
• REST: Public APIs, simple CRUD, caching
• gRPC: Internal services, streaming, contracts
• Kafka: Events, audit, replay, high throughput
• RabbitMQ: Commands, complex routing, priority
• Platform Events/CDC: Salesforce/SAP event-driven
```

---

### 7. Cloud Migration: "How do you migrate on-prem ERP to cloud?"

```
ERP CLOUD MIGRATION STRATEGY (SAP/Dynamics/Oracle):

1. ASSESSMENT:
   • Inventory: Custom code, integrations, data volume, interfaces
   • Readiness: Clean Core (SAP), Solution Health (Dynamics)
   • Business Case: TCO, Risk, Timeline, ROI

2. APPROACH SELECTION:
   • LIFT & SHIFT (Rehost) — VMs to cloud, minimal change
     → Fast, but misses cloud benefits
   • REPLATFORM (Lift & Optimize) — Minor refactor, managed services
     → Balanced
   • REFACTOR (Re-architect) — Clean Core, Cloud-native
     → Best long-term, highest effort
   • REPLACE (SaaS) — Move to standard SaaS (Dynamics, S/4HANA Cloud)
     → Lowest customization, fastest innovation

3. DATA MIGRATION:
   • Extract → Cleanse → Transform → Load (ETL)
   • Parallel runs (dual-write) for validation
   • Delta sync for cutover
   • Archival strategy for historical data

4. INTEGRATION REFACTORING:
   • Point-to-point → API Gateway / Event Mesh
   • SOAP/IDoc → REST/OData/Async Events
   • Cannibalize custom interfaces → Standard APIs

5. CUTOVER STRATEGY:
   • Big Bang (weekend) — High risk, fast
   • Phased (module by module) — Lower risk, longer
   • Parallel Run — Dual-write, compare, switch
   • Pilot (one division/region) → Expand

5. GOVERNANCE:
   • Clean Core enforcement (no custom code in core)
   • DevOps (CI/CD, ALM, Testing)
   • Change Management (Training, Support)
```

---

### 8. Multi-ERP: "How do you integrate SAP with Salesforce?"

```
SAP ↔ SALESFORCE INTEGRATION PATTERNS:

1. MULESOFT (Recommended by Salesforce):
   • Anypoint Platform — API-led connectivity
   • SAP Connector (IDoc, BAPI, RFC, OData)
   • Salesforce Connector (REST, Bulk, Streaming, CDC)
   • DataWeave transformation (XML/IDoc ↔ JSON)
   • Error handling, retry, monitoring built-in

2. SAP INTEGRATION SUITE (SAP BTP):
   • Cloud Integration (iFlows) — SAP → Salesforce
   • Prebuilt content (Order-to-Cash, Quote-to-Cash)
   • API Management (expose SAP as managed APIs)
   • Event Mesh (S/4HANA Business Events → Salesforce Platform Events)

3. NATIVE CONNECTORS:
   • Salesforce Connect (OData) — Virtual tables in SF from SAP
   • SAP OData Services → External Objects in SF
   • Real-time, read-only (mostly)

4. MIDDLEWARE ALTERNATIVES:
   • Dell Boomi, Celigo, Jitterbit (iPaaS)
   • Custom (Azure Logic Apps, AWS Step Functions, Kafka)

KEY DATA FLOWS:
• Account/Customer ↔ Business Partner (Bidirectional)
• Product/Material ↔ Product (SAP → SF)
• Quote/Order ↔ Sales Order (SF → SAP)
• Invoice/Payment ↔ Billing Document (SAP → SF)
• Inventory/ATP ↔ Product Availability (SAP → SF)

ARCHITECTURE PRINCIPLES:
• Master Data: Single source of truth (SAP for Finance/Logistics)
• Transactional Data: Event-driven (SAP Business Events → Platform Events)
• Idempotency: External IDs on both sides
• Error Handling: Dead letter queues, retry, alerting
• Monitoring: End-to-end correlation IDs
```

---

### 9. Low-Code: "When to use Power Apps vs custom development?"

```
POWER APPS (Canvas/Model-driven) — USE WHEN:
✓ Business logic owned by citizen developers
✓ Rapid prototyping (days, not weeks)
✓ Forms-over-data (CRUD + simple validation)
✓ Mobile/Offline requirements
✓ Integration with Dataverse/365 ecosystem
✓ Governance via Solutions, ALM pipelines

CUSTOM DEVELOPMENT (Apex/LWC/Pro Code) — USE WHEN:
✓ Complex algorithms, heavy computation
✓ High-volume batch processing
✓ Complex UI/UX (custom rendering, canvas, charts)
✓ Tight governor limit management needed
✓ Intellectual property / algorithm protection
✓ Performance-critical paths

HYBRID APPROACH (Best of Both):
• Power Apps for UI/Orchestration
• Custom Connectors → Azure Functions / API Management
• PCF (Power Apps Component Framework) for custom controls
• Dataverse Plugins (C#) for complex server logic
• Flow for orchestration, Apex for computation

GOVERNANCE:
• Environment strategy (Dev/Test/Prod)
• Solution layers (Base, Core, Extension)
• ALM (Pipelines, Git, Code Review)
• Security (DLP policies, Admin analytics)
• Monitoring (CoE Toolkit, Power Platform Analytics)
```

---

### 10. AI in Enterprise: "How are ERP vendors adding AI?"

```
AI IN ENTERPRISE SOFTWARE (2024):

SAP (Joule + AI Core):
• Generative: Code generation (ABAP), Test creation, Documentation
• Predictive: Cash forecasting, Demand sensing, Predictive maintenance
• Embedded: Invoice processing (OCR), Expense categorization
• AI Core: Train/deploy custom models on BTP

SALESFORCE (Einstein + Data Cloud):
• Generative: Email drafting, Case summaries, Code generation
• Predictive: Lead scoring, Opportunity forecasting, Churn prediction
• Conversational: Einstein Bots, Slack GPT
• Data Cloud: Unified profile → AI-ready data

MICROSOFT (Copilot + AI Builder):
• Copilot in Dynamics/Power Platform: App generation, Flow creation
• AI Builder: Form processing, Object detection, Prediction models
• Azure OpenAI integration: Custom LLM apps in Power Platform

ORACLE (AI in Fusion):
• Digital Assistant, AP Automation, Project forecasting

IBM (Watson + BAW):
• Process mining, Decision optimization, NLP for content

ARCHITECTURE PATTERN:
• Data Layer: Unified, governed, AI-ready (Data Cloud, Datasphere)
• Model Layer: Train/Tune/Serve (Managed ML platforms + AI Core, SageMaker, Vertex AI)
• Application Layer: Embedded (invisible) + Extensible (API/SDK)
• Governance: Model registry, Bias detection, Explainability, Compliance

INTEGRATION:
• External models via API (OpenAI, Anthropic, etc.)
• BYOM (Bring Your Own Model) + Embedded models
• RAG (Retrieval Augmented Generation) for enterprise knowledge
```

---

### 11. Data Mesh: "Explain Data Mesh principles and when to adopt."

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

### 12. Architecture Decision Records: "How do you implement ADRs in a team?"

```
ADR IMPLEMENTATION PLAYBOOK:

1. TEMPLATE (Markdown in /docs/adr/ADR-XXXX.md):
# ADR-XXXX: Title
## Status: Proposed | Accepted | Rejected | Superseded
## Context: What forces drove this?
## Decision: What are we doing?
## Consequences: + / - / Risks / Mitigations
## Alternatives: What else considered?
## Related: ADR-XXXX, RFC-XXXX

2. PROCESS:
• Any structural decision → ADR (in PR with code)
• Author writes draft (1-2 hours)
• 3-day async review (GitHub PR comments)
• Architecture Board sync (30 min, controversial only)
• Decider (Chief Architect) decides
• Merge → Communicate in #arch-decisions

3. TOOLING:
• adr-tools / adr-log for numbering
• GitHub Action: Validate template, check links
• Auto-generated index page

4. GOVERNANCE:
• Quarterly review: Any superseded? Any missing?
• Architecture Board: Reviews controversial ADRs
• Metrics: % decisions with ADR, time-to-decide, supersession rate
• Exception process: Time-boxed, documented in ADR

5. CULTURE:
• "Architecture is everyone's responsibility"
• ADRs written BY teams, not FOR teams
• Decisions documented, not tribal knowledge
• Dissenting views recorded (not suppressed)
```

---

### 13. Microservices Decomposition: "How do you identify service boundaries?"

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
   • Team has all skills to own service end-to-end
   • Cognitive load manageable (< 7 services/team)

ANTI-PATTERNS:
❌ Nanoservices (too small, chatty, operational hell)
❌ Shared database (defeats independence)
❌ Distributed monolith (deploy together, fail together)
❌ Anemic services (no behavior, just CRUD)

EVOLUTION:
Start Modular Monolith → Extract when pain > cost
Pain signals: Team friction, scaling needs, tech heterogeneity, release cadence
```

---

### 13. Observability: "What are the three pillars and how do you use them?"

```
THREE PILLARS (Cindy Sridharan):

1. METRICS (Aggregated, numeric, time-series)
   - RED: Rate, Errors, Duration (per service/endpoint)
   - USE: Saturation, Utilization, Errors (USE) for resources
   - SLI/SLO: Availability, Latency, Throughput
   - Tools: Prometheus, Grafana, Datadog
   - Alerting: Symptom-based (SLO burn rate), not cause-based

2. LOGS (Immutable, timestamped, structured)
   - JSON, correlation IDs (trace_id, span_id)
   - Levels: ERROR, WARN, INFO, DEBUG
   - Structured: {timestamp, level, message, trace_id, fields...}
   - Sampling: High-volume DEBUG → sample, ERROR → 100%
   - Tools: Loki, Elasticsearch, CloudWatch, Datadog

3. TRACES (Request flow across services)
   - W3C TraceContext (traceparent header)
   - Spans: Name, Kind, Start/End, Attributes, Events, Links
   - Sampling: Head-based (10%) + Tail-based (keep errors)
   - Tools: Jaeger, Zipkin, Tempo, X-Ray, Zipkin

UNIFIED WORKFLOW:
Dashboard (Metrics) → Alert fires → Click trace ID → View trace (spans) → Click span → View logs
```

---

### 14. Testing: "What's your testing strategy for microservices?"

```
TEST PYRAMID (Microservices):

1. UNIT TESTS (70-80%)
   - Pure functions, domain logic, validators
   - Fast (<1ms), isolated, no external deps
   - Framework: xUnit, JUnit, Jest, pytest
   - Target: 80%+ coverage on domain

2. INTEGRATION TESTS (15-20%)
   - Service + real dependencies (DB, Kafka, Redis)
   - Testcontainers for ephemeral infra
   - Contract tests (Consumer-driven: Pact)
   - API tests (happy path + 2 error paths per endpoint)

3. CONTRACT TESTS (Critical for microservices)
   - Consumer defines expectations (Pact)
   - Provider verifies against contract
   - Runs in both consumer + provider CI
   - Prevents breaking changes

4. END-TO-END (5-10%)
   - Critical user journeys (happy path only)
   - Staging environment, real data (anonymized)
   - Playwright, Cypress, Selenium
   - Slow, flaky — minimize

5. CHAOS ENGINEERING (Periodic)
   - Kill pods, inject latency, fail DNS
   - Verify resilience, SLOs
   - Tools: Litmus, Chaos Mesh, Gremlin

CI/CD GATES:
- PR: Unit + Contract + Static Analysis
- Merge: Integration + Contract (provider)
- Deploy: Smoke tests in staging
- Post-deploy: Synthetic monitoring
```

---

### 15. Architecture Decision: "How do you document and communicate architectural decisions?"

```
ARCHITECTURE DECISION RECORDS (ADRs):

FORMAT (Markdown in /docs/adr/ADR-XXXX.md):
# ADR-XXXX: Title
## Status: Proposed | Accepted | Rejected | Superseded
## Context: What forces drove this?
## Decision: What are we doing?
## Consequences: + / - / Risks / Mitigations
## Alternatives: What else considered?
## Related: ADR-XXXX, RFC-XXXX

PROCESS:
• Any structural decision → ADR (in PR with code)
• Author writes draft (1-2 hours)
• 3-day async review (GitHub PR comments)
• Architecture Board sync (30 min, controversial only)
• Decider (Chief Architect) decides
• Merge → Communicate in #arch-decisions

TOOLING:
• adr-tools / adr-log for numbering
• GitHub Action: Validate template, check links
• Auto-generated index page

GOVERNANCE:
• Quarterly review: Any superseded? Any missing?
• Architecture Board: Reviews controversial ADRs
• Metrics: % decisions with ADR, time-to-decide, supersession rate
• Exception process: Time-boxed, documented in ADR

CULTURE:
• "Architecture is everyone's responsibility"
• ADRs written BY teams, not FOR teams
• Decisions documented, not tribal knowledge
• Dissenting views recorded (not suppressed)
```