# METR-0001: Meteridian Platform Architecture

- **Status:** provisional
- **Authors:** @pgarciaq
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18

## Summary

Meteridian is a billing-grade metering, rating, and cost management platform for
hybrid infrastructure. It covers the full spectrum: Kubernetes, bare metal,
hypervisors, cloud providers (AWS, Azure, GCP), network devices, mainframes,
OpenStack, and AI/ML workloads. It provides telco-grade rating accuracy with
real-time balance management, deploys on-premises or in the cloud, and is fully
open source under Apache 2.0.

The name "Meteridian" is a portmanteau of "metering" and "meridian" -- a
measurement line and standard.

## Motivation

### Problems with the Current State (Koku)

1. **Limited infrastructure coverage** -- Only Kubernetes (OCP), AWS, Azure, GCP.
   No bare metal, hypervisors, network devices, mainframes, or OpenStack.
2. **Post-hoc accounting only** -- Costs are calculated after the fact. No
   real-time balance management, prepaid credits, or consumption enforcement.
3. **Cloud-first architecture** -- Depends on Trino/Hive for heavy aggregation.
   On-prem is a secondary path.
4. **No billing-grade accuracy** -- Eventual consistency; no exactly-once
   processing guarantees.
5. **No tokenomics** -- No credit abstraction, prepaid wallets, pooling,
   rollover, or budget enforcement.
6. **No optimization integration** -- Cost data is passive; no connection to
   rightsizing or AIOps engines.

### Goals

- Meter ANY infrastructure (not just cloud-native)
- Billing-grade accuracy (exactly-once, sub-minute latency)
- Real-time balance management with consumption enforcement
- On-premises first, cloud-ready
- Telco-grade rating (tiered, time-of-day, bundles, commitments, prepaid)
- Full tokenomics engine (credits, wallets, pooling, DePIN settlement)
- Optimization integration (robne, Kubernaut)
- Open source (Apache 2.0), no vendor lock-in

### Non-Goals

- OCI support (deprecated/removed)
- Replacing invoice generation (delegate to Lago/Stripe)
- Building a full ERP system
- Blockchain-native architecture (DePIN is an optional integration, not core)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA COLLECTION                                   │
│                                                                           │
│  meteridian-collector (Redpanda Connect + custom plugins)                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│  │Prometheus│ │K8s API   │ │SNMP/     │ │vSphere/  │ │Cloud APIs│     │
│  │OTel/PCP  │ │Informers │ │Redfish   │ │libvirt   │ │CUR/BQ    │     │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘     │
│       └──────────────┴───────────┴─────────────┴────────────┘           │
│                              │ CloudEvents                               │
└──────────────────────────────┼───────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         HOT PATH (~1.3s p99)                             │
│                                                                           │
│  Ingest ──► Validate ──► Deduplicate ──► Enrich ──► Rate ──► Store      │
│                                            │                              │
│                                            ├──► Balance Update (Valkey)   │
│                                            │                              │
│                                            └──► Enforce (budget caps)     │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ async fan-out
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         WARM PATH (async)                                 │
│                                                                           │
│  Anomaly Detection (Augurs) ──► Alert                                    │
│  Optimization (robne/Kubernaut) ──► Recommendation / Enforcement         │
│  Forecasting (Augurs ETS/MSTL) ──► Budget Projection                    │
│  Virtual Tags (ZEN Engine) ──► Enrichment Feedback                       │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         STORAGE                                           │
│                                                                           │
│  PostgreSQL + TimescaleDB          Valkey                                 │
│  ┌──────────────────────┐         ┌───────────────┐                     │
│  │ Immutable Event Store │         │ Real-Time     │                     │
│  │ (hypertables)         │         │ Balances      │                     │
│  │ Catalog + Contracts   │         │ Idempotency   │                     │
│  │ Tenant Schemas        │         │ Rate Cache    │                     │
│  └──────────────────────┘         └───────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         API + PORTALS                                     │
│                                                                           │
│  REST API ──► Service Provider Portal                                    │
│           ──► Tenant Admin Portal                                        │
│           ──► Customer Self-Service Portal                               │
│           ──► External Billing (Lago, Stripe, SAP)                       │
│           ──► FOCUS v1.1 Export                                          │
│           ──► TM Forum APIs (TMF635, TMF678)                            │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1. Data Collection Architecture

### Framework: Redpanda Connect (Apache 2.0)

Use Redpanda Connect (formerly Benthos) as the collector framework -- the same
approach OpenMeter uses. This gives 200+ data source connectors for free. Custom
plugins are built ONLY for infrastructure sources that Redpanda Connect doesn't
support natively.

### Built-in Sources (via Redpanda Connect)

| Category | Sources |
|----------|---------|
| **Cloud** | AWS (SQS, SNS, S3, Kinesis), GCP (Pub/Sub, Cloud Storage, BigQuery), Azure (Event Hubs, Blob, Queue) |
| **Messaging** | Kafka, AMQP, MQTT, NATS, Redis Streams, RabbitMQ |
| **Databases** | PostgreSQL, MySQL, MongoDB, ClickHouse |
| **Observability** | Prometheus, OpenTelemetry, StatsD |
| **Kubernetes** | Pod lifecycle events, resource usage |
| **HTTP** | Webhooks, REST APIs, WebSockets |
| **Files** | S3, SFTP, local filesystem |
| **AI/LLM** | LangChain, NVIDIA Run:ai |

### Custom Plugins (~10)

| Plugin | Source | Rationale |
|--------|--------|-----------|
| `snmp` | SNMP-enabled network devices | Infrastructure-specific |
| `redfish` | IPMI/Redfish BMC APIs (bare metal) | Hardware management protocol |
| `vsphere` | VMware vSphere/VCF API | Enterprise hypervisor |
| `libvirt` | KVM/QEMU via libvirt API | Open-source hypervisor |
| `nutanix` | Nutanix Prism Central API | Enterprise HCI |
| `hyperv` | Hyper-V via WMI/PowerShell | Windows hypervisor |
| `netflow` | NetFlow/sFlow/IPFIX collectors | Network traffic metering |
| `pcp` | Performance Co-Pilot | Red Hat performance monitoring |
| `mainframe` | CICS/IMS/zVM interfaces | Mainframe metering |
| `ocp-operator` | Evolved koku-metrics-operator | Emits CloudEvents instead of CSV tarballs |

### Deployment

The collector is deployed as:
- A Helm chart with presets (like OpenMeter)
- A standalone binary for non-Kubernetes environments
- An OCP operator (evolved from koku-metrics-operator)

All sources emit CloudEvents to the Meteridian ingestion API.

## 2. Performance Budget

### Target SLAs

- **30 seconds:** Resource consumption to metric sent (upstream of Meteridian)
- **60 seconds:** Metric received to billing-grade cost (Meteridian pipeline)

### Pipeline Latency Breakdown

| Stage | p99 Latency |
|-------|-------------|
| CloudEvents HTTP ingestion | < 100ms |
| Event bus publish + consume (NATS/Kafka) | < 500ms |
| Deduplication (Bloom filter / Valkey idempotency) | < 100ms |
| Enrichment (inventory lookup, cached in Valkey) | < 50ms |
| Rating (rate plan lookup + arithmetic) | < 200ms |
| Balance update (Valkey INCRBY atomic) | < 50ms |
| Store (async write to PostgreSQL) | < 200ms |
| **Total pipeline** | **~1.3s p99** |

At 1M events/sec: Horizontally partitioned by (tenant_id, resource_id). Each
partition handles ~10K events/sec. Latency stays constant.

### Hot Path vs. Warm Path

```
Hot Path (1.3s p99): Ingest -> Validate -> Enrich -> Rate -> Store
Warm Path (async):   Enrich -> fan-out -> Anomaly Detection -> Optimization Engine
                                         -> Alert if anomaly   -> Lifecycle event
```

Optimization/AI stages run on the warm path (async fan-out copy), NOT in the hot
path. If they produce lifecycle events (e.g., "resize VM"), those enter the hot
path as new events.

## 3. Database Architecture

### Core Principle

**Invariant across ALL deployment profiles: PostgreSQL + Valkey. Two databases.**

| Profile | Analytics Layer | Config/Billing | Real-Time Balances |
|---------|----------------|----------------|--------------------|
| **Micro** | PostgreSQL (vanilla, partitioned tables) | Same PostgreSQL instance | In-process Go cache (ristretto) |
| **Small** | PostgreSQL + TimescaleDB extension | Same PostgreSQL instance | Valkey |
| **Standard** | PostgreSQL + TimescaleDB | PostgreSQL | Valkey |
| **Enterprise** | ClickHouse (optional read accelerator) | PostgreSQL | Valkey cluster |
| **Hyperscale** | Citus (horizontal sharding by tenant_id) | PostgreSQL | Valkey cluster |

### Key Principles

- The system ALWAYS writes metering events to PostgreSQL (time-based partitioning)
- TimescaleDB continuous aggregates pre-compute rollups (hourly, daily, monthly)
- ClickHouse is introduced ONLY as an optional read-path accelerator
- For micro profile: even Valkey is optional (use in-process Go cache)

**Databases the team must support: 2** (PostgreSQL, Valkey). ClickHouse is optional.

### TimescaleDB vs. Citus

| Aspect | TimescaleDB | Citus |
|--------|-------------|-------|
| **Problem** | Time-series queries slow on large tables | Single PG node can't handle write/storage load |
| **Mechanism** | Hypertables (auto time-partitioning) + compression + continuous aggregates | Sharding across PG nodes by distribution column (tenant_id) |
| **Best for** | Metering events (timestamped, time-range queries) | Multi-tenant distribution (>50K TPS, >10TB) |
| **Scaling** | Vertical (bigger machine) | Horizontal (more machines) |

TimescaleDB is the default for all profiles up to Enterprise. Citus is the
upgrade path for hyperscale deployments only.

### PostgreSQL HA: CloudNativePG

CloudNativePG (CNCF Sandbox) is the standard for PostgreSQL on Kubernetes.

| Aspect | CloudNativePG | Patroni + pgBouncer |
|--------|--------------|---------------------|
| **Failover** | 5-10 seconds | 20-45 seconds |
| **Dependencies** | None (K8s API for leader election) | DCS (etcd or K8s endpoints) |
| **Pooler** | Built-in Pooler CRD | Separate pgBouncer |
| **Backup** | Built-in Barman Cloud (S3/GCS) | WAL-G (separate config) |
| **K8s integration** | Native (custom pod controller) | Bolted on (StatefulSet + sidecar) |

CloudNativePG works with TimescaleDB via custom Docker images (production-proven)
or ImageVolume extensions (K8s 1.33+).

### RPO/RTO by Profile

| Profile | RPO | RTO | Mechanism |
|---------|-----|-----|-----------|
| **Micro** | ~5 min | < 30 min | Single PG, pg_dump backup |
| **Small** | < 1 min | < 15 min | CloudNativePG (2 instances) |
| **Standard** | 0 | < 5 min | CloudNativePG (3 instances, synchronous) |
| **Enterprise** | 0 | < 1 min | CloudNativePG (3+, multi-AZ), Valkey cluster |

Safety net: event bus (NATS/Kafka) allows replay of unprocessed events after recovery.

## 4. Metering Dimensions

### What Koku Covers Today (~15 dimensions)

| Category | Dimensions |
|----------|------------|
| **Compute (OCP)** | CPU request/usage/limit hours, Memory request/usage/limit GB-hours |
| **Storage (OCP)** | PVC request/usage/capacity GB-months |
| **GPU (OCP)** | GPU request/usage hours |
| **Node capacity** | Node CPU cores, memory |
| **Cloud costs** | AWS CUR, Azure exports, GCP BigQuery (passthrough) |
| **VM (KubeVirt)** | VM CPU/memory hours |

### What Meteridian Adds

| Category | Dimensions |
|----------|------------|
| **Network** | Egress/ingress bytes, packets/sec, bandwidth utilization, inter-zone/cross-region traffic, NAT gateway throughput |
| **IP addressing** | IPv4 addresses (allocated, in-use, elastic), IPv6 prefixes, floating IPs |
| **Storage (advanced)** | IOPS (read/write), throughput MB/s, tiers (hot/warm/cold/archive), snapshot capacity, replication factor |
| **Software/Licensing** | vCPU-hours for licensed software, named/concurrent user seats, per-core licensing (Oracle, SQL Server) |
| **AI/ML** | Token consumption (input/output), GPU-hours by model type, inference requests, training job hours, MIG slice hours, AI agent BOM |
| **Energy/Sustainability** | kWh consumed, PUE-adjusted kWh, carbon emissions (kg CO2e), cooling overhead |
| **Time/Event** | API calls, function invocations, message queue depth, DNS queries, load balancer connections |
| **Compliance/Governance** | Data classification storage (PII GB-hours), audit log volume, encryption key operations |
| **Bare metal** | Provisioned capacity (CPU cores, RAM GB, disk TB), power draw, rack units |
| **Mainframes** | MIPS, MSU, LPAR cores, IFL hours, zVM guest hours |
| **Network devices** | Switch port utilization, router throughput, VLAN membership, firewall rule evaluations |

## 5. Virtual Tags

Virtual tags are computed labels derived from rules, evaluated at enrichment time.

### Rule Definition

Rules are defined using JSON Decision Model (JDM) via GoRules -- users build
rules visually, not by writing code.

```yaml
kind: VirtualTagRule
spec:
  tagKey: "environment_tier"
  ruleType: "jdm"
  jdmContent: "<JDM JSON blob>"
  inputs:
    - source: kubernetes_labels
    - source: cloud_tags
    - source: inventory_attributes
    - source: other_virtual_tags     # chaining supported
  evaluateAt: enrichment
```

### Capabilities

- Virtual tags can chain (topological sort, cycle detection at definition time)
- Usable everywhere real tags are: filtering, grouping, rate plan selection,
  cost allocation, anomaly scoping, dashboards, API queries
- Generalizes Koku's cost categories into a universal rule engine

### UI: GoRules JDM Editor

| Component | Purpose | License |
|-----------|---------|---------|
| **JDM Editor** (`@gorules/jdm-editor`) | Visual rule editor React component | MIT |
| **ZEN Engine** (`gorules/zen`) | Rule evaluation engine (Rust, Go bindings) | MIT |

Users build rules in a spreadsheet-style decision table (no code). An optional
AI chatbot (Ollama, self-hosted) can generate JDM templates from natural language.

## 6. Hypervisor Support

| Phase | Hypervisors | Market Share | Discovery |
|-------|-------------|-------------|-----------|
| **Day 1** | KVM/libvirt (RHEV, OCP Virt, Proxmox), VMware vSphere/VCF | ~60% | libvirt API, K8s CRDs, VCF Operations API |
| **Phase 2** | Hyper-V, Nutanix AHV, Proxmox VE, oVirt, Harvester | ~35% | WMI/PowerShell, Prism Central, Proxmox API, oVirt REST, K8s CRDs |
| **Phase 3** | Xen/XCP-ng, Oracle VM, Scale Computing | ~5% | XAPI, Oracle VM Manager, SC//Platform REST |

## 7. OpenStack Support

Code to the OpenStack API contract (Nova, Cinder, Neutron, Keystone), not to
vendor-specific APIs.

| Vendor | Support |
|--------|---------|
| RHOSO (Red Hat OpenStack Services on OpenShift) | Day 1 |
| Mirantis MOSK | Yes (standard APIs) |
| Canonical OpenStack (Sunbeam/MicroStack) | Yes (standard APIs) |
| OpenStack-Helm | Yes (standard APIs) |
| Standalone (Kolla-Ansible, etc.) | Phase 2 |

## 8. Forecasting and Anomaly Detection

### Tiered Strategy

| Tier | Method | Implementation | When |
|------|--------|---------------|------|
| **Tier 1: Statistical** | ETS, MSTL, Prophet, Z-score, MAD, DBSCAN, changepoint | Augurs (Rust) via Go FFI/WASM | Day 1 |
| **Tier 2: Rule-based** | User-defined CEL expressions | Go (cel-go library) | Day 1 |
| **Tier 3: ML (optional)** | Merlion, Kats, foundation models (Moirai/TimeFM) | Python sidecar (gRPC) | Phase 2 |

Augurs (Rust, MIT/Apache-2.0) eliminates the Python sidecar for 90% of use
cases. It provides ETS, MSTL, Prophet, outlier detection, changepoint detection,
and clustering -- all in Rust with Go bindings.

## 9. Rate Plan Approval Workflow

### Engine: Fluxo (MIT, Embeddable Go Library)

| Option | Complexity | Dependencies | Use Case |
|--------|-----------|--------------|----------|
| **Fluxo** (chosen) | Low | PostgreSQL (already have it) | 1-10 workflows, durability, no external services |
| Custom FSM | Minimal | None | Only 1 trivial workflow |
| Hatchet | Medium | Separate service + PG | Many workflows, monitoring UI |
| Temporal | High | Temporal cluster + DB | Complex cross-service workflows |

Fluxo is an embeddable Go library with PostgreSQL persistence. Deterministic,
retryable, durable. <1ms overhead per step. MIT license.

```go
wf := fluxo.NewWorkflow("rate_plan_approval").
    Step("validate", validateRatePlan).
    Step("check_authz", checkOpenFGAPermission).
    Step("await_approval", awaitApprovalSignal).
    Step("activate", activateRatePlan).
    Step("notify", emitCloudEvent)
```

Upgrade path: Hatchet (MIT, Go-native) when workflow complexity grows.

## 10. Consumption Control Enforcement

| Environment | Mechanism | How |
|-------------|-----------|-----|
| **OpenShift AI** | Limitador (Kuadrant) | Token-based rate limiting via Envoy |
| **Kubernetes** | ResourceQuota + admission webhook | Meteridian calls K8s API to patch quotas |
| **API gateway** | Envoy ext_authz | Meteridian as ext_authz backend, check balance |
| **VM workloads** | Hypervisor API lifecycle events | Suspend/resize via libvirt/VMware/Nutanix |
| **Kubernaut** | Recommendation + enforcement loop | Budget constraints from Meteridian |

## 11. Telco vs. Cloud Billing

| Aspect | Cloud-Grade | Telco-Grade |
|--------|-------------|-------------|
| Accuracy | Eventual consistency; reconciliation | Exactly-once; no reconciliation needed |
| Latency | Minutes to hours | Real-time (< 60s) |
| Rating complexity | Simple (per-unit, per-hour) | Tiered, time-of-day, bundles, commitments |
| Balance management | Post-paid (monthly) | Pre-paid + post-paid (real-time) |
| Regulatory | SOC2, GDPR | SOC2, GDPR + telecom regulations |
| Dispute resolution | Self-service portal | Formal mediation process |

Meteridian targets telco-grade accuracy (exactly-once processing, sub-minute
rating, real-time balance) without requiring telecom-specific infrastructure.

## 12. Product Catalog

### Hierarchy

```
Catalog
  +-- Product Family (e.g., "OpenShift Compute", "AI Services")
       +-- Product (e.g., "OCP Pod Hours", "GPU Inference")
            +-- Plan (e.g., "Standard", "Enterprise", "Commitment-1yr")
                 +-- Price Component (metric, pricing model, constraints)
                 +-- Add-On (e.g., "Premium Support", "GPU Burst Pack")
                 +-- Entitlement (e.g., "Access to namespace X")
                 +-- Feature Flag (e.g., "anomaly_detection_enabled")
```

### Design Principles

- **Versioned and immutable** -- Changes create new versions; historical invoices
  reference the version active at billing time
- **Effective dating** -- Plans have `valid_from` / `valid_to` for scheduled changes
- **Inheritance** -- Child plans inherit parent defaults; override only differences
- **Multi-currency** -- Price components store amounts in multiple currencies with
  validity periods
- **Composable** -- Plans bundle multiple products; products appear in multiple plans
- **API-first** -- Full CRUD REST API; GitOps-friendly (YAML/JSON export/import)

## 13. Portal Architecture

### 13a. Service Provider Portal

For operators of the Meteridian platform (internal IT, MSP, sovereign cloud).

| Capability | Description |
|------------|-------------|
| Catalog management | Create/edit products, plans, pricing, add-ons |
| Tenant lifecycle | Onboard, suspend, decommission tenants |
| Rate plan administration | Create, version, approve, activate |
| Global dashboards | Cross-tenant revenue, usage trends, capacity |
| System health | Pipeline latency, throughput, queue depths |
| Compliance | Audit logs, data sovereignty map, GDPR requests |
| Collector management | Deploy/configure collectors, monitor connectivity |
| Integration config | External billing connections (Lago, Stripe, SAP) |

### 13b. Tenant Admin Portal

For FinOps leads and platform team leads within a customer organization.

| Capability | Description |
|------------|-------------|
| Cost allocation | Virtual tags, cost categories, showback rules |
| Budget management | Org/team/user budgets, alerts, enforcement |
| Credit management | Balances, top-ups, rollover policies |
| User management | BYOA integration, team hierarchies |
| Rate plan selection | Browse plans, request changes, view contracts |
| Report scheduling | Periodic cost reports (email, S3, webhook) |
| Optimization settings | robne/Kubernaut thresholds, approve recommendations |

### 13c. Customer Self-Service Portal

For end users consuming resources (developers, data scientists).

| Capability | Description |
|------------|-------------|
| Usage dashboard | Real-time personal/team consumption |
| Credit balance | Current balance, burn rate, depletion forecast |
| Cost breakdown | By project, namespace, resource type, period |
| Invoice history | Download invoices, dispute charges |
| Recommendations | Rightsizing from robne/Kubernaut |
| Resource catalog | Browse services, pricing, place orders |

### Technology

- React + PatternFly 6 (consistent with koku-ui heritage)
- Federated modules (each portal tier is independently deployable)
- Embeddable widgets (credit balance, usage sparkline) via Web Components
- Mobile-responsive (PWA for budget alerts)

## 14. Pricing Simulation

Run "what-if" scenarios on historical usage data with hypothetical rate plans.

### Use Cases

1. Price change impact analysis
2. Plan migration modeling
3. Discount negotiation (revenue impact)
4. New product launch projections
5. A/B testing rate plans

### Architecture

```
Historical Events (immutable store)
    |
    +-- Shadow Rating Engine (same code, different rate plan)
    |       |
    |       +-- Shadow Ledger (temporary, isolated)
    |               |
    |               +-- Comparison Report (delta vs. actual)
```

Simulation uses the SAME rating engine as production. Results stored in temporary
shadow tables (auto-expire). Supports batch (large date ranges) or interactive.

### API

```
POST /api/v1/simulations
{
  "name": "GPU price increase impact",
  "rate_plan_id": "draft-plan-uuid",
  "date_range": {"start": "2026-01-01", "end": "2026-03-31"},
  "scope": {"tenant_ids": ["*"]},
  "compare_to": "active"
}

GET /api/v1/simulations/{id}/results
{
  "summary": {"revenue_delta": "+12.4%", "affected_tenants": 47},
  "per_tenant": [{"tenant_id": "...", "current": 1200.00, "simulated": 1348.80}]
}
```

## 15. SQL-Based Metrics and Data Extracts

### SQL-Based Billable Metrics

Define billable metrics using SQL queries against the event store, enabling
arbitrarily complex metric definitions without code changes.

```sql
-- Example: "Distinct active containers per day"
SELECT
  date_trunc('day', event_time) AS period,
  tenant_id,
  COUNT(DISTINCT container_id) AS billable_quantity
FROM metering_events
WHERE event_type = 'container.active'
  AND event_time BETWEEN :start AND :end
GROUP BY 1, 2
```

Metrics are validated at creation time (EXPLAIN plan, syntax check). Executed by
TimescaleDB continuous aggregates (real-time) or batch jobs (complex queries).
Sandboxed (read-only, timeout, row limit).

### Data Extracts

| Type | Format | Delivery | Use Case |
|------|--------|----------|----------|
| Usage report | CSV, Parquet, JSON | S3/GCS, SFTP, webhook | Cost analysis |
| FOCUS export | FOCUS v1.1 (Parquet) | S3/GCS | FinOps tooling |
| Event replay | CloudEvents (JSON) | Kafka, S3 | Customer analytics |
| Invoice data | CSV, PDF | Email, S3 | Accounting |
| Audit log | JSON | S3, syslog | Compliance |

Scheduled (cron) or on-demand (API). Optional read-only SQL endpoint via
PgBouncer for direct query access (scoped to tenant schema).

## 16. Backdating and Late-Arriving Events

### Approach: Immutable Append-Only Event Store + Query-Time Rating

1. **All events are immutable.** Once written, never modified or deleted.
2. **Corrections are new events.** `correction_of: <original_id>`, type: void/amend/rerate.
3. **Billing computed at query time.** Rating engine replays all events for a period.
4. **Continuous aggregates handle 99%.** Late arrivals trigger incremental refresh.
5. **No fixed backdating window.** Events can arrive at any time.

### Correction Flow

```
Late event arrives (event_time = 30 days ago)
    |
    +-- Write to immutable event store (TimescaleDB hypertable)
    +-- Invalidate affected continuous aggregate chunk
    +-- If within current billing period:
    |       +-- Recalculate balance (Valkey update)
    |       +-- Emit: billing.balance.adjusted
    +-- If in closed period:
            +-- Flag for credit/debit memo on next invoice
            +-- Emit: billing.correction.pending
```

Every correction creates a provenance chain (original -> correction -> ...).

## 17. Enterprise Contract Management

### Contract Structure

```
Contract
  +-- Term (dates, renewal type)
  +-- Committed Spend (minimum, true-up schedule, shortfall penalty)
  +-- Rate Schedule (custom rates, effective periods, volume tiers)
  +-- Credit Grants (prepaid blocks, promotional, priority, rollover)
  +-- Amendments (price/scope changes, approval workflow)
  +-- Hierarchical Accounts (parent/children, shared/separate pools)
  +-- SLA Terms (uptime, credit penalties, performance guarantees)
```

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Committed spend tracking | Real-time progress; alerts at 50%, 75%, 90% |
| True-up calculation | Automatic shortfall at true-up dates |
| Ramp deals | Increasing commitment over term |
| Burndown visibility | Credit depletion forecast (Monte Carlo) |
| Amendment workflow | Fluxo-powered approval for all changes |
| Revenue recognition | ASC 606-compliant scheduling |
| Renewal management | 90/60/30-day alerts; auto-generate proposals |
| Multi-entity | Single contract spanning multiple legal entities |

## 18. Tokenomics Engine

### Three Dimensions

1. **Credit-Based Billing** -- Abstract consumption into credits; prepaid wallets
2. **DePIN Settlement** -- Usage triggers token burns; providers receive minted tokens
3. **Internal Token Economy** -- Departments get token budgets; gamified chargeback

### Credit Wallet System

```
Tenant
  +-- Wallet (per credit type)
       +-- Credit Grants (ordered by priority)
       |     +-- Grant 1: "Annual Commitment" (1M, expires 2027-01-01, priority: 1)
       |     +-- Grant 2: "Promotional" (10K, expires 2026-09-01, priority: 0)
       |     +-- Grant 3: "Top-Up" (50K, no expiry, priority: 2)
       +-- Balance (real-time: Valkey DECRBY)
       +-- Policies
             +-- Rollover: 25% max, 90-day window
             +-- Overage: PayGo at 1.5x standard rate
             +-- Auto-top-up: trigger at 10% remaining
```

Lowest priority number burns first (promotional before paid).

### Conversion Rates

```yaml
credit_types:
  - name: "compute_credits"
    conversions:
      - metric: "cpu_core_hours"
        rate: 10          # 1 CPU-hour = 10 credits
      - metric: "gpu_hours_a100"
        rate: 500         # 1 A100-hour = 500 credits
      - metric: "memory_gb_hours"
        rate: 2           # 1 GB-hour = 2 credits
    version: 3
    effective_from: "2026-07-01"
```

Conversion rates are versioned with effective dates. Operators choose whether
infrastructure cost savings pass to customers or increase margin.

### Pooling and Hierarchical Budgets

```
Organization Pool (1M credits/month)
  +-- Team A Budget (400K) -- enforcement: hard block
  |     +-- User A1 (100K) -- enforcement: soft alert
  |     +-- User A2 (100K)
  +-- Team B Budget (300K) -- enforcement: throttle
  +-- Reserve (300K) -- overflow for any team
```

Enforcement policies per budget node: `alert`, `throttle`, `block`, `auto_upgrade`.

### DePIN Settlement (Phase 3)

For decentralized infrastructure operators:

```
Usage Event --> Rate --> Fiat cost ($X) --> Token burn (Y = $X / price)
    --> On-chain: smart contract burns Y tokens
    --> Provider: mint Z tokens (emission schedule)
```

Chain-agnostic adapter interface (Ethereum, Cosmos, Solana, Substrate).
Smart contract templates for Burn-and-Mint Equilibrium.

### API Surface

```
POST   /api/v1/wallets                    # Create wallet
GET    /api/v1/wallets/{id}/balance       # Real-time balance (<50ms)
POST   /api/v1/wallets/{id}/grants        # Add credit grant
POST   /api/v1/wallets/{id}/topup         # Purchase credits
GET    /api/v1/wallets/{id}/transactions  # Ledger history
PUT    /api/v1/wallets/{id}/policies      # Set rollover/expiry/overage
POST   /api/v1/budgets                    # Create budget node
PUT    /api/v1/budgets/{id}/limits        # Set caps and enforcement
GET    /api/v1/budgets/{id}/utilization   # Current spend vs. budget
```

## 19. Comprehensive Market Requirements

### 19a. FinOps Standards

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| FOCUS v1.1 native export | Must-have | Built-in schema mapper; Parquet to S3/GCS |
| Unit economics | Must-have | Derived metrics: cost per transaction/user/feature |
| Budget guardrails (FinOps-as-Code) | Must-have | GitOps YAML policies; OPA evaluation |
| Anomaly detection with root cause | Must-have | Augurs + drill-down API |
| Commitment utilization tracking | Must-have | Real-time burn rate; waste alerts |
| Shared cost allocation (TBM) | Should-have | ATUM/TBM taxonomy mapping |

### 19b. Sovereign Cloud and Data Residency

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| Data residency enforcement | Must-have | OPA policy: jurisdiction tagging + routing rules |
| Gaia-X compliance labels | Should-have | Self-description API (JSON-LD) |
| Data provenance (UDLM) | Must-have | Every event carries provenance metadata |
| Sovereignty zones | Must-have | PG schema-per-jurisdiction OR row-level tagging |
| Right to erasure (GDPR Art. 17) | Must-have | Cryptographic erasure for immutable store |

### 19c. Security and Revenue Assurance

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| SOC 2 Type II (Processing Integrity) | Must-have | Idempotent processing, reconciliation, audit log |
| Fraud detection | Should-have | Anomaly detection on consumption (>10x spike) |
| Revenue leakage detection | Must-have | Reconcile metered vs. rated vs. invoiced |
| Immutable audit trail | Must-have | Append-only PG table with cryptographic chaining |
| Separation of duties | Must-have | 2-person approval (Fluxo + OpenFGA) |
| Tamper-evident billing | Should-have | Merkle tree over invoice line items |

### 19d. Telco BSS Features

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| Mediation engine | Must-have | CDR/EDR to CloudEvents; dedup; enrich; validate |
| Revenue sharing | Should-have | Multi-party settlement splits |
| Bill shock prevention | Must-have | Real-time alerts at N% of average |
| Dispute management | Must-have | Formal workflow (open/investigate/resolve) |
| Proration | Must-have | Proportional charges on mid-cycle changes |

### 19e. Enterprise Financial

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| Revenue recognition (ASC 606) | Should-have | Multi-element scheduling; deferred revenue |
| Multi-entity consolidation | Should-have | Cross-subsidiary aggregation |
| CPQ integration | Should-have | REST API for Salesforce CPQ, DealHub |
| Tax engine integration | Must-have | Avalara/TaxJar/Anrok REST |
| Credit notes and refunds | Must-have | Automated credit memo + refund workflow |

### 19f. GreenOps and Sustainability

| Requirement | Priority | Implementation |
|-------------|----------|----------------|
| Carbon metering | Should-have | kWh -> CO2e via WattTime/Electricity Maps |
| PUE-adjusted energy | Should-have | Data center PUE factor applied |
| Sustainability dashboards | Should-have | Carbon per namespace/team; trends |
| Carbon budgets | Nice-to-have | CO2 budgets alongside cost budgets |

### 19g. TM Forum Open APIs

| API | TMF ID | Purpose | Priority |
|-----|--------|---------|----------|
| Usage Management | TMF635 | Standard usage event format | Must-have |
| Customer Bill | TMF678 | Invoice format | Should-have |
| Account Management | TMF666 | Account hierarchy | Should-have |
| Product Catalog | TMF620 | Catalog structure | Should-have |

Implemented as optional API gateway translation layer (sidecar/adapter for
telco customers), not embedded in core.

## 20. Competitive Position

### Market Landscape

| Platform | Open Source | Self-Hosted | Focus |
|----------|-----------|-------------|-------|
| **OpenMeter** (Kong) | Apache 2.0 | Yes | AI/API metering |
| **Lago** | AGPLv3 | Yes | Usage-based billing |
| **Metronome** (Stripe) | No | No | Enterprise contracts |
| **Orb** | No | No | Developer-first billing |
| **Amberflo** | No | No | End-to-end metering |
| **Monetize360** | No | VPC | AI economy / NeoCloud |
| **Meteridian** | Apache 2.0 | Yes (primary) | Infrastructure + telco-grade |

### Unique Value (No Competitor Covers)

1. Infrastructure metering (bare metal, hypervisors, network, mainframes, OpenStack)
2. Telco-grade rating engine (time-of-day, bundles, commitments, prepaid)
3. Cost allocation with distribution (platform, worker, storage, network, GPU)
4. Optimization engine integration (robne, Kubernaut)
5. Virtual tags with visual rule builder
6. Tokenomics engine with DePIN settlement
7. On-premises first with sovereign cloud compliance

## 21. Reference Integrations

| Integration | Priority | Complexity |
|-------------|----------|------------|
| **Lago** (open-source billing) | 1st | Low (Docker Compose, REST API) |
| **Stripe Billing** | 2nd | Low (test mode, excellent docs) |
| **ServiceNow ITFM** | 3rd | Medium (free developer instances) |
| **SAP S/4HANA** (OData) | Community | High (requires SAP expertise) |

## 22. License Audit

All core dependencies use permissive licenses compatible with Apache 2.0.

| Component | License | Risk |
|-----------|---------|------|
| GoRules ZEN Engine | MIT | None |
| GoRules JDM Editor | MIT | None |
| Augurs (Grafana) | MIT OR Apache-2.0 | None |
| Fluxo | MIT | None |
| CloudNativePG | Apache 2.0 | None |
| Redpanda Connect OSS | Apache 2.0 | None |
| OpenFGA | Apache 2.0 | None |
| CEL-Go | Apache 2.0 | None |
| TimescaleDB | TSL + Apache 2.0 | Free to use; cannot redistribute as competing managed DB |
| Lago | AGPLv3 | REST API integration only; no code embedding |

## 23. UDLM Integration

UDLM (Universal Data Lifecycle Management) is a specification for infrastructure
lifecycle management. Meteridian adopts selected concepts:

**Adopted:**
- Resource type FQN taxonomy (`Compute.VirtualMachine`, `Storage.BlockVolume`)
- Field-level provenance model (origin, actor, timestamp, reason)
- Sovereignty zone concept (jurisdiction boundaries for data routing)

**Not applicable:**
- 4-state lifecycle (metering starts at Realized)
- Provider contract (our providers are data sources)
- Event envelope (we use CloudEvents)

If DCM (UDLM realization) is the provisioning layer, Meteridian subscribes to
lifecycle events (`entity.created`, `entity.decommissioned`) to trigger metering
start/stop.

---

## Open Questions

1. Should the product catalog support subscription management natively, or
   always delegate to Lago/Stripe?
2. What is the minimum DePIN integration that provides value without requiring
   full blockchain infrastructure?
3. Should TM Forum API compliance be a first-class feature or a community-contributed adapter?
4. How do we handle multi-region deployments with data sovereignty requirements
   while maintaining a single-pane view?

## References

- [FOCUS Specification v1.1](https://focus.finops.org/)
- [Redpanda Connect](https://docs.redpanda.com/redpanda-connect/)
- [CloudNativePG](https://cloudnative-pg.io/)
- [GoRules ZEN Engine](https://github.com/gorules/zen)
- [Augurs](https://github.com/grafana/augurs)
- [Fluxo](https://github.com/petrijr/fluxo)
- [OpenFGA](https://openfga.dev/)
- [TimescaleDB](https://www.timescale.com/)
- [TM Forum Open APIs](https://www.tmforum.org/open-apis/)
- [Gaia-X Trust Framework](https://docs.gaia-x.eu/)
