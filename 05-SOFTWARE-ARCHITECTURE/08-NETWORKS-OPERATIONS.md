# NETWORKS & OPERATIONS KNOWLEDGE
## Exam-Style Study Notes

**Roadmap Section:** Networks / Operations Knowledge → LeSS, SAFe, Cloud Providers, TOGAF, Linux/Unix, XP, Service Mesh, Kanban, OSI, Scrum, CI/CD, TCP/IP, Agile, Containers, HTTP/HTTPS, Cloud Design Patterns, Proxies, Firewalls

---

## 1. LESS (LARGE-SCALE SCRUM)

### WHY? (Problem Statement)

**Scrum works for one team. Multiple teams need coordination without bureaucracy.** LeSS = Scrum at scale with minimal extra roles/artifacts.

---

### HOW? (LeSS Principles & Structure)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LESS FRAMEWORK                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LESS PRINCIPLES:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Large-Scale Scrum is Scrum                                       │   │
│  │ • Empirical Process Control (Transparency, Inspection, Adaptation) │   │
│  │ • Whole Product Focus (One Product Backlog, One Product Owner)     │   │
│  │ • Customer-Centric (Not component teams)                           │   │
│  │ • Continuous Improvement Towards Perfection                        │   │
│  │ • Systems Thinking (Optimize whole, not parts)                     │   │
│  │ • Lean Thinking (Eliminate waste, optimize flow)                   │   │
│  │ • Queueing Theory (Manage queues, limit WIP)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LESS STRUCTURE (2-8 Teams):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • ONE Product Owner                                                │   │
│  │ • ONE Product Backlog                                              │   │
│  │ • ONE Definition of Done                                           │   │
│  │ • MULTIPLE Feature Teams (cross-functional, end-to-end)            │   │
│  │ • Scrum Masters (1 per 1-3 teams, servant-leaders)                │   │
│  │ • NO: Project Managers, Component Teams, Architecture Owners       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LESS HUGE (8+ Teams):                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Area Product Owners (subset of backlog)                          │   │
│  │ • Requirement Areas (group of teams working on related features)   │   │
│  │ • Still: One overall Product Owner, One Product Backlog            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LESS EVENTS (Additional to Scrum):                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Overall Sprint Review (All teams + stakeholders)                 │   │
│  │ • Overall Retrospective (Scrum Masters + PO + management)          │   │
│  │ • Multi-Team Sprint Planning (Part 1: What, Part 2: How)           │   │
│  │ • Cross-Team Coordination (Ad-hoc, Communities of Practice)        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (LeSS vs SAFe vs Scrum@Scale)

| Aspect | LeSS | SAFe | Scrum@Scale |
|--------|------|------|-------------|
| **Philosophy** | Minimal, Scrum-only | Prescriptive, heavy | Modular, Scrum-based |
| **Roles** | PO, SM, Teams | Many (RTE, STE, etc.) | Scrum Master, PO, Teams |
| **Artifacts** | Scrum only | Many (PI Plan, etc.) | Scrum + few additions |
| **Planning** | Multi-team Sprint Planning | PI Planning (quarterly) | Scrum of Scrums |
| **Best For** | 2-8 teams, Scrum maturity | Large orgs, compliance | Any size, modular |

---

## 2. SAFE (SCALED AGILE FRAMEWORK)

### WHY? (Problem Statement)

**Large enterprises need structured scaling with governance.** SAFe = comprehensive, prescriptive framework.

---

### HOW? (SAFe 6.0 Core)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SAFe 6.0 CONFIGURATIONS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ESSENTIAL SAFe (Minimum):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Team Level: Scrum/XP/Kanban                                      │   │
│  │ • Program Level: Agile Release Train (ART)                         │   │
│  │   - PI Planning (2-day, quarterly)                                 │   │
│  │   - System Demo, Inspect & Adapt                                   │   │
│  │   - Release Train Engineer (RTE)                                   │   │
│  │   - Product Management, System Architect                           │   │
│  │ • Portfolio Level: Lean Portfolio Management                       │   │
│  │   - Strategy & Investment Funding                                  │   │
│  │   - Agile Portfolio Operations                                     │   │
│  │   - Lean Governance                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LARGE SOLUTION SAFe:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Multiple ARTs + Solution Train                                   │   │
│  │ • Solution Management, Solution Architect                          │   │
│  │ • Solution Train Engineer (STE)                                    │   │
│  │ • Pre/Post-PI Planning                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PORTFOLIO SAFe:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Enterprise Strategy, Investment Funding                          │   │
│  │ • Lean Budget Guardrails, Epic Owners                              │   │
│  │ • Participatory Budgeting, Strategic Themes                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  FULL SAFe: All of the above.                                             │
│                                                                             │
│  SAFe CORE VALUES:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Alignment          • Built-in Quality    • Transparency          │   │
│  │ • Program Execution                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SAFe PRINCIPLES (10):                                                     │
│  1. Take an economic view          6. Visualize and limit WIP           │
│  2. Apply systems thinking         7. Apply cadence, synchronize       │
│  3. Assume variability; preserve options  8. Unlock intrinsic motivation│
│  4. Build incrementally with fast, integrated learning cycles          │
│  5. Base milestones on objective evaluation of working systems         │
│  9. Decentralize decision-making   10. Organize around value           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (SAFe Key Ceremonies)

| Ceremony | Cadence | Participants | Purpose |
|----------|---------|--------------|---------|
| **PI Planning** | Quarterly (2 days) | All ART (50-125 people) | Align on objectives, dependencies |
| **Iteration Planning** | Every 2 weeks | Team | Sprint planning |
| **Daily Standup** | Daily | Team | Sync |
| **System Demo** | End of PI | ART + Stakeholders | Show integrated system |
| **Inspect & Adapt** | End of PI | ART | Retro + Problem solving workshop |
| **Scrum of Scrums** | 2x/week | Scrum Masters | Cross-team coordination |
| **PO Sync** | Weekly | Product Owners | Backlog alignment |
| **Architecture Sync** | Weekly | Architects | Technical alignment |

---

## 3. CLOUD PROVIDERS (AWS / AZURE / GCP)

### WHY? (Problem Statement)

**Cloud is the default platform.** Architects must know core services, patterns, and trade-offs.

---

### HOW? (Core Service Mapping)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLOUD SERVICE MAPPING (Core)                             │
├──────────────────┬──────────────────┬──────────────────┬───────────────────┤
│     CATEGORY     │       AWS        │      AZURE       │       GCP         │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ COMPUTE          │ EC2, Lambda,     │ VMs, Functions,  │ Compute Engine,   │
│                  │ ECS, EKS, Fargate│ Container Apps,  │ Cloud Run, GKE,   │
│                  │                  │ AKS, ACI         │ Cloud Functions   │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ STORAGE          │ S3, EBS, EFS,    │ Blob, Files,     │ Cloud Storage,    │
│                  │ FSx              │ Disks, NetApp    │ Filestore, Persistent│
│                  │                  │                  │ Disk              │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ DATABASE         │ RDS, Aurora,     │ SQL Database,    │ Cloud SQL,        │
│                  │ DynamoDB,        │ Cosmos DB,       │ Firestore,        │
│                  │ ElastiCache,     │ PostgreSQL,      │ Spanner, Bigtable,│
│                  │ Redshift,        │ MySQL, Redis     │ Memorystore,      │
│                  │ DocumentDB       │                  │ BigQuery          │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ NETWORKING       │ VPC, ALB/NLB,    │ VNet, App Gateway,│ VPC, Cloud Load   │
│                  │ CloudFront,      │ Front Door,      │ Balancing, CDN,   │
│                  │ Route 53,        │ Traffic Manager, │ Cloud Armor,      │
│                  │ Direct Connect,  │ DNS, ExpressRoute│ Cloud DNS,        │
│                  │ Transit Gateway  │                  │ Interconnect      │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ MESSAGING        │ SQS, SNS,        │ Service Bus,     │ Pub/Sub,          │
│                  │ EventBridge,     │ Event Grid,      │ Eventarc,         │
│                  │ Kinesis, MQ      │ Event Hubs       │ Dataflow          │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ SECURITY         │ IAM, KMS,        │ Entra ID, Key    │ Cloud IAM, KMS,   │
│                  │ Secrets Manager, │ Vault, Sentinel  │ Secret Manager,   │
│                  │ WAF, Shield,     │                  │ Cloud Armor,      │
│                  │ GuardDuty        │                  │ SCC               │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ OBSERVABILITY    │ CloudWatch,      │ Monitor,         │ Cloud Monitoring, │
│                  │ X-Ray, CloudTrail│ App Insights,    │ Cloud Trace,      │
│                  │                  │ Log Analytics    │ Cloud Logging     │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ IaC              │ CloudFormation,  │ Bicep, ARM,      │ Deployment Manager,│
│                  │ CDK, Terraform   │ Terraform        │ Terraform         │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ CONTAINERS       │ ECS, EKS, Fargate│ ACI, AKS,        │ Cloud Run, GKE,   │
│                  │                  │ Container Apps   │ Autopilot         │
└──────────────────┴──────────────────┴──────────────────┴───────────────────┘
```

---

### WHAT? (Architect's Cloud Decision Framework)

```
CLOUD SELECTION CRITERIA:
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. EXISTING INVESTMENT & SKILLS                                            │
│     • Team expertise, certifications, current workloads                     │
│                                                                             │
│  2. SERVICE MATURITY FOR USE CASE                                           │
│     • Kubernetes: EKS (mature) vs AKS (integrated) vs GKE (innovative)     │
│     • Serverless: Lambda (mature) vs Functions (integrated) vs Cloud Run   │
│     • Data: Aurora (PostgreSQL/MySQL) vs Cosmos DB (multi-model) vs Spanner│
│                                                                             │
│  3. HYBRID / MULTI-CLOUD STRATEGY                                           │
│     • Anthos (GCP), Arc (Azure), Outposts (AWS)                            │
│     • Crossplane, Terraform for multi-cloud                                │
│                                                                             │
│  4. COMPLIANCE & DATA RESIDENCY                                             │
│     • Regions, Sovereign clouds (GovCloud, Azure Government, Assured)      │
│                                                                             │
│  5. COST OPTIMIZATION                                                       │
│     • Reserved Instances, Savings Plans, Committed Use Discounts           │
│     • Spot/Preemptible for batch, FinOps tooling                           │
│                                                                             │
│  6. VENDOR LOCK-IN ASSESSMENT                                               │
│     • Portable: Kubernetes, Terraform, OpenTelemetry                       │
│     • Locked: Lambda@Edge, Cloud Functions, Cosmos DB, DynamoDB            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. TOGAF (See Frameworks Section - Cross-Reference)

---

## 5. LINUX/UNIX FUNDAMENTALS

### WHY? (Problem Statement)

**Everything runs on Linux.** Architects must understand OS primitives for performance, security, debugging.

---

### HOW? (Essential Linux Knowledge)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LINUX/UNIX ESSENTIALS FOR ARCHITECTS                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROCESS MANAGEMENT:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • ps, top, htop, pidstat — Process inspection                      │   │
│  │ • nice, renice — Priority                                          │   │
│  │ • systemctl — Service management (systemd)                         │   │
│  │ • cgroups — Resource limits (CPU, memory, I/O)                     │   │
│  │ • namespaces — Isolation (PID, NET, MNT, UTS, IPC, USER)           │   │
│  │ • /proc filesystem — Runtime process info                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MEMORY MANAGEMENT:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • free, vmstat, /proc/meminfo                                      │   │
│  │ • Virtual vs Physical vs RSS vs Shared                             │   │
│  │ • Page cache, buffer cache, slab                                   │   │
│  │ • Swap (swappiness, zram)                                          │   │
│  │ • OOM Killer (oom_score_adj)                                       │   │
│  │ • Huge pages (transparent, explicit)                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  FILE SYSTEM & I/O:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • ext4, xfs, btrfs, zfs — File systems                            │   │
│  │ • iostat, iotop, fio — I/O analysis                               │   │
│  │ • Mount options (noatime, nodiratime, discard)                    │   │
│  │ • Inodes, dentry cache, page cache                                │   │
│  │ • LVM, RAID, mdadm, dm-crypt/LUKS                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  NETWORKING:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • ip, ss, netstat, nmap, tcpdump, wireshark                       │   │
│  │ • iptables/nftables, firewalld, ufw                               │   │
│  │ • Network namespaces, veth, bridge, vxlan, wireguard             │   │
│  │ • /proc/net, /sys/class/net                                       │   │
│  │ • systemd-resolved, NetworkManager                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PERFORMANCE TOOLS:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • perf — CPU profiling, hardware counters                         │   │
│  │ • bpftrace / bcc — eBPF tracing (kernel + user)                   │   │
│  │ • strace / ltrace — Syscall/library tracing                       │   │
│  │ • flamegraph — Visualization                                      │   │
│  │ • sar, collectl — System activity reporter                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONTAINER INTERNALS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • runc, containerd, CRI-O                                         │   │
│  │ • OverlayFS, snapshotters                                         │   │
│  │ • seccomp, AppArmor, SELinux                                      │   │
│  │ • cgroups v2 (unified hierarchy)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Essential Commands Cheat Sheet)

| Task | Command |
|------|---------|
| **Find process by port** | `ss -tlnp \| grep :8080` or `lsof -i :8080` |
| **CPU per process** | `pidstat -u 1` or `top -H -p <pid>` |
| **Memory map** | `pmap -x <pid>` or `cat /proc/<pid>/smaps` |
| **Disk I/O by process** | `iotop -o` or `pidstat -d 1` |
| **Network connections** | `ss -tunap` or `netstat -tunap` |
| **Packet capture** | `tcpdump -i any -w capture.pcap port 8080` |
| **System calls** | `strace -f -p <pid> -e trace=network` |
| **eBPF trace** | `bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'` |
| **Disk usage** | `df -h`, `du -sh * \| sort -hr` |
| **Inode usage** | `df -i` |
| **Kernel logs** | `dmesg -T`, `journalctl -xe`, `journalctl -u <service>` |

---

## 6. XP (EXTREME PROGRAMMING)

### WHY? (Problem Statement)

**Engineering practices for sustainable, high-quality delivery.** XP = technical excellence + customer satisfaction.

---

### HOW? (XP Practices)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         XP PRACTICES (12 Core)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. PLANNING GAME — Customer + Dev plan together (stories, estimates)     │
│  2. SMALL RELEASES — Frequent, incremental (every 1-3 weeks)              │
│  3. METAPHOR — Shared system metaphor for communication                    │
│  4. SIMPLE DESIGN — Passes tests, no duplication, expresses intent        │
│  5. TEST-DRIVEN DEVELOPMENT (TDD) — Red-Green-Refactor                    │
│  6. REFACTORING — Continuous improvement without changing behavior         │
│  7. PAIR PROGRAMMING — Two devs, one machine (knowledge transfer, quality)│
│  8. COLLECTIVE OWNERSHIP — Anyone can change any code                      │
│  9. CONTINUOUS INTEGRATION — Integrate multiple times/day                 │
│  10. 40-HOUR WEEK — Sustainable pace, no overtime                         │
│  11. ON-SITE CUSTOMER — Real customer available to team                   │
│  12. CODING STANDARDS — Consistent style, automated formatting            │
│                                                                             │
│  XP VALUES:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Communication    • Simplicity    • Feedback    • Courage          │   │
│  │ • Respect                                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  XP IN MODERN CONTEXT:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • TDD → Universal (not just XP)                                    │   │
│  │ • Pair Programming → Mob Programming, Ensemble Programming         │   │
│  │ • Continuous Integration → CI/CD Pipelines                         │   │
│  │ • Collective Ownership → Git, Code Review, Shared Standards        │   │
│  │ • 40-Hour Week → Sustainable Pace, WIP Limits, No Hero Culture    │   │
│  │ • Coding Standards → Linters, Formatters, Pre-commit Hooks         │   │
│  │ • Simple Design → YAGNI, Evolutionary Architecture                │   │
│  │ • Small Releases → Continuous Delivery, Feature Flags              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. SERVICE MESH

### WHY? (Problem Statement)

**Microservices need cross-cutting concerns without code changes.** Service Mesh = infrastructure layer for service-to-service communication.

---

### HOW? (Service Mesh Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DATA PLANE (Sidecar Proxies):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Deployed per pod (sidecar) or per node (daemonset)              │   │
│  │ • Intercepts all inbound/outbound traffic                          │   │
│  │ • Envoy (most common), Linkerd-proxy, MOSN                        │   │
│  │ • Handles: mTLS, load balancing, retries, timeouts, routing       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONTROL PLANE:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Configuration distribution (xDS APIs: LDS, CDS, RDS, EDS)       │   │
│  │ • Certificate management (SPIFFE/SPIRE integration)               │   │
│  │ • Policy enforcement (AuthZ, rate limit, quota)                   │   │
│  │ • Telemetry collection (metrics, traces, access logs)             │   │
│  │ • Istiod (Istio), Linkerd Control Plane, Consul Control Plane     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY CAPABILITIES:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ TRAFFIC MANAGEMENT:                                                  │   │
│  │ • Traffic splitting (canary, blue/green, mirroring)               │   │
│  │ • Fault injection (delay, abort)                                   │   │
│  │ • Retries, timeouts, circuit breaking                              │   │
│  │ • Request/response transformation                                 │   │
│  │                                                                    │   │
│  │ SECURITY:                                                            │   │
│  │ • mTLS (auto cert rotation via SPIFFE/SPIRE)                      │   │
│  │ • Peer authentication (JWT, mTLS)                                  │   │
│  │ • Authorization (OPA, RBAC, custom)                                │   │
│  │ • Certificate rotation (short-lived, 24h)                          │   │
│  │                                                                    │   │
│  │ OBSERVABILITY:                                                       │   │
│  │ • Distributed tracing (OpenTelemetry, Zipkin, Jaeger)             │   │
│  │ • Metrics (Prometheus: request rate, latency, errors)             │   │
│  │ • Access logs (structured, correlated)                             │   │
│  │                                                                    │   │
│  │ RESILIENCE:                                                          │   │
│  │ • Retries with backoff, timeouts, circuit breakers                │   │
│  │ • Bulkhead (connection pool limits)                                │   │
│  │ • Outlier detection (eject unhealthy endpoints)                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  POPULAR IMPLEMENTATIONS:                                                  │
│  ┌──────────────┬────────────────┬──────────────────────────────────────┐ │
│  │ Istio        │ Feature-rich   │ Complex, resource-heavy, Envoy       │ │
│  │              │ (traffic, sec, │ based, large community, CNCF         │ │
│  │              │  obs, policy)  │ graduated                           │ │
│  ├──────────────┼────────────────┼──────────────────────────────────────┤ │
│  │ Linkerd      │ Lightweight,   │ Rust proxy, simpler, faster,        │ │
│  │              │ security-focus │ CNCF graduated                      │ │
│  ├──────────────┼────────────────┼──────────────────────────────────────┤ │
│  │ Consul Connect│ HashiCorp     │ Multi-runtime, VM + K8s,            │ │
│  │              │ ecosystem      │ Built-in service discovery          │ │
│  ├──────────────┼────────────────┼──────────────────────────────────────┤ │
│  │ Cilium       │ eBPF-based,    │ High performance, no sidecar        │ │
│  │              │ no-sidecar     │ (ambient mode), network policy      │ │
│  ├──────────────┼────────────────┼──────────────────────────────────────┤ │
│  │ AWS App Mesh │ AWS managed    │ Envoy, integrates with ECS/EKS      │ │
│  │              │                │ App Mesh Controller for K8s         │ │
│  └──────────────┴────────────────┴──────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (When to Adopt Service Mesh)

```
DECISION MATRIX:
┌─────────────────────────────────────────────────────────────────────────────┐
│  ADOPT SERVICE MESH WHEN:                                                   │
│  ✓ 20+ services with complex communication                                │
│  ✓ mTLS required (compliance, zero trust)                                 │
│  ✓ Advanced traffic management (canary, mirroring, fault injection)       │
│  ✓ Uniform observability without code changes                             │
│  ✓ Centralized resilience policies (retry, timeout, CB)                  │
│  ✓ Multi-cluster / hybrid cloud                                           │
│  ✓ Platform team exists to operate mesh                                   │
│                                                                             │
│  DON'T ADOPT WHEN:                                                          │
│  ✗ < 10 services                                                           │
│  ✗ Simple communication patterns                                           │
│  ✗ Team lacks platform expertise                                           │
│  ✗ Latency budget extremely tight (sidecar adds 1-2ms)                   │
│  ✗ No platform team to operate                                             │
│                                                                             │
│  ALTERNATIVES:                                                              │
│  • Library-based (Polly, gRPC interceptors, OTEL SDKs)                   │
│  • API Gateway + Libraries for east-west                                  │
│  • Sidecar-less (Cilium, Istio Ambient) — Lower overhead                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. KANBAN

### WHY? (Problem Statement)

**Visualize work, limit WIP, manage flow.** Kanban = evolutionary improvement for any process.

---

### HOW? (Kanban Core Practices)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KANBAN METHOD                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  6 CORE PRACTICES:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. VISUALIZE — Board with columns (To Do → Doing → Done)          │   │
│  │ 2. LIMIT WORK IN PROGRESS (WIP) — Explicit limits per column      │   │
│  │ 3. MANAGE FLOW — Monitor lead time, cycle time, throughput        │   │
│  │ 4. MAKE POLICIES EXPLICIT — Definition of Ready, Done, WIP rules  │   │
│  │ 5. IMPLEMENT FEEDBACK LOOPS — Daily standup, retrospectives       │   │
│  │ 6. IMPROVE COLLABORATIVELY, EVOLVE EXPERIMENTALLY — Kaizen        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KANBAN METRICS:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • LEAD TIME — Request to delivery (customer perspective)          │   │
│  │ • CYCLE TIME — Start work to delivery (team perspective)          │   │
│  │ • THROUGHPUT — Items completed per time unit                       │   │
│  │ • WIP — Items in progress                                          │   │
│  │ • FLOW EFFICIENCY — Value-add time / Lead time                    │   │
│  │ • CUMULATIVE FLOW DIAGRAM — WIP over time, bottlenecks            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KANBAN vs SCRUM:                                                          │
│  ┌─────────────────┬─────────────────┬─────────────────────────────────┐  │
│  │ Aspect          │ Scrum           │ Kanban                          │  │
│  ├─────────────────┼─────────────────┼─────────────────────────────────┤  │
│  │ Cadence         │ Fixed sprints   │ Continuous flow                 │  │
│  │ Roles           │ PO, SM, Team    │ No required roles               │  │
│  │ Commitment      │ Sprint goal     │ WIP limits                      │  │
│  │ Changes         │ Sprint boundary │ Anytime (if WIP allows)         │  │
│  │ Metrics         │ Velocity        │ Lead/Cycle time, Throughput     │  │
│  │ Best For        │ Product dev     │ Support, Ops, Maintenance       │  │
│  └─────────────────┴─────────────────┴─────────────────────────────────┘  │
│                                                                             │
│  SCRUMBAN (Hybrid):                                                        │
│  • Sprint cadence for planning/review                                     │
│  • Kanban board + WIP limits within sprint                               │
│  • Continuous flow within sprint                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. OSI MODEL

### WHY? (Problem Statement)

**Network troubleshooting and design require layer thinking.** OSI = conceptual framework for network communication.

---

### HOW? (OSI 7 Layers - Top Down for Architects)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OSI MODEL (7 Layers)                                │
├──────┬──────────────────┬──────────────────────────────────────────────────┤
│ LAYER│ NAME             │ RESPONSIBILITY                                   │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  7   │ APPLICATION      │ User-facing protocols (HTTP, gRPC, DNS, SMTP)   │
│      │                  │ APIs, authentication, encryption (TLS)          │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  6   │ PRESENTATION     │ Data formatting (JSON, XML, Protobuf, Avro)     │
│      │                  │ Encryption/decryption, compression              │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  5   │ SESSION          │ Connection management (setup, teardown, sync)   │
│      │                  │ RPC sessions, WebSocket, TLS handshake          │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  4   │ TRANSPORT        │ End-to-end delivery (TCP, UDP, QUIC)            │
│      │                  │ Reliability, flow control, congestion control   │
│      │                  │ Ports (source/destination)                      │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  3   │ NETWORK          │ Routing, addressing (IP, ICMP, IPsec)           │
│      │                  │ Logical addressing (IPv4/IPv6), fragmentation   │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  2   │ DATA LINK        │ Frame formatting, error detection (Ethernet)    │
│      │                  │ MAC addressing, VLANs, switches                 │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  1   │ PHYSICAL         │ Bits on wire (cables, fiber, radio, voltages)  │
│      │                  │ Cables (Cat6, fiber), WiFi, 5G, transceivers    │
└──────┴──────────────────┴──────────────────────────────────────────────────┘

ENCAPSULATION (Top → Down):
Application Data
    → Presentation (encode/encrypt)
        → Session (session header)
            → Transport (TCP/UDP header: ports, seq, ack)
                → Network (IP header: src/dst IP)
                    → Data Link (Ethernet header: MAC, FCS)
                        → Physical (bits)

ARCHITECT'S VIEW (Where decisions happen):
• Layer 7: API design, auth, rate limiting, caching
• Layer 6: Serialization (Protobuf vs JSON), compression, TLS termination
• Layer 5: Session affinity, connection pooling, WebSocket
• Layer 4: Load balancing (L4 vs L7), TCP tuning, QUIC
• Layer 3: VPC design, subnets, routing, VPN, IPsec
• Layer 2: VLANs, MTU, jumbo frames, L2 security
• Layer 1: Bandwidth, latency, physical redundancy
```

---

## 10. SCRUM

### WHY? (Problem Statement)

**Framework for complex product development.** Scrum = empirical process control for product teams.

---

### HOW? (Scrum Framework - 2020 Guide)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SCRUM FRAMEWORK                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SCRUM TEAM (3 Roles):                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PRODUCT OWNER — Maximizes value, owns Product Backlog            │   │
│  │ • SCRUM MASTER — Servant leader, facilitates Scrum, removes impediments│   │
│  │ • DEVELOPERS — Cross-functional, self-managing, create Done increment│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SCRUM EVENTS (Time-boxed):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • SPRINT (≤1 month) — Container for all other events               │   │
│  │ • SPRINT PLANNING (8h max) — What + How                           │   │
│  │ • DAILY SCRUM (15 min) — Inspect progress toward Sprint Goal      │   │
│  │ • SPRINT REVIEW (4h max) — Inspect outcome, adapt backlog         │   │
│  │ • SPRINT RETROSPECTIVE (3h max) — Inspect process, improve        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SCRUM ARTIFACTS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PRODUCT BACKLOG — Ordered list of everything needed              │   │
│  │ • SPRINT BACKLOG — Sprint Goal + selected PBIs + plan             │   │
│  │ • INCREMENT — Sum of Done PBIs (usable, potentially releasable)   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMITMENTS:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PRODUCT GOAL — Long-term objective for Product Backlog           │   │
│  │ • SPRINT GOAL — Single objective for the Sprint                    │   │
│  │ • DEFINITION OF DONE — Shared understanding of "Done"             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SCRUM VALUES:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Commitment    • Focus    • Openness    • Respect    • Courage     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. CI/CD

### WHY? (Problem Statement)

**Automate integration, testing, delivery.** CI/CD = fast feedback, reliable releases.

---

### HOW? (CI/CD Pipeline Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CI/CD PIPELINE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONTINUOUS INTEGRATION (CI):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ TRIGGER: Push to branch, PR opened/updated                          │   │
│  │                                                                       │   │
│  │ STAGES (Parallel where possible):                                    │   │
│  │ 1. CHECKOUT & SETUP — Cache dependencies                             │   │
│  │ 2. STATIC ANALYSIS — Lint, format, type check, SAST (CodeQL)        │   │
│  │ 3. UNIT TESTS — Fast, isolated, >80% coverage                       │   │
│  │ 4. BUILD — Compile, package (Docker image, JAR, binary)             │   │
│  │ 5. CONTAINER SCAN — Trivy, Grype (vulnerabilities, secrets)         │   │
│  │ 6. INTEGRATION TESTS — Testcontainers (DB, Kafka, Redis)           │   │
│  │ 7. CONTRACT TESTS — Consumer-driven (Pact), Provider verification   │   │
│  │ 8. SECURITY SCAN — SCA (Dependabot, Snyk), SAST, Secrets            │   │
│  │ 9. POLICY CHECK — OPA/Gatekeeper (no privileged, resource limits)   │   │
│  │ 10. ARTIFACT PUBLISH — Registry (Docker, Maven, npm, NuGet)        │   │
│  │ 11. SBOM GENERATION — Syft (CycloneDX/SPDX)                        │   │
│  │                                                                       │   │
│  │ GATES: All must pass → Merge allowed                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONTINUOUS DELIVERY (CD):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ TRIGGER: Merge to main, tag push, manual                            │   │
│  │                                                                       │   │
│  │ ENVIRONMENTS (Promotion):                                            │   │
│  │ 1. DEV / CI — Auto-deploy on merge (smoke test)                     │   │
│  │ 2. STAGING — Auto or manual (full integration, E2E tests)          │   │
│  │ 3. PRODUCTION — Manual approval (or auto with feature flags)        │   │
│  │                                                                       │   │
│  │ DEPLOYMENT STRATEGIES:                                               │   │
│  │ • BLUE/GREEN — Two identical environments, switch traffic           │   │
│  │ • CANARY — Gradual traffic shift (5% → 25% → 100%)                  │   │
│  │ • ROLLING — Incremental pod replacement (K8s default)              │   │
│  │ • FEATURE FLAGS — Deploy dark, enable via config                    │   │
│  │                                                                       │   │
│  │ POST-DEPLOY:                                                         │   │
│  │ • Smoke tests / Synthetic monitoring                                 │   │
│  │ • Auto-rollback on error rate / latency SLO breach                  │   │
│  │ • Notification (Slack, PagerDuty)                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GITOPS (CD via Git):                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Git = Desired state (infra + app)                                 │   │
│  │ • ArgoCD / Flux syncs cluster to Git                                │   │
│  │ • PR-based changes, audit trail, rollback = git revert             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (CI/CD Best Practices)

| Practice | Why |
|----------|-----|
| **Fast feedback** | CI < 10 min, fail fast |
| **Test pyramid** | 70% unit, 20% integration, 10% E2E |
| **Immutable artifacts** | Build once, promote same artifact |
| **Version everything** | SemVer, Git tags, container tags |
| **Automate rollback** | Auto on metric breach |
| **Security left** | SAST/SCA/Secrets in CI |
| **Policy as code** | OPA/Gatekeeper in CI + Admission |
| **Observability** | Pipeline metrics, deployment tracking |

---

## 12. TCP/IP

### WHY? (Problem Statement)

**Internet runs on TCP/IP.** Architects need protocol understanding for performance, security, troubleshooting.

---

### HOW? (TCP/IP Suite - 4 Layers)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TCP/IP MODEL (4 Layers)                             │
├──────┬──────────────────┬──────────────────────────────────────────────────┤
│ LAYER│ NAME             │ PROTOCOLS / RESPONSIBILITY                        │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  4   │ APPLICATION      │ HTTP/HTTPS, gRPC, DNS, SMTP, SSH, FTP, DHCP,    │
│      │                  │ SNMP, Syslog, NTP, QUIC (HTTP/3)                │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  3   │ TRANSPORT        │ TCP (Reliable, ordered, flow/congestion control)│
│      │                  │ UDP (Unreliable, fast, no ordering)             │
│      │                  │ QUIC (UDP-based, multiplexed, encrypted)        │
│      │                  │ Ports (0-65535: Well-known, Registered, Dynamic)│
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  2   │ INTERNET         │ IP (v4/v6), ICMP, ICMPv6, IPsec, ARP, NDP       │
│      │                  │ Routing, fragmentation, addressing              │
│      │                  │ IPv4: 32-bit, NAT, CIDR                         │
│      │                  │ IPv6: 128-bit, no NAT, NDP, SLAAC               │
├──────┼──────────────────┼──────────────────────────────────────────────────┤
│  1   │ NETWORK ACCESS   │ Ethernet, WiFi, PPP, DSL, Fiber, 5G            │
│      │ (Link/Physical)  │ MAC addressing, framing, error detection        │
└──────┴──────────────────┴──────────────────────────────────────────────────┘

KEY TCP CONCEPTS:
┌─────────────────────────────────────────────────────────────────────────────┐
│ THREE-WAY HANDSHAKE:                                                        │
│ SYN → SYN-ACK → ACK                                                         │
│                                                                             │
│ CONGESTION CONTROL:                                                         │
│ • Slow Start → Congestion Avoidance → Fast Recovery → Fast Retransmit     │
│ • CUBIC (Linux default), BBR (Google, better for high latency)            │
│                                                                             │
│ FLOW CONTROL:                                                               │
• Receiver window (rwnd) — Advertised in TCP header                          │
• Sender window = min(cwnd, rwnd)                                            │
                                                                             │
TCP TUNING (Linux):                                                           │
• net.ipv4.tcp_congestion_control = bbr                                     │
• net.ipv4.tcp_slow_start_after_idle = 0                                    │
• net.ipv4.tcp_tw_reuse = 1                                                 │
• net.core.somaxconn = 65535                                                │
• net.ipv4.tcp_fin_timeout = 15                                             │
└─────────────────────────────────────────────────────────────────────────────┘

KEY IP CONCEPTS:
┌─────────────────────────────────────────────────────────────────────────────┐
│ IPV4: 32-bit, ~4.3B addresses, NAT, Private ranges (10/8, 172.16/12, 192.168/16)│
│ IPV6: 128-bit, 340 undecillion, No NAT, NDP, SLAAC, Privacy extensions   │
│ CIDR: /24 = 256 IPs, /16 = 65K, /8 = 16M                                 │
│ SUBNETTING: VPC design, AZs, Public/Private/DB subnets                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. AGILE

### WHY? (Problem Statement)

**Mindset over methodology.** Agile = values + principles for adaptive delivery.

---

### HOW? (Agile Manifesto + Principles)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGILE MANIFESTO                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WE VALUE:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Individuals and interactions  OVER  Processes and tools            │   │
│  │ Working software              OVER  Comprehensive documentation     │   │
│  │ Customer collaboration        OVER  Contract negotiation            │   │
│  │ Responding to change          OVER  Following a plan               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  12 PRINCIPLES:                                                            │
│  1. Customer satisfaction through early, continuous delivery              │
│  2. Welcome changing requirements, even late in development               │
│  3. Deliver working software frequently (weeks, not months)              │
│  4. Business people and developers work together daily                    │
│  5. Build projects around motivated individuals, trust them              │
│  6. Face-to-face conversation (most effective communication)             │
│  7. Working software is the primary measure of progress                  │
│  8. Sustainable development (constant pace indefinitely)                 │
│  9. Continuous attention to technical excellence, good design            │
│  10. Simplicity — maximize work not done                                  │
│  11. Self-organizing teams (best architectures, requirements, designs)   │
│  12. Regular reflection and adaptation (retrospectives)                  │
│                                                                             │
│  AGILE ≠ SCRUM ≠ KANBAN ≠ XP:                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Agile = Mindset + Values + Principles                               │   │
│  │ Scrum = Framework (roles, events, artifacts)                        │   │
│  │ Kanban = Method (visualize, limit WIP, manage flow)                │   │
│  │ XP = Practices (TDD, pairing, CI, simple design)                   │   │
│  │                                                                       │   │
│  │ YOU CAN: Scrum + Kanban (Scrumban), Scrum + XP, Kanban + XP        │   │
│  │ PICK practices that serve Agile values in YOUR context             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. CONTAINERS

### WHY? (Problem Statement)

**Standardized packaging for consistent deployment.** Containers = process isolation + filesystem packaging.

---

### HOW? (Container Internals & Orchestration)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CONTAINER ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONTAINER RUNTIME:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • runc (OCI reference) — Low-level container execution            │   │
│  │ • containerd — Image management, snapshotters, CRI plugin         │   │
│  │ • CRI-O — Kubernetes CRI implementation, lighter                  │   │
│  │ • crun — Rust implementation, faster, smaller                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  IMAGE SPEC (OCI):                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Manifest — Config + Layer digests                                │   │
│  │ • Config — Entrypoint, Cmd, Env, Labels, User, Volumes            │   │
│  │ • Layers — Tarballs (diff from parent), content-addressable        │   │
│  │ • Content-addressable (SHA256) — Immutable, deduplicated          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ISOLATION PRIMITIVES (Linux):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • NAMESPACES — PID, NET, MNT, UTS, IPC, USER, CGROUP              │   │
│  │ • CGROUPS v2 — CPU, Memory, I/O, PIDs limits (unified hierarchy)  │   │
│  │ • SECCOMP — Syscall filtering (profiles: default, custom)         │   │
│  │ • CAPABILITIES — Fine-grained root privileges (drop ALL, add needed)│   │
│  │ • SELINUX/APPARMOR — MAC (Mandatory Access Control)               │   │
│  │ • OVERLAYFS — Copy-on-write layered filesystem                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KUBERNETES (Orchestration):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ CONTROL PLANE:                                                       │   │
│  │ • kube-apiserver — REST API, auth, validation                       │   │
│  │ • kube-scheduler — Pod placement (affinity, taints, resources)    │   │
│  │ • kube-controller-manager — ReplicaSet, Deployment, Service, etc.  │   │
│  │ • etcd — Distributed key-value store (state)                       │   │
│  │ • cloud-controller-manager — Cloud integration                     │   │
│  │                                                                       │   │
│  │ WORKER NODE:                                                         │   │
│  │ • kubelet — Node agent, pod lifecycle                               │   │
│  │ • kube-proxy — Service networking (IPVS/iptables)                 │   │
│  │ • Container Runtime (containerd/CRI-O)                             │   │
│  │ • CNI Plugin (Calico, Cilium, Flannel, Weave)                     │   │
│  │ • CSI Driver (Storage)                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 15. HTTP/HTTPS

### WHY? (Problem Statement)

**Web communication protocol.** HTTP = application layer protocol for distributed systems.

---

### HOW() (HTTP Evolution & Best Practices)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HTTP EVOLUTION                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HTTP/1.0 (1996): One connection per request, no keep-alive               │
│  HTTP/1.1 (1997): Keep-alive, pipelining (broken), chunked encoding       │
│  HTTP/2 (2015): Binary framing, multiplexing, header compression (HPACK), │
│                  server push, stream prioritization                        │
│  HTTP/3 (2022): QUIC (UDP), 0-RTT, connection migration, no head-of-line  │
│                  blocking, mandatory encryption                            │
│                                                                             │
│  HTTP METHODS SEMANTICS:                                                   │
│  ┌────────────┬─────────────┬──────────┬──────────┬────────────────────┐  │
│  │ Method     │ Safe        │ Idempotent│ Cacheable│ Purpose           │  │
│  ├────────────┼─────────────┼──────────┼──────────┼────────────────────┤  │
│  │ GET        │ ✅          │ ✅       │ ✅       │ Retrieve          │  │
│  │ HEAD       │ ✅          │ ✅       │ ✅       │ Metadata only     │  │
│  │ POST       │ ❌          │ ❌       │ ❌*      │ Create/Action     │  │
│  │ PUT        │ ❌          │ ✅       │ ❌       │ Replace (full)    │  │
│  │ PATCH      │ ❌          │ ❌*      │ ❌       │ Partial update    │  │
│  │ DELETE     │ ❌          │ ✅       │ ❌       │ Remove            │  │
│  │ OPTIONS    │ ✅          │ ✅       │ ❌       │ Capabilities      │  │
│  │ TRACE      │ ✅          │ ✅       │ ❌       │ Debug             │  │
│  └────────────┴─────────────┴──────────┴──────────┴────────────────────┘  │
│  * POST can be idempotent with Idempotency-Key; PATCH can be idempotent  │
│                                                                             │
│  STATUS CODES (Key for Architects):                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1xx Informational     2xx Success       3xx Redirection           │   │
│  │ 100 Continue          200 OK            301 Moved Permanently     │   │
│  │ 101 Switching         201 Created       302 Found                 │   │
│  │                       202 Accepted      304 Not Modified          │   │
│  │                       204 No Content    307 Temporary Redirect    │   │
│  │                       206 Partial       308 Permanent Redirect    │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ 4xx Client Error      5xx Server Error                              │   │
│  │ 400 Bad Request       500 Internal Server Error                     │   │
│  │ 401 Unauthorized      502 Bad Gateway                               │   │
│  │ 403 Forbidden         503 Service Unavailable                       │   │
│  │ 404 Not Found         504 Gateway Timeout                           │   │
│  │ 409 Conflict          501 Not Implemented                           │   │
│  │ 422 Unprocessable     502/503/504 — Retryable                      │   │
│  │ 429 Too Many Requests                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  HTTPS / TLS:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • TLS 1.3 (mandatory for HTTP/3, recommended for all)             │   │
│  │ • Certificate: X.509, Let's Encrypt (ACME), mTLS                  │   │
│  │ • HSTS (Strict Transport Security), HPKP (deprecated)            │   │
│  │ • Certificate pinning (mobile), Certificate Transparency          │   │
│  │ • TLS termination at Gateway/Load Balancer (mTLS to services)     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 16. CLOUD DESIGN PATTERNS

### WHY? (Problem Statement)

**Reusable solutions for cloud-native challenges.** Cloud Design Patterns = proven architectures.

---

### HOW() (Key Patterns by Category)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLOUD DESIGN PATTERNS (Azure/AWS Catalog)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  AVAILABILITY & RESILIENCE:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • CIRCUIT BREAKER — Fail fast, prevent cascade                     │   │
│  │ • RETRY — Exponential backoff + jitter                            │   │
│  │ • TIMEOUT — Bound wait time                                        │   │
│  │ • BULKHEAD — Isolate critical resources                           │   │
│  │ • LEADER ELECTION — Coordinate distributed instances              │   │
│  │ • HEALTH ENDPOINT MONITORING — Liveness/readiness probes          │   │
│  │ • THROTTLING — Protect downstream services                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA MANAGEMENT:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • CACHE-ASIDE — Lazy load, TTL + invalidation                     │   │
│  │ • CQRS — Separate read/write models                               │   │
│  │ • EVENT SOURCING — Append-only event log                          │   │
│  │ • MATERIALIZED VIEW — Pre-computed query results                  │   │
│  │ • SHARDING — Horizontal partitioning                              │   │
│  │ • EVENTUAL CONSISTENCY — Accept stale reads for scale             │   │
│  │ • DATA PARTITIONING — Vertical/horizontal/functional              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MESSAGING & COMMUNICATION:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • COMPETING CONSUMERS — Work distribution                          │   │
│  │ • PUB/SUB — Event broadcasting                                    │   │
│  │ • PRIORITY QUEUE — Urgent first                                   │   │
│  │ • SCHEDULER AGENT SUPERVISOR — Distributed scheduling             │   │
│  │ • CLAIM CHECK — Large payloads via blob storage + reference       │   │
│  │ • IDempotency — Safe retries                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DESIGN & DEPLOYMENT:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • SIDECAR — Helper container (logging, proxy, config)             │   │
│  │ • AMBASSADOR — Proxy for legacy/external services                 │   │
│  │ • ANTI-CORRUPTION LAYER — Translate between domains               │   │
│  │ • BACKENDS FOR FRONTENDS (BFF) — Per-client API                   │   │
│  │ • STRANGLER FIG — Incremental migration                           │   │
│  │ • DEPLOYMENT STAMPS — Isolated deployment units                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SECURITY:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • GATEKEEPER — Policy enforcement (OPA)                           │   │
│  │ • VAULT — Secrets management                                      │   │
│  │ • IDENTITY PROVIDER — Centralized auth (OIDC, SAML)              │   │
│  │ • ZERO TRUST NETWORK — mTLS, micro-segmentation                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MANAGEMENT & MONITORING:                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • HEALTH ENDPOINT — Liveness/readiness/startup                     │   │
│  │ • INSTRUMENTATION — OpenTelemetry (metrics, traces, logs)        │   │
│  │ • LOG AGGREGATION — Centralized structured logging               │   │
│  │ • DISTRIBUTED TRACING — W3C TraceContext, OpenTelemetry          │   │
│  │ • AUDIT LOGGING — Immutable, tamper-evident                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 17. PROXIES

### WHY? (Problem Statement)

**Intermediaries for control, security, performance.** Proxies = forward, reverse, sidecar.

---

### HOW() (Proxy Types & Use Cases)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROXY TYPES                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FORWARD PROXY (Client → Internet):                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Client configures proxy                                         │   │
│  │ • Use cases: Egress control, caching, anonymity, content filtering│   │
│  │ • Examples: Squid, NGINX (forward), AWS NAT Gateway               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  REVERSE PROXY (Internet → Server):                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Server-side, clients unaware                                     │   │
│  │ • Use cases: Load balancing, SSL termination, caching, WAF,       │   │
│  │   compression, rate limiting, A/B testing                          │   │
│  │ • Examples: NGINX, HAProxy, Envoy, AWS ALB, Cloudflare, Kong      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SIDECAR PROXY (Service Mesh):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Per-pod, intercepts all traffic                                 │   │
│  │ • mTLS, observability, resilience, routing                        │   │
│  │ • Envoy (Istio, Consul, App Mesh), Linkerd-proxy                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  API GATEWAY (Specialized Reverse Proxy):                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Auth, rate limit, routing, transformation, aggregation          │   │
│  │ • Kong, Apigee, AWS API GW, Azure APIM, GCP API GW, Traefik       │   │
│  │ • YARP (.NET native), Ocelot                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  LOAD BALANCER (L4 vs L7):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ L4 (TCP/UDP): Fast, low latency, no content awareness             │   │
│  │   Examples: AWS NLB, Azure LB, HAProxy (TCP mode), IPVS           │   │
│  │ L7 (HTTP/HTTPS): Content-aware, routing, SSL termination, WAF    │   │
│  │   Examples: AWS ALB, NGINX, Envoy, HAProxy (HTTP mode), Kong      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 18. FIREWALLS

### WHY() (Problem Statement)

**Network security boundary enforcement.** Firewalls = traffic filtering, inspection, control.

---

### HOW() (Firewall Types & Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FIREWALL ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FIREWALL GENERATIONS:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. PACKET FILTER (Stateless) — L3/L4 headers (IP, port, protocol) │   │
│  │ 2. STATEFUL INSPECTION — Tracks connection state (SYN, ESTABLISHED)│   │
│  │ 3. APPLICATION LAYER (L7) — Understands HTTP, DNS, SMTP, etc.     │   │
│  │ 4. NEXT-GEN (NGFW) — App-ID, User-ID, IPS, SSL inspection, sandbox│   │
│  │ 5. CLOUD-NATIVE / DISTRIBUTED — Micro-segmentation, zero trust    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPLOYMENT MODELS:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PERIMETER — Edge (Internet ↔ DMZ ↔ Internal)                     │   │
│  │ • INTERNAL / EAST-WEST — Between zones, micro-segmentation         │   │
│  │ • HOST-BASED — iptables/nftables, Windows Firewall, Cilium        │   │
│  │ • CONTAINER / K8S — NetworkPolicy, Cilium, Calico, Istio AuthZ    │   │
│  │ • CLOUD — Security Groups (AWS), NSG (Azure), Firewall (GCP)      │   │
│  │ • WAF (Web Application Firewall) — L7, OWASP CRS, rate limiting   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ZERO TRUST NETWORK (Modern):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • No implicit trust (verify every request)                        │   │
│  │ • Identity-based (not IP-based)                                   │   │
│  │ • Micro-segmentation (per workload)                               │   │
│  │ • mTLS everywhere (SPIFFE/SPIRE)                                  │   │
│  │ • Least privilege access                                          │   │
│  │ • Continuous verification                                         │   │
│  │ • Tools: Istio/Linkerd/Cilium, SPIFFE/SPIRE, OPA, Cloud Provider  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CLOUD FIREWALLS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AWS: Security Groups (stateful, instance-level), NACL (stateless,  │   │
│  │      subnet-level), WAF, Network Firewall (managed, scalable)      │   │
│  │ AZURE: NSG (subnet/NIC), Azure Firewall (managed, L3-L7),         │   │
│  │      Application Gateway WAF, Firewall Manager                     │   │
│  │ GCP: VPC Firewall Rules (global, stateful), Cloud Armor (WAF,      │   │
│  │      DDoS), Cloud NAT, Hierarchical Firewall Policies             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PATTERNS & ANTI-PATTERNS

| Area | Pattern | Anti-Pattern | Why Avoid |
|------|---------|--------------|-----------|
| **Scaling Frameworks** | LeSS (minimal) / SAFe (governance) | Custom "Spotify Model" without understanding | Cargo cult, missing guardrails |
| **Cloud** | Multi-cloud via Terraform/K8s | Single-cloud lock-in (Lambda, Cosmos) | Vendor leverage, migration impossible |
| **Linux** | Observability (eBPF, metrics, logs) | SSH debugging in production | Not reproducible, slow, risky |
| **XP Practices** | TDD, Pairing, CI, Simple Design | "We'll refactor later" | Technical debt compounds |
| **Service Mesh** | Adopt when >20 services, mTLS needed | Mesh for 5 services | Complexity > benefit |
| **Kanban** | WIP limits, flow metrics, explicit policies | No WIP limits, no metrics | Bottlenecks invisible |
| **OSI/TCP/IP** | Layer-appropriate decisions (L4 vs L7 LB) | Everything at Layer 7 | Unnecessary complexity, latency |
| **Scrum** | Sprint Goal, DoD, Retro action items | "Agile" without ceremonies | Cargo cult, no improvement |
| **CI/CD** | Fast feedback, immutable artifacts, GitOps | Manual deployments, snowflake envs | Drift, unreproducible, slow |
| **HTTP** | Proper semantics, caching, TLS 1.3 | GET with body, POST for everything | Caching broken, semantics lost |
| **Containers** | Distroless, non-root, read-only rootfs | Latest tag, root, fat images | Security, reproducibility |
| **Cloud Patterns** | Circuit Breaker, Retry, Cache-Aside | No resilience, direct calls | Cascade failures |
| **Proxies** | Right type (forward/reverse/sidecar/API GW) | Direct service-to-service | No observability, no control |
| **Firewalls** | Zero Trust, micro-segmentation, host-based | Perimeter only | Lateral movement |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. LeSS vs SAFe: "How do you choose between LeSS and SAFe?"

**Answer:**
```
LESS (Choose when):
✓ Organization already does Scrum well
✓ 2-8 teams (LeSS) or 8+ (LeSS Huge)
✓ Want minimal framework overhead
✓ Culture supports self-organization
✓ Can invest in Scrum mastery

SAFE (Choose when):
✓ Large enterprise (100+ teams)
✓ Need governance, compliance, audit trails
✓ Multiple stakeholders, complex dependencies
✓ Need predictable quarterly planning (PI)
✓ Organization wants prescribed roles/artifacts

KEY DIFFERENCE: LeSS = "Descale" (remove complexity), SAFe = "Scale" (add structure)
HYBRID: SAFe at Portfolio level, LeSS at Team level (common in practice)
```

---

### 2. Cloud: "How do you design for multi-cloud?"

**Answer:**
```
MULTI-CLOUD DESIGN PRINCIPLES:

1. ABSTRACT CLOUD DIFFERENCES:
   • IaC: Terraform/OpenTofu (not CloudFormation/Bicep)
   • Containers: Kubernetes (EKS/AKS/GKE) + Crossplane
   • Services: Use portable (Kafka, Redis, PostgreSQL) not proprietary

2. DATA GRAVITY:
   • Primary data in one cloud (cost, latency, consistency)
   • Read replicas / CDC for cross-cloud access
   • Event-driven sync (Kafka MirrorMaker, Debezium)

3. IDENTITY & NETWORKING:
   • Federated identity (OIDC, SAML) → Cloud provider IAM
   • Service mesh (Istio/Linkerd) for cross-cluster mTLS
   • Private connectivity (Interconnect, Transit Gateway, VPN)

2. FAILURE DOMAINS:
   • Active/Passive or Active/Active per region
   • DNS failover (Route 53, Cloud DNS, Traffic Manager)
   • Data replication (async for multi-region, sync for AZ)

4. OPERATIONS:
   • Unified observability (Grafana, Datadog, OpenTelemetry)
   • GitOps (ArgoCD/Flux) across clusters
   • Chaos engineering across clouds

RULE: Default to single cloud. Multi-cloud only when business requires.
```

---

### 3. Linux: "How do you debug high CPU in production?"

**Answer:"
```
DEBUG HIGH CPU (Systematic):

1. IDENTIFY PROCESS:
   top -H -p <pid>          # Thread-level CPU
   pidstat -u -t -p <pid> 1 # Per-thread
   ps -eo pid,tid,pcpu,comm --sort=-pcpu | head

2. ANALYZE WHAT IT'S DOING:
   perf top -p <pid>        # Function-level (needs perf perms)
   perf record -g -p <pid> -- sleep 30; perf report
   bpftrace -e 'profile:hz:99 /pid==1234/ { @[kstack] = count(); }'
   strace -f -p <pid> -c    # Syscall counts

3. COMMON CULPRITS:
   • GC pressure (Java/.NET/Go) → Heap sizing, allocation rate
   • Serialization (JSON) → Use Protobuf/MessagePack
   • Regex catastrophica → ReDoS, use RE2
   • Busy loops / spinlocks → Condition variables
   • Crypto (TLS) → Hardware acceleration, session reuse
   • Logging (sync, debug level) → Async, structured, sampling

4. CONTAINER/K8S:
   kubectl top pod --containers
   kubectl exec -it <pod> -- /bin/sh -c "top -H"
   Check CPU limits/requests (throttling?)
```

---

### 4. XP: "Is pair programming worth it?"

**Answer:"
```
PAIR PROGRAMMING ROI:

EVIDENCE:
• 15% more time, 15% fewer defects (Williams et al.)
• Knowledge transfer (bus factor), onboarding acceleration
• Collective ownership, code consistency

WHEN IT WORKS:
✓ Complex logic, unknown domain
✓ Onboarding (new hire + senior)
✓ Refactoring risky code
✓ Architecture decisions

WHEN IT DOESN'T:
✗ Simple, well-understood tasks
✗ Both devs junior (blind leading blind)
✗ Personality clash
✗ Remote with bad setup (latency, audio)

ALTERNATIVES:
• MOB PROGRAMMING (3-5 devs, one navigator, rotate)
• ENSEMBLE PROGRAMMING (structured mob)
• CODE REVIEW (async, but slower feedback)
• RUBBER DUCKING (explain to inanimate object)

RULE: Default to pairing for complex work; solo for simple.
MEASURE: Defect rate, cycle time, knowledge distribution.
```

---

### 5. Service Mesh: "Istio vs Linkerd vs Cilium?"

**Answer:"
```
ISTIO (Feature-rich, complex):
✓ Full traffic management, security, observability
✓ Multi-cluster, VM integration, WASM plugins
✗ Resource heavy (1-2 CPU, 1-2GB per node)
✗ Steep learning curve, complex troubleshooting
✓ Best for: Large orgs, complex requirements, platform team

LINKERD (Lightweight, security-first):
✓ Rust proxy (linkerd2-proxy), minimal resources
✓ Automatic mTLS, zero config
✗ Limited traffic management vs Istio
✗ No VM support, smaller ecosystem
✓ Best for: Security-first, resource-constrained, simpler needs

CILIUM (eBPF, sidecar-less):
✓ Kernel-level (eBPF), no sidecar overhead
✓ Network policy, L7 visibility, load balancing
✗ Service mesh features newer (beta/graduating)
✓ Best for: K8s-native, performance, network policy focus

DECISION:
Platform team + complex needs → Istio
Security-first, simple → Linkerd
K8s-native, performance, network policy → Cilium
AWS + simple → App Mesh
```

---

### 6. Kanban: "How do you set WIP limits?"

**Answer:"
```
SETTING WIP LIMITS:

STARTING POINT:
• Team size × 1.5 to 2 (e.g., 5 devs → WIP 8-10)
• Or: 1 item per developer (focus)

CALIBRATION:
1. Measure current cycle time & throughput
2. Set initial limit (conservative)
3. Monitor for 2-4 weeks:
   - Cycle time decreasing? → Good
   - Throughput stable/increasing? → Good
   - Blocked items piling up? → Too high
   - Idle time high? → Too low
4. Adjust by ±1, re-measure

PER-COLUMN LIMITS (Advanced):
• Backlog: ∞ (or large)
• Ready: 2× team size
• In Progress: Team size × 1.5
• Code Review: 2× team size
• Testing: Team size
• Done: ∞

SIGNALS TO ADJUST:
↑ WIP if: Cycle time low, throughput stable, people waiting
↓ WIP if: Cycle time high, blocked items, context switching

RULE: WIP limit is a constraint, not a target. "Stop starting, start finishing."
```

---

### 7. OSI/TCP/IP: "Explain TCP connection establishment and termination."

**Answer:"
```
TCP 3-WAY HANDSHAKE (Establishment):
Client                          Server
  │                                 │
  │──── SYN (seq=x) ──────────────►│
  │◄─── SYN-ACK (seq=y, ack=x+1) ──│
  │──── ACK (ack=y+1) ────────────►│
  │                                 │
  ESTABLISHED                    ESTABLISHED

TCP 4-WAY TERMINATION (Graceful Close):
Client                          Server
  │                                 │
  │──── FIN (seq=u) ─────────────►│
  │◄─── ACK (ack=u+1) ────────────│  (Server can still send)
  │                                 │
  │◄─── FIN (seq=v) ──────────────│
  │──── ACK (ack=v+1) ───────────►│
  │                                 │
  TIME_WAIT (2*MSL)              CLOSED
  (2 min default)                  │
                                     CLOSED

KEY STATES:
• LISTEN, SYN_SENT, SYN_RECEIVED, ESTABLISHED
• FIN_WAIT_1, FIN_WAIT_2, CLOSE_WAIT, CLOSING
• LAST_ACK, TIME_WAIT, CLOSED

TIME_WAIT PURPOSE: Handle delayed packets, prevent old connection interference
TUNING: net.ipv4.tcp_tw_reuse=1, net.ipv4.tcp_fin_timeout=15

ABORTIVE CLOSE (RST): Immediate, no TIME_WAIT, data loss possible
```

---

### 8. Scrum: "What's a good Definition of Done?"

**Answer:**
```
DEFINITION OF DONE (DoD) — Example for Microservices:

CODE QUALITY:
☐ Code compiles, passes static analysis (lint, type check)
☐ Unit tests pass (>80% coverage on new code)
☐ Code reviewed (2 approvals, no unresolved comments)
☐ No TODO/FIXME without linked issue

TESTING:
☐ Integration tests pass (Testcontainers)
☐ Contract tests pass (consumer + provider)
☐ E2E smoke tests pass in staging

SECURITY:
☐ No secrets in code (scanned)
☐ Dependency scan clean (no CRITICAL/HIGH CVEs)
☐ Container scan clean (base image, runtime)

DEPLOYABILITY:
☐ Docker image built, tagged (semver + git SHA)
☐ Helm chart / K8s manifests updated
☐ Migration scripts included (if DB changes)
☐ Feature flag added (if gradual rollout)

OBSERVABILITY:
☐ Structured logging (JSON, correlation IDs)
☐ Metrics exposed (RED: rate, errors, duration)
☐ Distributed tracing (OpenTelemetry)
☐ Health endpoints (liveness, readiness, startup)

DOCUMENTATION:
☐ API spec updated (OpenAPI/Protobuf)
☐ README updated (run, test, config)
☐ ADR written (if architectural decision)
☐ Runbook updated (if new failure modes)

ARCHITECTURE:
☐ ADR written for structural decisions
☐ No new technical debt without tracked issue
☐ Performance budget met (latency, memory)
```

---

### 9. CI/CD: "How do you implement zero-downtime deployments?"

**Answer:**
```
ZERO-DOWNTIME DEPLOYMENT STRATEGIES:

1. ROLLING UPDATE (K8s Default):
• maxSurge: 25%, maxUnavailable: 25%
• New pods ready → Old pods terminate
• Requires: Readiness probes, graceful shutdown (SIGTERM handling)

2. BLUE/GREEN:
• Two identical environments (Blue=current, Green=new)
• Deploy to Green → Smoke test → Switch traffic (DNS/LB)
• Instant rollback (switch back)
• Cost: 2x infrastructure

3. CANARY (Progressive Delivery):
• Deploy new version to 5% → 25% → 50% → 100%
• Automated promotion based on metrics (error rate, latency, business KPIs)
• Tools: Flagger, Argo Rollouts, Istio, Flux
• Automatic rollback on SLO breach

4. FEATURE FLAGS:
• Deploy code dark (flag OFF)
• Enable for internal → Beta → 10% → 100%
• Decouple deploy from release
• Tools: LaunchDarkly, Unleash, OpenFeature, homegrown

REQUIREMENTS FOR ALL:
✅ Readiness probes (not just liveness)
✅ Graceful shutdown (handle SIGTERM, drain connections)
✅ Backward-compatible API changes (additive only)
✅ Database migrations: Expand → Migrate → Contract (3-phase)
✅ Idempotent consumers (safe replay)
✅ Automated rollback on SLO breach (error rate, latency)
✅ Observability: Deployment annotations in metrics/traces
```

---

### 10. HTTP: "HTTP/2 vs HTTP/3 — what's the difference?"

**Answer:"
```
HTTP/2 (2015) vs HTTP/3 (2022):

TRANSPORT:
• HTTP/2: TCP (single connection, multiplexed streams)
• HTTP/3: QUIC (UDP-based, multiplexed streams)

HEAD-OF-LINE BLOCKING:
• HTTP/2: Yes (TCP packet loss blocks ALL streams)
• HTTP/3: No (QUIC streams independent, packet loss affects one stream)

CONNECTION ESTABLISHMENT:
• HTTP/2: TCP 3-way + TLS 1.2/1.3 handshake (2-3 RTT)
• HTTP/3: QUIC 0-RTT (resume) or 1-RTT (new) — combined crypto+transport

CONNECTION MIGRATION:
• HTTP/2: No (TCP tied to 4-tuple: src/dst IP:port)
• HTTP/3: Yes (Connection ID survives IP/port change — mobile!)

ENCRYPTION:
• HTTP/2: TLS 1.2/1.3 over TCP (optional but standard)
• HTTP/3: Mandatory TLS 1.3 built into QUIC

HEADER COMPRESSION:
• HTTP/2: HPACK (Huffman + static/dynamic table)
• HTTP/3: QPACK (similar, but handles out-of-order)

ADOPTION:
• HTTP/2: ~50% of web (universal server support)
• HTTP/3: ~25% growing (Cloudflare, Google, Facebook, CDNs)

WHEN TO USE HTTP/3:
✓ Mobile clients (network changes)
✓ High latency / lossy networks
✓ Real-time (WebRTC, gaming, streaming)
✓ Already using CDN/Load Balancer with HTTP/3 support

FALLBACK: ALPN negotiates HTTP/3 → HTTP/2 → HTTP/1.1
```

---

### 11. Containers: "How do you secure containers in production?"

**Answer:**
```
CONTAINER SECURITY CHECKLIST:

IMAGE HARDENING:
✅ Distroless / Scratch / Alpine base (minimal attack surface)
✅ Non-root USER (UID > 0)
✅ Read-only root filesystem
✅ Drop ALL capabilities, add only needed
✅ No shell (rm /bin/sh) if not needed
✅ Multi-stage builds (build → runtime separation)
✅ Sign images (cosign, Notary), verify on deploy

RUNTIME:
✅ SecurityContext: runAsNonRoot, readOnlyRootFs, capabilities
✅ PodSecurity Standards (Restricted) / Kyverno / OPA Gatekeeper
✅ Seccomp profile (default or custom)
✅ AppArmor/SELinux profile
✅ NetworkPolicy (default deny, allow explicit)
✅ Resource limits (CPU, memory, ephemeral-storage)

SUPPLY CHAIN:
✅ SBOM (Syft → CycloneDX/SPDX) on every build
✅ SCA scanning (Trivy, Grype, Snyk) in CI + Registry
✅ Image signing (cosign) + verification (sigstore)
✅ Base image pinning (digest, not tag)
✅ Automated rebuild on base image updates

K8S HARDENING:
✅ PodSecurity Admission (Restricted)
✅ RuntimeClass (gVisor/Kata for untrusted workloads)
✅ Admission controllers (NodeRestriction, PodSecurity)
✅ Audit logging, Falco/Cilium Tetragon for runtime threats
✅ CIS Benchmarks (kube-bench)

SECRETS:
✅ Never in image/env vars
✅ External Secrets Operator / CSI Driver (Vault, AWS Secrets Manager)
✅ Rotation automation
```

---

### 12. Cloud Patterns: "How do you implement the Circuit Breaker pattern?"

**Answer:"
```
CIRCUIT BREAKER IMPLEMENTATION (Polly .NET / Resilience4j Java / go-breaker):

STATES:
CLOSED (Normal) → Failure threshold exceeded → OPEN (Fail fast)
OPEN → Timeout elapsed → HALF_OPEN (Test)
HALF_OPEN → Success threshold met → CLOSED
HALF_OPEN → Failure → OPEN

CONFIGURATION (Typical):
• Failure threshold: 5 failures in 10 seconds
• Timeout (Open → Half-Open): 30 seconds
• Success threshold (Half-Open → Closed): 3 successes
• Exceptions handled: HttpRequestException, 5xx, Timeout

CODE (Polly):
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, ts) => logger.LogWarning("Circuit OPEN"),
        onReset: () => logger.LogInformation("Circuit CLOSED"),
        onHalfOpen: () => logger.LogInformation("Circuit HALF_OPEN"));

COMPOSITION (Order matters!):
Policy.WrapAsync(retry, circuitBreaker, bulkhead, timeout)

MONITORING:
• Circuit state changes (metrics: open/closed/half-open)
• Failure rate, latency, throughput
• Alert on circuit open (> 1 min)
```

---

### 13. Proxies: "When do you use forward vs reverse proxy?"

**Answer:"
```
FORWARD PROXY (Client → Internet):
✅ Client explicitly configures proxy
✅ Use cases:
   • Egress control (allowlist destinations)
   • Caching (reduce bandwidth, latency)
   • Anonymity (hide client IP)
   • Content filtering (malware, compliance)
   • SSL inspection (decrypt, inspect, re-encrypt)
✅ Examples: Squid, NGINX (forward), AWS NAT Gateway, Cloudflare Gateway

REVERSE PROXY (Internet → Server):
✅ Server-side, client unaware
✅ Use cases:
   • Load balancing (L4/L7)
   • SSL termination (cert management)
   • Caching (static assets, API responses)
   • WAF (OWASP CRS, rate limiting, bot protection)
   • Compression (gzip, brotli)
   • Request/response transformation
   • A/B testing, canary routing
✅ Examples: NGINX, HAProxy, Envoy, AWS ALB, Cloudflare, Kong, Traefik, YARP

SIDECAR PROXY (Service Mesh):
✅ Per-pod, intercepts ALL traffic
✅ mTLS, observability, resilience, routing
✅ Envoy (Istio, Consul, App Mesh), Linkerd-proxy

KEY DIFFERENCE:
• Forward: Client knows, Server doesn't
• Reverse: Server knows, Client doesn't
• Sidecar: Neither knows (transparent)
```

---

### 14. Firewalls: "How do you implement Zero Trust networking?"

**Answer:"
```
ZERO TRUST NETWORK ARCHITECTURE:

PRINCIPLES:
1. NEVER TRUST, ALWAYS VERIFY (no implicit trust)
2. LEAST PRIVILEGE (minimal access required)
3. ASSUME BREACH (design for compromise)
4. CONTINUOUS VERIFICATION (not one-time)

IMPLEMENTATION LAYERS:

1. IDENTITY (Who):
   • SPIFFE/SPIRE for workload identity (X.509 certs, auto-rotation)
   • OIDC/OAuth2 for human identity (Entra ID, Okta, Keycloak)
   • Service accounts per service (no shared)

2. NETWORK (Where):
   • Micro-segmentation (per workload, not per subnet)
   • Cilium/Calico NetworkPolicy (K8s) / Security Groups (Cloud)
   • mTLS everywhere (Service Mesh: Istio/Linkerd/Cilium)
   • No flat networks, no implicit trust zones

3. AUTHORIZATION (What):
   • OPA/Gatekeeper for admission + runtime authZ
   • RBAC/ABAC policies (attribute-based)
   • Just-in-time access (Teleport, HashiCorp Boundary)

4. DATA (What):
   • Encryption in transit (mTLS, TLS 1.3)
   • Encryption at rest (KMS, envelope encryption)
   • Field-level encryption (PII, PCI)

5. OBSERVABILITY (Verify):
   • Audit logs (immutable, tamper-evident)
   • Anomaly detection (ML on auth patterns)
   • SIEM integration (Splunk, Sentinel, Elastic)

TOOLS:
• Service Mesh: Istio/Linkerd/Cilium (mTLS, authZ)
• Identity: SPIFFE/SPIRE, Keycloak, Entra ID
• Policy: OPA/Gatekeeper, Kyverno
• Secrets: Vault, AWS Secrets Manager, Azure Key Vault
• Runtime: Falco, Tetragon, Tracee

MIGRATION PATH:
1. Inventory & map (current trust boundaries)
2. Deploy Service Mesh (permissive mode)
3. Enable mTLS (permissive → strict)
4. Define policies (start with deny-all)
5. Enable micro-segmentation
6. Continuous verification & audit
```

---

### 15. Architecture: "How do you handle architecture decisions in a team?"

**Answer:"
```
ARCHITECTURE DECISION RECORDS (ADR) PROCESS:

1. TRIGGER: Any structural decision (language, framework, DB, pattern, cloud)

2. TEMPLATE (Markdown in /docs/adr/ADR-XXXX.md):
# ADR-XXXX: Title
## Status: Proposed | Accepted | Rejected | Superseded
## Context: Forces, constraints, requirements
## Decision: What we're doing
## Consequences: + / - / Risks / Mitigations
## Alternatives: What else considered, why rejected
## Related: ADR-XXXX, RFC-XXXX

3. PROCESS:
• Author writes draft (1-2 hours)
• 3-day async review (GitHub PR comments)
• Architecture Board sync (30 min, controversial only)
• Decider (Chief Architect) decides
• Merge → Communicate in #arch-decisions

4. TOOLING:
• adr-tools / adr-log for numbering
• GitHub Action: Validate template, check links
• Auto-generated index page

5. GOVERNANCE:
• Quarterly review: Superseded? Missing?
• Metrics: % decisions with ADR, time-to-decide, supersession rate
• Exception process: Time-boxed, documented in ADR

CULTURE:
• "Architecture is everyone's responsibility"
• ADRs written BY teams, not FOR teams
• Decisions documented, not tribal knowledge
• Dissenting views recorded (not suppressed)
```

---

## QUICK REFERENCE: NETWORKS & OPERATIONS CHEAT SHEET

### Scaling Frameworks
| Teams | Framework | Key Characteristic |
|-------|-----------|-------------------|
| 1 | Scrum | Sprint, PO, SM, DoD |
| 2-8 | LeSS | One PO, One Backlog, Feature Teams |
| 8+ | LeSS Huge | Area POs, Requirement Areas |
| 50+ | SAFe | PI Planning, ARTs, RTE |
| Any | Scrum@Scale | Modular, Scrum-based |

### Cloud Provider Strengths
| Need | AWS | Azure | GCP |
|------|-----|-------|-----|
| Kubernetes | EKS (mature) | AKS (integrated) | GKE (innovative) |
| Serverless | Lambda (mature) | Functions (integrated) | Cloud Run (knative) |
| Data Warehouse | Redshift | Synapse | BigQuery |
| AI/ML | SageMaker | ML Studio | Vertex AI |
| Hybrid | Outposts | Arc | Anthos |
| Enterprise | Largest share | Microsoft shops | Data/Analytics |

### Linux Debugging
| Symptom | Tool |
|---------|------|
| High CPU | perf, bpftrace, pidstat |
| High Memory | pmap, /proc/pid/smaps, valgrind |
| Disk I/O | iostat, iotop, fio |
| Network | tcpdump, ss, bpftrace |
| Syscalls | strace, bpftrace |
| Kernel | dmesg, journalctl, bpftrace |

### Container Security
| Layer | Control |
|-------|---------|
| Image | Distroless, non-root, signed, scanned |
| Runtime | Seccomp, AppArmor, capabilities, read-only fs |
| Network | NetworkPolicy, mTLS, Service Mesh |
| Supply Chain | SBOM, SLSA, cosign, signed images |
| K8s | PodSecurity Restricted, OPA, Kyverno, CIS |

### HTTP Evolution
| Version | Transport | Multiplexing | Encryption | HOL Blocking |
|---------|-----------|--------------|------------|--------------|
| HTTP/1.1 | TCP | No (pipelining broken) | Optional | Yes |
| HTTP/2 | TCP | Yes (streams) | Optional (TLS) | Yes (TCP level) |
| HTTP/3 | QUIC (UDP) | Yes (streams) | Mandatory (TLS 1.3) | No (stream level) |

### Resilience Patterns
| Pattern | Purpose | Key Config |
|---------|---------|------------|
| Circuit Breaker | Fail fast | 5 failures/10s → open 30s |
| Retry | Transient failures | Exp backoff + jitter, max 3 |
| Timeout | Bound wait | Per dependency |
| Bulkhead | Isolate resources | Max 10 concurrent |
| Idempotency | Safe retries | Client key + server dedupe |

### Zero Trust
| Principle | Implementation |
|-----------|----------------|
| Verify Identity | SPIFFE/SPIRE (workloads), OIDC (humans) |
| Least Privilege | OPA/Gatekeeper, RBAC/ABAC |
| Assume Breach | Micro-segmentation, mTLS everywhere |
| Continuous Verification | Audit logs, anomaly detection, SIEM |

---

*This document covers all topics from the Networks / Operations Knowledge section of the roadmap.sh Software Architect roadmap with exam-style depth.*