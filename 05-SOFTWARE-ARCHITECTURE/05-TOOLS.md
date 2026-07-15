# TOOLS
## Exam-Style Study Notes

**Roadmap Section:** Tools → Slack, Communication, GitHub, Marketing Skills, Actors

---

## 1. SLACK (TEAM COMMUNICATION)

### WHY? (Problem Statement)

**Communication is the lifeblood of distributed teams.** Slack (or Teams, Discord, Mattermost) is where:
- Decisions happen (or get lost)
- Incidents are coordinated
- Knowledge is shared (or siloed)
- Culture is built (or eroded)

**Poor Slack hygiene =** noise, burnout, missed decisions, tribal knowledge.

---

### HOW? (Slack Architecture for Engineering Teams)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED CHANNEL TOPOLOGY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  #team-<name>                    → Team internal (private)                 │
│  #proj-<name>                    → Project/initiative (public)             │
│  #svc-<service-name>             → Service ownership (public)              │
│  #inc-<severity>                 → Incident response (public)              │
│  #oncall-<rotation>              → On-call coordination (private)          │
│  #deploy-<env>                   → Deployment notifications (public)       │
│  #alert-<system>                 → Alerting (public, threaded)             │
│  #rfc-<topic>                    → Request for Comments (public)           │
│  #learn-<topic>                  → Knowledge sharing (public)              │
│  #random / #watercooler          → Culture (public)                        │
│                                                                             │
│  NAMING CONVENTION:                                                         │
│  ✅ #svc-order-api, #proj-black-friday, #inc-sev1, #rfc-auth-model         │
│  ❌ #order-api, #project1, #alerts, #general-discussion                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Slack Standards & Automation)

#### A. CHANNEL HYGIENE

| Practice | Standard |
|----------|----------|
| **Purpose** | Every channel has `#channel-purpose` topic |
| **Defaults** | Public by default, private only when necessary |
| **Archiving** | Archive inactive > 90 days (bot reminder) |
| **Threads** | Use threads for replies (reduce noise) |
| **Reactions** | 👀 = seen, ✅ = done, ❓ = question, 🚨 = urgent |
| **Mentions** | `@here` for time-sensitive, `@channel` rarely, prefer threads |

#### B. INCIDENT COMMUNICATION PROTOCOL

```
INCIDENT DECLARED → #inc-sev1 (or sev2/sev3)
│
├─ Incident Commander posts: 
│   "INC-2025-001 | SEV-1 | Order API 5xx | IC: @jane | Scribe: @bob"
│
├─ All updates in THREAD under incident message
│   "🔍 Investigating payment gateway timeout"
│   "🔧 Rolling back deploy v1.3.2"
│   "✅ Resolved - payment gateway restored"
│
├─ Status page updated via /statuspage command
├─ Stakeholder summary posted to #inc-sev1-summary (read-only for execs)
└─ Post-incident: Link to incident doc, retrospective scheduled
```

#### C. AUTOMATION (Slack Apps / Workflows)

```yaml
# .github/workflows/slack-notify.yml
name: Deploy Notification
on:
  deployment_status:
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: slackapi/slack-github-action@v1.23.0
        with:
          channel-id: ${{ secrets.SLACK_DEPLOY_CHANNEL }}
          slack-message: |
            *Deployment ${{ job.status }}*
            Repo: ${{ github.repository }}
            Env: ${{ github.event.deployment.environment }}
            Ref: ${{ github.event.deployment.ref }}
            Actor: ${{ github.actor }}
            <${{ github.event.deployment_status.target_url }}|View Details>
```

**Essential Slack Integrations:**
- GitHub / GitLab (PRs, deploys, releases)
- PagerDuty / Opsgenie (on-call, incidents)
- Datadog / Grafana / Prometheus (alerts)
- Jira / Linear (issue updates)
- Custom: `/deploy`, `/rollback`, `/incident`, `/rfc` slash commands

---

## 2. COMMUNICATION (ASYNC-FIRST ENGINEERING)

### WHY? (Problem Statement)

**Synchronous communication (meetings, calls) doesn't scale.**
- Time zone distribution
- Deep work interruption
- Knowledge not captured
- Decision latency

**Async-first =** decisions documented, inclusive, searchable, scalable.

---

### HOW? (Communication Protocols)

#### A. DECISION-MAKING: RFC PROCESS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RFC (REQUEST FOR COMMENTS) LIFECYCLE                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. DRAFT (Author)                                                          │
│     ├─ Create RFC in /rfc folder (markdown)                                │
│     ├─ Template: Context, Problem, Proposal, Alternatives, Risks          │
│     ├─ Post to #rfc-<topic> with link                                      │
│     └─ Tag reviewers (@architect, @tech-lead, @security)                   │
│                                    │                                        │
│                                    ▼                                        │
│  2. DISCUSSION (7 days minimum)                                             │
│     ├─ Comments in doc (GitHub/Confluence)                                 │
│     ├─ Sync clarification call (optional, recorded)                        │
│     ├─ Author updates RFC based on feedback                                │
│     └─ Controversial? → Architecture Review Board                          │
│                                    │                                        │
│                                    ▼                                        │
│  3. DECISION (Decider)                                                      │
│     ├─ Status: ACCEPTED / REJECTED / DEFERRED                              │
│     ├─ Rationale documented                                                │
│     ├─ If ACCEPTED → Create ADR, assign implementation owner              │
│     └─ Close RFC thread, link to ADR                                       │
│                                    │                                        │
│                                    ▼                                        │
│  4. IMPLEMENTATION                                                          │
│     ├─ Track in project board                                              │
│     ├─ Link PRs to RFC                                                     │
│     └─ Close when deployed                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**RFC Template:**
```markdown
# RFC-0042: Migrate Order Service to Event-Driven Fulfillment

**Status:** Draft | **Author:** @jane | **Created:** 2025-01-15
**Reviewers:** @architect, @payment-lead, @security

## Context
Current synchronous fulfillment creates cascade failures...

## Problem
- Payment timeout → Order stuck
- Inventory unavailable → 500 to customer
- No audit trail for compliance

## Proposal
Adopt event-driven choreography with Kafka...

## Alternatives Considered
1. Saga Orchestrator — rejected: over-engineered
2. CDC from Order DB — rejected: leaky abstraction

## Risks & Mitigations
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Event loss | Low | High | Outbox pattern + idempotent consumers |
| Schema evolution | Medium | Medium | Schema Registry (BACKWARD compat) |

## Open Questions
- [ ] How handle partial fulfillment rollback?
- [ ] Migration strategy for in-flight orders?
```

#### B. MEETING HYGIENE (When Sync Is Needed)

| Meeting Type | Max Duration | Required | Output |
|--------------|--------------|----------|--------|
| **Architecture Review** | 60 min | RFC author, reviewers | Decision recorded |
| **Incident Retrospective** | 60 min | IC, scribe, involved | Action items + doc |
| **Sprint Planning** | 90 min | Whole team | Sprint goal + committed work |
| **1:1** | 30 min | EM + report | Career, blockers, feedback |
| **All Hands** | 30 min | Company | Strategy, wins, Q&A |

**Meeting Rules:**
- 📋 Agenda in invite (no agenda = decline)
- 📝 Notes in shared doc (not Slack)
- ✅ Action items: Owner + Due Date
- 🎥 Recorded for async consumption
- ❌ No "status update" meetings (use dashboard)

#### C. WRITTEN COMMUNICATION STANDARDS

```
BLUF (Bottom Line Up Front):
"Decision: We're adopting Kafka for order fulfillment. 
 Reason: Decouples services, enables replay, meets audit needs.
 Action: Team leads review RFC-0042 by Friday."

STRUCTURE:
1. BLUF (Decision/Ask/Info)
2. Context (Why now?)
3. Details (What/How)
4. Action Required (Who/When)
5. References (Links)
```

---

## 3. GITHUB (SOURCE CONTROL & COLLABORATION)

### WHY? (Problem Statement)

GitHub is **more than git hosting** — it's the collaboration platform for:
- Code review (quality gate)
- CI/CD (automation)
- Project management (issues, projects)
- Documentation (wiki, pages)
- Security (Dependabot, CodeQL, secrets)
- Release management

---

### HOW? (GitHub Configuration Standards)

#### A. BRANCH PROTECTION (Required for all repos)

```yaml
# Branch protection rules (via API or UI)
main:
  required_status_checks:
    strict: true
    contexts:
      - "ci/lint"
      - "ci/test"
      - "ci/security"
      - "ci/contract"
  enforce_admins: true
  required_pull_request_reviews:
    required_approving_review_count: 2
    dismiss_stale_reviews: true
    require_code_owner_reviews: true
  restrictions:
    users: []
    teams: ["platform-team"]
  allow_force_pushes: false
  allow_deletions: false
  required_linear_history: true
  required_signatures: true
  lock_branch: false
  allow_fork_syncing: true
```

#### B. REPOSITORY STRUCTURE (Standard)

```
repo-root/
├── .github/
│   ├── workflows/           # CI/CD pipelines
│   ├── ISSUE_TEMPLATE/      # Bug, Feature, RFC, Security
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── dependabot.yml       # Auto-updates
│   ├── CODEOWNERS           # Review routing
│   └── SECURITY.md          # Vulnerability reporting
├── docs/
│   ├── adr/                 # Architecture Decision Records
│   ├── api/                 # OpenAPI specs
│   ├── runbooks/            # Operational procedures
│   └── onboarding.md
├── src/                     # Source code
├── tests/                   # Test code (mirrors src)
├── scripts/                 # Automation scripts
├── Dockerfile               # Container build
├── docker-compose.yml       # Local dev stack
├── Makefile / justfile      # Common commands
├── README.md                # Project overview
├── CONTRIBUTING.md          # Contribution guide
├── LICENSE
├── .gitignore
├── .editorconfig
└── renovate.json            # Dependency updates
```

#### C. PR TEMPLATE (`.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
## Description
<!-- Link to RFC/Issue: RFC-0042, JIRA-1234 -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Refactoring
- [ ] Documentation
- [ ] Performance
- [ ] Security

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Contract tests updated
- [ ] Manual verification steps:
  1. Deploy to staging
  2. Run `./scripts/smoke-test.sh`
  3. Verify metrics in dashboard

## Checklist
- [ ] Code follows style guide (lint passes)
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated (README, ADR, API specs)
- [ ] No secrets committed
- [ ] Dependencies scanned (Dependabot clean)
- [ ] Migration scripts included (if DB changes)
- [ ] Feature flag added (if gradual rollout)
- [ ] Rollback plan documented

## Screenshots / Logs
<!-- If UI or observable change -->

## Deployment Notes
- [ ] Requires DB migration (run before deploy)
- [ ] Requires config change (feature flag: `new-fulfillment`)
- [ ] Breaking API change (version bump: v1 → v2)
- [ ] Canary deployment recommended (10% → 50% → 100%)
```

#### D. GITHUB ACTIONS (CI/CD Standards)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet format --verify-no-changes
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release --filter "Category=Unit" --collect:"XPlat Code Coverage"
      - uses: codecov/codecov-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v2
        with: { languages: csharp }
      - uses: github/codeql-action/analyze@v2
      - uses: actions/setup-dotnet@v4
      - run: dotnet tool restore
      - run: dotnet nuget list package --vulnerable --include-transitive

  contract:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pact-foundation/pact-action@v2
        with: { provider: 'order-api', consumer: 'web-app' }

  build:
    needs: [lint, security, contract]
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/org/order-api
          tags: |
            type=ref,event=branch
            type=sha
            type=raw,value=latest,enable={{is_default_branch}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

### WHAT? (GitHub Project Management)

#### A. ISSUE TEMPLATES

```yaml
# .github/ISSUE_TEMPLATE/bug.yml
name: Bug Report
description: Report a reproducible issue
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: "Thanks for reporting! Please fill all sections."
  - type: input
    id: service
    attributes: { label: "Affected Service", placeholder: "order-api" }
    validations: { required: true }
  - type: textarea
    id: reproduction
    attributes: { label: "Steps to Reproduce", placeholder: "1. ...\n2. ...\n3. ..." }
    validations: { required: true }
  - type: textarea
    id: expected
    attributes: { label: "Expected Behavior" }
    validations: { required: true }
  - type: textarea
    id: actual
    attributes: { label: "Actual Behavior" }
    validations: { required: true }
  - type: textarea
    id: logs
    attributes: { label: "Logs/Traces", render: shell }
  - type: dropdown
    id: severity
    attributes: { label: "Severity", options: ["SEV-1 (Critical)", "SEV-2 (High)", "SEV-3 (Medium)", "SEV-4 (Low)"] }
    validations: { required: true }
```

#### B. GITHUB PROJECTS (Sprint Board)

```
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│   BACKLOG   │   READY     │ IN PROGRESS │ IN REVIEW   │    DONE     │
│             │ (Sprint)    │  (WIP: 3)   │             │             │
├─────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ RFC-0042    │ #1234       │ #1235       │ #1236       │ #1230       │
│ JIRA-567    │ #1237       │ #1238       │ #1239       │ #1231       │
│             │ #1239       │             │             │ #1232       │
│  (Triaged,  │  (Estimated,│  (Assigned, │  (PR open,  │  (Merged,   │
│  prioritized)│  no blockers)│  active)   │  CI passing)│  deployed)  │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

AUTOMATION:
- Issue moved to "In Progress" → Assign to PR author
- PR opened → Move issue to "In Review"
- PR merged → Move issue to "Done", delete branch
- Stale > 30 days → Comment, then close
```

---

## 4. MARKETING SKILLS (INTERNAL TECHNICAL MARKETING)

### WHY? (Problem Statement)

**Great architecture unseen = wasted architecture.** Architects must:
- Sell vision to leadership (funding)
- Recruit adopters (adoption)
- Advocate for quality (priority)
- Build credibility (trust)

---

### HOW? (Internal Marketing Playbook)

#### A. ARCHITECTURE NEWSLETTER (Monthly)

```
📰 ARCHITECTURE MONTHLY — January 2025

🎯 THIS MONTH'S FOCUS: Resilience Patterns
├── ✅ Circuit Breaker deployed to 12 services
├── ✅ Bulkhead isolation for payment path
├── 🔄 Retry-with-jitter standardization (PR #2341)
└── 📊 Result: 67% fewer cascade incidents

📈 METRICS THAT MATTER
├── Deployment frequency: 47/day (+12%)
├── Lead time: 2.3hr → 1.8hr
├── Change failure rate: 3.1% → 1.8%
├── MTTR: 45min → 18min
└── Availability: 99.94% → 99.97%

🏆 WINS
├── Order Service: 99.99% SLO achieved (Black Friday!)
├── Migration: 3 services strangled from monolith
├── Cost: $12K/mo saved via right-sizing

📚 LEARNING
├── RFC-0042 accepted: Event-driven fulfillment
├── Tech Talk: "Distributed Tracing Deep Dive" (recording)
├── Office Hours: Fridays 2pm, #arch-office-hours

🔮 NEXT MONTH
├── CQRS read model for Order History
├── Schema Registry rollout
├── Chaos Engineering: Game Day Feb 14
```

#### B. TECH TALKS / BROWN BAGS

| Format | Frequency | Audience | Example Topics |
|--------|-----------|----------|----------------|
| **Deep Dive** | Monthly | Engineers | "How we reduced Kafka lag 90%" |
| **Architecture Kata** | Quarterly | Mixed | "Design a URL shortener in 60 min" |
| **Incident Review** | Post-incident | All | "SEV-1: Payment Gateway Outage" |
| **Tool Showcase** | Ad-hoc | Engineers | "New: GitHub CodeQL for security" |
| **Office Hours** | Weekly | Anyone | "Bring your design questions" |

#### C. RFC ANNOUNCEMENTS

```markdown
# 📢 RFC-0042 OPEN FOR COMMENT: Event-Driven Fulfillment

**What:** Migrating Order Service fulfillment from sync REST to event-driven choreography

**Why:** 
- Eliminate cascade failures (payment timeout → stuck orders)
- Enable audit trail for compliance (SOX)
- Support 10x scale for holiday peaks

**Impact on You:**
- Payment Team: New `PaymentReserved` event to consume
- Inventory Team: New `StockReserved` event to produce
- All: New correlation IDs in logs (trace across services)

**Timeline:**
- Comments due: Jan 22 (Friday)
- Decision: Jan 25 (Monday)
- Implementation: Sprint 4-6 (Feb-Mar)
- Migration: Canary → Full (Apr)

**Links:**
- 📄 RFC: https://github.com/org/arch/rfc/0042
- 💬 Discussion: #rfc-event-driven-fulfillment
- 👤 Author: @jane (Order Team)
- 👥 Reviewers: @architect, @payment-lead, @security

Please review and comment in the doc!
```

---

## 5. ACTORS (STAKEHOLDER MANAGEMENT)

### WHY? (Problem Statement)

**Every architecture decision has human stakeholders.** Missing actors = blind spots, resistance, failed adoption.

---

### HOW? (Actor Mapping & Engagement)

#### A. STAKEHOLDER MAP (RACI per Decision)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DECISION: EVENT-DRIVEN FULFILLMENT                       │
├──────────────────┬──────────┬──────────┬──────────┬──────────┬──────────────┤
│ ACTOR            │ RESPONS. │ ACCOUNT. │ CONSULT. │ INFORM.  │ ENGAGEMENT   │
├──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────────┤
│ CTO              │          │    A     │    C     │    I     │ Monthly sync │
│ VP Engineering   │          │          │    C     │    I     │ Bi-weekly    │
│ Architect (You)  │    R     │    A     │          │    I     │ Daily        │
│ Order Tech Lead  │    R     │          │    C     │    I     │ Daily        │
│ Payment Tech Lead│    R     │          │    C     │    I     │ Weekly       │
│ Inventory Lead   │    R     │          │    C     │    I     │ Weekly       │
│ Security Eng     │          │          │    C     │    I     │ RFC review   │
│ Platform Team    │    R     │          │    C     │    I     │ Infra support│
│ Product Manager  │          │          │    A     │    I     │ Sprint plan  │
│ Support Lead     │          │          │    C     │    I     │ Runbook input│
│ Compliance       │          │          │    C     │    I     │ Audit review │
└──────────────────┴──────────┴──────────┴──────────┴──────────┴──────────────┘

R = Responsible (does work)  A = Accountable (decides)
C = Consulted (two-way)      I = Informed (one-way)
```

#### B. ENGAGEMENT STRATEGIES PER ACTOR TYPE

| Actor Type | Needs | Communication | Cadence |
|------------|-------|---------------|---------|
| **Executive (CTO/VP)** | Business outcome, risk, investment | 1-pager, dashboard, ROI | Monthly |
| **Product Manager** | Timeline, dependencies, user impact | User journey, release plan | Sprint |
| **Tech Lead** | Technical design, team capacity | Architecture diagrams, RFCs | Weekly |
| **Engineer** | Implementation details, patterns | Code, ADRs, pair programming | Daily |
| **Security** | Threat model, controls, compliance | Threat model, evidence | Per RFC |
| **Platform/Infra** | Resource needs, contracts, SLAs | Capacity plan, contracts | Per project |
| **Support/Ops** | Runbooks, alerts, debugging | Runbooks, dashboards | Pre-launch |
| **Compliance/Legal** | Audit trail, data handling, contracts | Evidence, certifications | Quarterly |

#### C. CONFLICT RESOLUTION (When Actors Disagree)

```
DISAGREEMENT RESOLUTION PROCESS:

1. IDENTIFY ROOT CAUSE
   ├─ Different priorities? → Align on business goals
   ├─ Different assumptions? → Surface & validate assumptions
   ├─ Different risk tolerance? → Quantify risks (probability × impact)
   └─ Different information? → Share data, spike if needed

2. USE DECISION FRAMEWORK
   ├─ RAPID roles clear? (who decides?)
   ├─ Trade-off matrix documented?
   ├─ Reversible? (Type 1 vs Type 2 decision)
   └─ Time-boxed? (decide by date)

3. ESCALATION PATH
   ├─ Level 1: Tech Leads agree (30 min)
   ├─ Level 2: Architect mediates (1 hr)
   ├─ Level 3: VP Engineering decides (1 day)
   └─ Level 4: CTO decides (rare)

4. DOCUMENT OUTCOME
   ├─ Decision in ADR
   ├─ Dissenting view recorded (not suppressed)
   ├─ Commit date for review (if experimental)
   └─ Communicate to all actors
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Context | Anti-Pattern | Why Avoid |
|---------|---------|--------------|-----------|
| **Async-First Communication** | Distributed teams | Meeting-heavy culture | Doesn't scale, excludes time zones |
| **RFC Process** | Significant decisions | Decisions in Slack/verbal | No record, no review, no accountability |
| **Structured Slack Channels** | Org-wide comms | Single #general channel | Noise, missed info, no searchability |
| **Branch Protection + PR Template** | Code quality | Direct pushes to main | Bugs, security issues, no review |
| **GitHub Projects (Kanban)** | Sprint visibility | Spreadsheet/Jira only | Stale, disconnected from code |
| **Internal Newsletter** | Architecture visibility | Word of mouth | Inconsistent, siloed knowledge |
| **Actor Map per Decision** | Stakeholder alignment | "Everyone knows" | Surprises, resistance, rework |
| **Decision Framework (RAPID)** | Conflict resolution | Consensus-seeking | Slow, watered-down, no ownership |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. Slack: "How do you prevent Slack from becoming a distraction?"

**Answer:**
```
STRUCTURE + NORMS + AUTOMATION:

STRUCTURE:
- Purpose-driven channels (#svc-*, #proj-*, #inc-*, #rfc-*)
- Public by default, threads for discussion
- Archive policy (90 days inactive)

NORMS:
- @here only for time-sensitive, @channel rarely
- Reactions: 👀=seen, ✅=done, 🚨=urgent
- No expectation of immediate response (async-first)
- Status: 🟢 Focus mode, 🔴 In meeting, 🟡 Available

AUTOMATION:
- GitHub → #deploy, #prs (reduce manual posts)
- PagerDuty → #inc-sev1 (incident coordination)
- Alerts → #alert-<system> (threaded, deduplicated)
- Reminders: /remind #channel "Archive review" in 90 days

CULTURE:
- "Slack is for coordination, not collaboration"
- Deep work = status 🟢 + notifications off
- Urgent = phone call / PagerDuty, not Slack
```

---

### 2. Communication: "How do you make architectural decisions in a distributed team?"

**Answer:**
```
RFC PROCESS (async-first, inclusive, documented):

1. AUTHOR writes RFC (template: Context, Problem, Proposal, Alternatives, Risks)
2. POSTS to #rfc-<topic> + GitHub/Confluence, tags reviewers
3. 7-DAY COMMENT PERIOD (async, threaded in doc)
4. OPTIONAL: 30-min sync call (recorded) for clarification
5. DECIDER (per RAPID) makes call: ACCEPTED/REJECTED/DEFERRED
6. IF ACCEPTED → ADR created, implementation owner assigned
7. CLOSE: Link RFC → ADR, announce in #arch-decisions

BENEFITS:
- Timezone-inclusive (no meeting required)
- Permanent record (why, alternatives, dissent)
- Scales (anyone can review, not just attendees)
- Forces clarity (writing exposes gaps)
```

---

### 3. GitHub: "What's your branching strategy and why?"

**Answer:**
```
TRUNK-BASED DEVELOPMENT (with short-lived feature branches):

MAIN BRANCH:
- Always deployable (protected, CI required)
- Squash merge only (clean history)
- Tags for releases (v1.3.0, v1.3.1-hotfix)

FEATURE BRANCHES:
- Lifetime < 2 days (prefer hours)
- Small PRs (< 400 lines)
- Named: feat/, fix/, chore/, docs/
- Deleted after merge

HOTFIXES:
- Branch from tag: hotfix/v1.3.1-sev1
- Fast-track review (1 approver + CI)
- Cherry-pick to main if needed

WHY NOT GITFLOW?
- Release branches = integration hell
- Long-lived branches = merge conflicts, stale code
- Main not deployable = deployment anxiety

WHY NOT MAINLINE ONLY (no branches)?
- PR review = quality gate
- CI gates (security, contract, perf) before merge
- Team prefers async review over pair-only
```

---

### 4. GitHub: "How do you handle code review at scale?"

**Answer:**
```
SCALABLE CODE REVIEW SYSTEM:

AUTOMATION FIRST (gate before human):
- Lint/format (fail fast)
- Unit/integration tests
- Security scan (CodeQL, Dependabot)
- Contract tests (Pact, Schema Registry)
- Coverage threshold

HUMAN REVIEW (focus on what tools can't catch):
- Architecture fit (layers, boundaries, patterns)
- Error handling (Result types, not exceptions for flow)
- Observability (logs, metrics, traces, correlation IDs)
- Security (validation, secrets, authorization)
- Performance (N+1, pooling, pagination, caching)
- Tests: meaningful assertions, edge cases, property-based

PROCESS:
- CODEOWNERS routes to right reviewers
- 2 approvals required (1 from owner team)
- Stale reviews dismissed on new commits
- PR size limit: 400 lines (enforced by danger/GitHub)
- Draft PRs for early feedback
- Review rotation (not same people always)

METRICS:
- PR cycle time (open → merge): target < 4 hours
- Review depth (comments per PR): target > 3 meaningful
- Rework rate (commits after review): target < 20%
```

---

### 5. Marketing: "How do you get engineers to adopt a new architectural standard?"

**Answer:**
```
ADOPTION = DESIRE + ABILITY + REINFORCEMENT

DESIRE (Why should I care?):
- Newsletter: "This standard reduced incidents 60%"
- Tech talk: Live demo of problem → solution
- RFC: Engineers co-author, feel ownership
- Pilot: Volunteer team tries first, shares wins

ABILITY (Can I do it?):
- Paved road: Template repo, scaffolding CLI (`arch new svc`)
- Documentation: Step-by-step guide + examples
- Office hours: Fridays 2pm, #arch-help
- Pair programming: Architect pairs with first adopters
- Training: 2-hr workshop in sprint planning

REINFORCEMENT (Is it worth continuing?):
- CI gate: Fails build if standard violated (with fix hint)
- Code review: Checklist includes standard
- Metrics dashboard: Adoption % per team, incident correlation
- Recognition: "Team X first to 100% — shoutout in all-hands"
- Retrospective: "What's painful? Fix the standard, not the team"

TIMELINE:
- Month 1: Pilot (1 team), gather feedback
- Month 2: Refine, document, scaffold
- Month 3: Opt-in (3 teams), office hours
- Month 4: Default for new services
- Month 6: Required (CI gate), legacy migration plan
```

---

### 6. Actors: "How do you handle a stakeholder who blocks a necessary architectural change?"

**Answer:**
```
BLOCKER RESOLUTION FRAMEWORK:

1. UNDERSTAND THEIR CONCERN (not position)
   - "Help me understand what risk you see"
   - Document their specific fears (data loss? timeline? compliance?)
   - Often: unspoken assumption or past trauma

2. QUANTIFY THE TRADE-OFF
   - Current path: Cost X, Risk Y, Timeline Z
   - Proposed path: Cost A, Risk B, Timeline C
   - Status quo cost: "If we don't change, we hit scale wall in Q3"

3. FIND CREATIVE OPTIONS (not A vs B)
   - Can we phase it? (Strangler Fig)
   - Can we add safety net? (Canary, feature flag, rollback)
   - Can we isolate risk? (New service, not core migration)
   - Can we time-box experiment? (2 sprints, then decide)

4. USE DECISION FRAMEWORK
   - RAPID: Who is D (Decider)? 
   - If they're A (Agree) not D → they advise, don't decide
   - Escalate to D with trade-off matrix

5. DOCUMENT & COMMIT
   - ADR records: Decision, dissenting view, review date
   - "We'll revisit in 3 months with production data"
   - Builds trust: "You were heard, we're measuring"

EXAMPLE:
Security blocked event-driven (fear: PII in Kafka).
Resolution: Field-level encryption + schema registry PII tags + audit log.
Result: Security became champion, helped design encryption library.
```

---

### 7. Slack: "How do you handle incident communication in Slack?"

**Answer:**
```
INCIDENT COMMUNICATION PROTOCOL:

CHANNELS:
- #inc-sev1 (SEV-1/2) — public, threaded
- #inc-sev2 (SEV-3/4) — public, threaded  
- #oncall-<rotation> — private, pages
- #inc-sev1-summary — read-only, exec summary

PROCESS:
1. INCIDENT DECLARED (PagerDuty → Slack)
   "🚨 INC-2025-001 | SEV-1 | Order API 5xx | IC: @jane | Scribe: @bob"
   
2. ALL UPDATES IN THREAD (not new messages)
   🔍 "Investigating payment gateway — 504s on /charge"
   🔧 "Rolling back deploy v1.3.2 to v1.3.1"
   ✅ "Payment gateway restored — monitoring"
   
3. ROLES (explicit in channel topic):
   IC: Incident Commander (coordinates, decides)
   Scribe: Documents timeline, actions
   Comms: Updates #inc-sev1-summary, stakeholders
   Engineers: Debug, fix
   
4. STATUS PAGE (automated)
   /statuspage create "Order API degraded" — posts to status.company.com
   
5. RESOLUTION
   "✅ RESOLVED | Root cause: Payment provider cert expiry | Action: Auto-renewal enabled"
   Link to incident doc posted
   
6. RETROSPECTIVE (within 48 hrs)
   Blameless, action items → Jira, owners, due dates
```

---

### 8. Communication: "How do you write an effective RFC?"

**Answer:**
```
RFC STRUCTURE (MANDATORY SECTIONS):

# RFC-XXXX: Title (Action-Oriented)

**Status:** Draft | In Review | Accepted | Rejected | Deferred
**Author:** @name | **Created:** YYYY-MM-DD
**Reviewers:** @architect, @security, @team-lead
**Decider:** @vp-eng (per RAPID)

## Context (Why now?)
Business driver, technical pain, regulatory deadline, scale projection

## Problem (What's broken?)
Specific, measurable: "p99 latency 800ms → 200ms target", "3 SEV-1s last quarter"

## Proposal (What are we doing?)
Architecture diagram, data flow, API contracts, migration steps

## Alternatives Considered (What else?)
| Option | Pros | Cons | Why Rejected |
|--------|------|------|--------------|
| Sync orchestration | Simple, consistent | Coupling, SPOF | Availability risk |
| CDC from DB | No code change | Leaky abstraction | Schema = contract |

## Risks & Mitigations
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Event loss | Low | High | Outbox pattern + idempotent consumers |

## Open Questions
- [ ] Migration strategy for in-flight orders?
- [ ] Rollback plan if consumer fails?

## Migration Plan
Phase 1: Outbox pattern (Sprint 1-2)
Phase 2: New consumer (Sprint 3-4)
Phase 3: Canary cutover (Sprint 5)
Phase 4: Deprecate sync (Sprint 6)

## Success Metrics
- p99 latency < 200ms
- Zero data loss incidents
- Consumer lag < 30s
```

---

### 9. GitHub: "How do you manage dependencies securely?"

**Answer:**
```
MULTI-LAYER DEPENDENCY SECURITY:

1. LOCKFILES COMMITTED (all ecosystems)
   - package-lock.json / pnpm-lock.yaml / go.mod+go.sum / Cargo.lock
   - packages.lock.json (.NET)
   - Renovate/Dependabot PRs update lockfiles atomically

2. AUTOMATED UPDATES (Renovate/Dependabot)
   - Group: security (immediate), patch (weekly), minor (bi-weekly), major (monthly)
   - Require: CI pass + 1 approval
   - Auto-merge: patch + security (if tests pass)

3. VULNERABILITY SCANNING (CI + Registry)
   - CI: Trivy/Grype/CodeQL on every PR (fail on CRITICAL/HIGH)
   - Registry: Harbor/JFrog/Xray scans on push (block deploy on CRITICAL)
   - SBOM: CycloneDX/SPDX generated on every build (Syft)

4. POLICY ENFORCEMENT
   - Allowed licenses (MIT, Apache-2.0, BSD) — deny GPL/AGPL
   - Prohibited packages (known malicious, unmaintained > 2yrs)
   - Private registry proxy (Artifactory/Nexus) — no direct npmjs/pypi
   - Signature verification (cosign/slsa) for critical deps

5. SUPPLY CHAIN INTEGRITY
   - SLSA Level 3 target (provenance, hermetic builds, tamper-proof)
   - GitHub Actions: pin to SHA (uses: actions/checkout@11bd719...)
   - Reproducible builds (fixed timestamps, deterministic output)
```

---

### 10. Marketing: "How do you measure the success of an architecture initiative?"

**Answer:**
```
SUCCESS METRICS (Leading + Lagging):

LEADING (Predictive, during initiative):
├── ADOPTION RATE
│   ├── % services compliant with standard
│   ├── % new services using paved road
│   └── Time to onboard new service (target: < 1 day)
├── DEVELOPER EXPERIENCE
│   ├── PR cycle time (target: < 4 hrs)
│   ├── Build success rate (target: > 95%)
│   ├── "Works on my machine" incidents (target: 0)
│   └── Survey NPS (quarterly, target: > 50)
├── QUALITY GATES
│   ├── CI pass rate (target: > 90% first try)
│   ├── Security findings per deploy (target: 0 critical)
│   └── Contract test pass rate (target: 100%)

LAGGING (Outcome, post-initiative):
├── RELIABILITY
│   ├── Availability (target: 99.95%+)
│   ├── MTTR (target: < 30 min)
│   ├── Incident count (target: -50% YoY)
│   └── Cascade incidents (target: 0)
├── VELOCITY
│   ├── Deploy frequency (target: 50+/day)
│   ├── Lead time (target: < 2 hrs)
│   └── Change failure rate (target: < 2%)
├── COST
│   ├── Infrastructure cost per transaction
│   ├── Cost per service (right-sizing)
│   └── Waste (idle resources, over-provisioned)
└── BUSINESS
    ├── Feature delivery predictability
    ├── Time-to-market for new capabilities
    └── Customer satisfaction (NPS)

DASHBOARD: Shared Grafana/Datadog, reviewed monthly in Architecture Review
```

---

### 11. Actors: "How do you identify all stakeholders for a major architectural change?"

**Answer:**
```
STAKEHOLDER DISCOVERY PROCESS:

1. SYSTEM BOUNDARY ANALYSIS
   - Upstream callers (who calls this API?)
   - Downstream dependencies (what does this call?)
   - Data consumers (who reads this DB/events?)
   - Data providers (who writes to this DB?)

2. ORGANIZATIONAL MAPPING
   - Team topology (Conway's Law): service owners, platform, security
   - Management chain: EM → Director → VP → CTO
   - Cross-functional: Product, Design, Support, Sales, Legal, Compliance

3. RACI WORKSHOP (30 min with Tech Leads)
   For each decision area, assign R/A/C/I:
   - Technical approach
   - Timeline/scope
   - Resource allocation
   - Risk acceptance
   - Rollback criteria

4. VALIDATION QUESTIONS
   "Who would be surprised by this decision?"
   "Who would block this if they knew?"
   "Who has veto power (compliance, legal, security)?"
   "Who owns the runbook for this?"

5. LIVING DOCUMENT
   - Stakeholder map in RFC/ADR
   - Review at each decision gate
   - Add/remove as scope clarifies

TOOL: Stakeholder Map Template (Confluence/GitHub Wiki)
```

---

### 12. Slack: "How do you reduce notification fatigue?"

**Answer:**
```
NOTIFICATION HYGIENE (Default Deny, Opt-In):

INDIVIDUAL SETTINGS (coach team):
- Desktop: Only @mentions + DMs
- Mobile: Only @mentions + DMs + PagerDuty
- Mute: #random, #general, #memes, high-volume channels
- Schedule: Do Not Disturb 10pm-7am, weekends
- Keywords: Alert on "sev-1", "incident", "production", your name

CHANNEL DEFAULTS (enforced by admins):
- @channel: Disabled (use @here rarely)
- @here: Only for time-sensitive coordination
- Threads: Default for all replies (reduce unread count)
- Email notifications: Off (Slack is the inbox)

AUTOMATION OVER NOTIFICATIONS:
- GitHub: Batched daily digest (not per-PR)
- CI: Only on failure (not success)
- Alerts: Deduplicated, grouped, routed to #alert-<system>
- Deploy: Only to #deploy-<env> (not #general)

CULTURE:
- "If it's urgent, call/page — don't Slack"
- "No Slack after hours unless on-call"
- "Read receipts = 👀 reaction, not 'read'"
- Quarterly: Notification audit (what did you mute/unmute?)
```

---

### 13. Communication: "How do you communicate technical risk to non-technical leadership?"

**Answer:**
```
TRANSLATION FRAMEWORK (Technical → Business):

TEMPLATE:
"RISK: [Technical description]
IMPACT: [Business outcome if realized]
PROBABILITY: [Data-driven estimate]
COST TO MITIGATE: [Engineering weeks / $]
COST OF INACTION: [Revenue loss / compliance fine / customer churn]
RECOMMENDATION: [Specific ask with timeline]"

EXAMPLE:
"RISK: Payment service uses deprecated TLS 1.0 (PCI-DSS 4.0 requires 1.2+ by March 2025)
IMPACT: Cannot process payments → $2M/day revenue loss + contract breach
PROBABILITY: 100% (hard deadline, auditor will fail us)
COST TO MITIGATE: 3 eng-weeks ($45K) + 1 week testing
COST OF INACTION: $2M/day + fines + reputation
RECOMMENDATION: Approve 2 engineers for 4 weeks (Sprint 1-4), complete by Jan 31

DECISION NEEDED: Resource allocation by Sprint Planning Friday"

VISUAL AIDS:
- Risk heat map (Probability × Impact)
- Timeline with deadline markers
- Cost comparison (mitigate vs incident)
- Competitor/industry benchmarks
```

---

### 14. GitHub: "How do you handle database migrations in CI/CD?"

**Answer:**
```
DATABASE MIGRATION PIPELINE (Zero-Downtime):

PRINCIPLES:
- Backward compatible (old code works with new schema)
- Reversible (down migration tested)
- Idempotent (safe to re-run)
- Observable (progress, duration, status)

PIPELINE STAGES:

1. PR VALIDATION
   - Migration lint (naming, no DROP COLUMN, no NOT NULL without default)
   - Dry run on staging copy (pg_dump → restore → migrate)
   - Performance check (EXPLAIN ANALYZE, lock duration)

2. STAGING DEPLOY
   - Deploy migration + new code (feature flag OFF)
   - Run migration (track duration, locks)
   - Smoke tests (read/write both schemas)
   - Rollback test (down migration + old code)

3. PRODUCTION CANARY
   - Feature flag: 5% traffic → new code + migrated schema
   - Monitor: error rate, latency, DB locks, replication lag
   - Gradual ramp: 5% → 25% → 50% → 100% (15 min intervals)
   - Auto-rollback on error rate > 1% or p99 > 2x baseline

4. POST-DEPLOY
   - Verify migration complete (no pending)
   - Remove old columns/indexes (next migration, after 2 weeks)
   - Update documentation (schema diagrams, data dictionary)

TOOLING:
- Migration runner: golang-migrate / Flyway / EF Core Migrations
- Lock timeout: SET lock_timeout = '10s'; (prevents stuck)
- Statement timeout: SET statement_timeout = '30s';
- pg_stat_progress: monitor long-running ALTER TABLE
- gh-ost / pt-online-schema-change for massive tables
```

---

### 15. Actors: "How do you run an effective Architecture Review Board?"

**Answer:**
```
ARCHITECTURE REVIEW BOARD (Monthly, 90 min):

COMPOSITION:
- Chair: CTO / Chief Architect
- Members: Domain Architects, Security Lead, Platform Lead, Principal Engineers
- Observers: Tech Leads (rotating), Product (invited)

AGENDA (Fixed):
1. DECISIONS (30 min)
   - 3-5 RFCs ready for decision
   - 5 min each: Author presents BLUF → Board asks clarifying → Decide
   - Output: ACCEPTED / REJECTED / DEFERRED (with reason)

2. RETROSPECTIVES (20 min)
   - 1-2 past decisions: "How did it go?"
   - Metrics: Adoption, incidents, velocity, cost
   - Action: Update standard, revert, double-down

3. STRATEGIC REVIEW (20 min)
   - Technology Radar updates (Adopt/Trial/Assess/Hold)
   - Technical debt portfolio (top 5, investment % )
   - Upcoming decisions (preview next month)

4. GOVERNANCE (20 min)
   - Exception requests (review, approve/deny with conditions)
   - Standards compliance report (SonarQube, Dependabot, CI gates)
   - Team cognitive load survey results

RULES:
- Pre-read: RFCs distributed 48 hrs before
- Decisions recorded in ADR (linked from board minutes)
- Dissenting views documented (not suppressed)
- Action items: Owner + Due Date + Success Criteria
- No "surprise" items — all on agenda 24 hrs prior

OUTPUT:
- Board minutes (Confluence)
- Updated ADRs
- Updated Technology Radar
- Exception log
- Action item tracker (Jira Epic)
```

---

## TOOLS QUICK REFERENCE

| Category | Standard | Alternative | When to Deviate |
|----------|----------|-------------|-----------------|
| **Chat** | Slack | Teams, Mattermost, Discord | Org mandate |
| **Git Hosting** | GitHub | GitLab, Bitbucket, Azure DevOps | Feature parity + cost |
| **CI/CD** | GitHub Actions | GitLab CI, Jenkins, CircleCI | Complex pipelines, self-hosted |
| **Code Review** | GitHub PR | Gerrit, Phabricator | Mandatory line-by-line |
| **Project Mgmt** | GitHub Projects | Jira, Linear, Azure Boards | Complex portfolio |
| **Documentation** | Markdown in Git + Confluence | Notion, GitBook, Wiki | Searchability, versioning |
| **RFC/ADR** | Markdown in `/docs/adr` | Confluence, Notion | Version control + code proximity |
| **Incident Comms** | Slack + PagerDuty | Teams + Opsgenie | On-call tool integration |
| **Dependency Mgmt** | Renovate + Dependabot | WhiteSource, Snyk, FOSSA | Policy engine needs |
| **SBOM** | Syft + CycloneDX | SPDX, in-toto | Customer requirement |