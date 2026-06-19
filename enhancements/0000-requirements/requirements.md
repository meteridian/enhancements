# METR-0000: Meteridian Requirements

- **Status:** implementable
- **Authors:** @pgarciaq
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18

## Product Vision

Meteridian is an open-source, billing-grade metering and rating platform for
hybrid infrastructure. It enables organizations to measure, price, and control
consumption across any infrastructure type -- from a single Kubernetes cluster
to a global telco sovereign cloud.

**One-liner:** "Meter anything, rate it instantly, enforce budgets in real-time."

## Goals

1. Meter ANY infrastructure at billing-grade accuracy
2. Rate consumption in real-time (< 60 seconds end-to-end)
3. Enforce budgets and quotas before overspend occurs
4. Deploy anywhere: on-premises, cloud, edge, air-gapped
5. Replace Koku with a fundamentally more capable system
6. Remain fully open source (Apache 2.0) with no proprietary dependencies

## Non-Goals

1. Full invoice generation (delegate to Lago/Stripe/SAP)
2. Building a general-purpose ERP
4. Requiring blockchain infrastructure for core functionality
5. Replacing existing monitoring systems (consume their data instead)

---

## Supported Infrastructure

| Category | Targets | Priority |
|----------|---------|----------|
| **Kubernetes** | OpenShift, vanilla K8s, EKS, AKS, GKE | Day 1 |
| **Cloud providers (Tier 1)** | AWS, Azure, GCP (cost data ingestion) | Day 1 |
| **Cloud providers (Tier 2)** | IBM Cloud, Oracle Cloud, Alibaba Cloud, Huawei Cloud | Phase 2 |
| **SaaS and AI providers** | OpenAI, Anthropic, Snowflake, Datadog, and others | Phase 2 |
| **FOCUS data** | Any provider exporting FOCUS-compliant cost and usage data | Day 1 |
| **Hypervisors** | KVM/libvirt, VMware vSphere/VCF | Day 1 |
| **Hypervisors** | Hyper-V, Nutanix, Proxmox, oVirt, Harvester | Phase 2 |
| **Hypervisors** | Xen/XCP-ng, Oracle VM, Scale Computing | Phase 3 |
| **OpenStack** | RHOSO, Mirantis, Canonical, OpenStack-Helm | Day 1 |
| **Bare metal** | IPMI/Redfish-enabled servers | Day 1 |
| **Network devices** | SNMP, NetFlow, sFlow, IPFIX | Day 1 |
| **Mainframes** | IBM Z (LPAR, IFL, zVM, CICS) | Phase 2 |
| **AI and ML** | GPU clusters, LLM inference, training jobs | Day 1 |
| **Windows** | WMI/WinRM-based metering | Phase 2 |
| **IBM POWER** | PowerVM, LPAR | Phase 3 |
| **Oracle Exadata** | Database Machine metering | Phase 3 |
| **Storage** | Ceph, NetApp, Pure Storage, Dell PowerScale | Phase 2 |

---

## Functional Requirements

### FR-100: Metering

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-101 | Ingest metering events via CloudEvents HTTP API at >= 1M events/sec | Must |
| FR-102 | Exactly-once event processing (deduplication via idempotency keys) | Must |
| FR-103 | Support standard dimensions: CPU, memory, storage, GPU, network, IP, energy | Must |
| FR-104 | Support custom/user-defined metering dimensions without code changes | Must |
| FR-105 | Collect data from Prometheus, K8s informers, OTel, SNMP, Redfish, syslog, PCP | Must |
| FR-106 | IPv6 metering (prefix allocation, traffic by address family) | Must |
| FR-107 | AI/ML metering: tokens (input/output), GPU-hours by type, MIG slices | Must |
| FR-108 | Event enrichment: attach inventory metadata, virtual tags, provenance | Must |
| FR-109 | Multi-tenant event isolation (events never cross tenant boundaries) | Must |
| FR-110 | Late-arriving events accepted without time limit (no backdating window) | Must |

### FR-200: Rating

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-201 | Per-unit, tiered, graduated, volume, and package pricing models | Must |
| FR-202 | Time-of-day / day-of-week differential pricing | Must |
| FR-203 | Bundle pricing (multiple metrics in a single charge) | Must |
| FR-204 | Commitment-based pricing (reserved capacity, committed spend) | Must |
| FR-205 | Tag-based rating (rate selection based on resource labels) | Must |
| FR-206 | Multi-currency support with exchange rate validity periods | Must |
| FR-207 | Rate plan versioning with effective dates (schedule future changes) | Must |
| FR-208 | Markup/discount application at any level (tenant, team, resource) | Must |
| FR-209 | Rating latency < 60 seconds from event ingestion to billed cost | Must |
| FR-210 | Constant-currency reporting (pin rates for period comparisons) | Should |

### FR-300: Balance Management and Tokenomics

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-301 | Credit wallets with prepaid drawdown (atomic deduction) | Must |
| FR-302 | Real-time balance queries (< 50ms response time) | Must |
| FR-303 | Credit grants with priority ordering (promotional burns first) | Must |
| FR-304 | Configurable rollover policies (percentage, time window) | Must |
| FR-305 | Credit expiry with configurable TTL (30-365 days or never) | Must |
| FR-306 | Organization-level credit pooling across teams/users | Must |
| FR-307 | Hierarchical budgets (org > team > user) with inheritance | Must |
| FR-308 | Overage handling: PayGo, auto-top-up, hard block, or throttle | Must |
| FR-309 | Abstract credit units with versioned conversion rates | Must |
| FR-310 | Capacity tokenization: support capacity token issuance and redemption via custom block | Nice |

### FR-400: Cost Allocation and Virtual Tags

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-401 | Showback: display costs by project/team/resource without billing | Must |
| FR-402 | Chargeback: bill internal departments for resource consumption | Must |
| FR-403 | Platform cost distribution (proportional by CPU, memory, or custom) | Must |
| FR-404 | Worker and unallocated cost distribution to user namespaces (when configured) | Must |
| FR-405 | Virtual tags: computed labels from rules (no manual tagging required) | Must |
| FR-406 | Virtual tag chaining (tag A depends on tag B) | Must |
| FR-407 | Visual rule builder for virtual tags (no coding required) | Must |
| FR-408 | Cost categories: infrastructure, supplementary, distributed, unattributed | Must |
| FR-409 | Split costs across multiple cost centers by configurable ratios | Should |
| FR-410 | Shared tenancy cost allocation (noisy neighbor detection) | Should |

### FR-500: Product Catalog and Contracts

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-501 | Hierarchical product catalog (family > product > plan > component) | Must |
| FR-502 | Immutable catalog versions (historical invoices reference version) | Must |
| FR-503 | Effective-dated plans (schedule price changes in advance) | Must |
| FR-504 | Multi-year enterprise contracts with committed spend | Must |
| FR-505 | True-up calculation at configurable intervals (quarterly, annual) | Must |
| FR-506 | Ramp deals (increasing commitment over contract term) | Should |
| FR-507 | Contract amendments with approval workflow | Must |
| FR-508 | Hierarchical accounts (parent contract, child accounts) | Must |
| FR-509 | SLA terms with automatic credit penalties on breach | Should |
| FR-510 | Renewal management (alerts, auto-proposals) | Should |

### FR-600: Enforcement and Control

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-601 | Budget alerts at configurable thresholds (50%, 75%, 90%, 100%) | Must |
| FR-602 | Hard budget enforcement: block new resource requests when exhausted | Must |
| FR-603 | Throttling: reduce resource allocation at budget threshold | Should |
| FR-604 | K8s enforcement via ResourceQuota/LimitRange patching | Must |
| FR-605 | API gateway enforcement via ext_authz (check balance per request) | Must |
| FR-606 | VM lifecycle enforcement (suspend/resize via hypervisor API) | Should |
| FR-607 | Bill shock prevention (alert when spend exceeds N% of average) | Must |
| FR-608 | Integration with Kubernaut for AI-driven enforcement | Should |
| FR-609 | Consumption restriction by time-of-day or day-of-week | Nice |
| FR-610 | Graceful degradation: warn, then throttle, then block (configurable) | Must |

### FR-700: Optimization and Intelligence

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-701 | Anomaly detection on consumption patterns (statistical + rule-based) | Must |
| FR-702 | Cost forecasting (next month, quarter, year) | Must |
| FR-703 | Rightsizing recommendations via robne integration | Must |
| FR-704 | AI-driven recommendations via Kubernaut integration | Should |
| FR-705 | Pricing simulation (what-if on historical data) | Should |
| FR-706 | Commitment utilization tracking with waste alerts | Must |
| FR-707 | Unit economics calculation (cost per transaction/user/feature) | Should |
| FR-708 | Carbon emissions estimation (kWh to CO2e) | Nice |
| FR-709 | Predictive budget exhaustion alerts | Must |
| FR-710 | Optimization recommendation approval workflow | Should |

### FR-800: API and Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-801 | RESTful API for all platform capabilities | Must |
| FR-802 | FOCUS v1.4 native export (Parquet to S3/GCS) | Must |
| FR-803 | Lago integration (billing delegation) | Must |
| FR-804 | Stripe Billing integration | Should |
| FR-805 | ServiceNow ITFM integration | Should |
| FR-806 | SAP S/4HANA integration (OData) | Nice |
| FR-807 | Scheduled data extracts (CSV, Parquet, JSON to S3/SFTP/webhook) | Must |
| FR-808 | SQL-based billable metric definitions | Must |
| FR-809 | TM Forum API compliance (TMF635, TMF678) as adapter | Nice |
| FR-810 | CloudEvents emission for all state changes | Must |

### FR-900: Portals and UX

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-901 | Service provider portal (catalog, tenants, system health) | Must |
| FR-902 | Tenant admin portal (budgets, cost allocation, credit management) | Must |
| FR-903 | Customer self-service portal (usage, balance, invoices, recommendations) | Must |
| FR-904 | Embeddable widgets (credit balance, usage sparkline) via Web Components | Should |
| FR-905 | Mobile-responsive design (PWA for budget alerts) | Should |
| FR-906 | White-label / OEM support (custom branding, no Meteridian visible) | Should |
| FR-907 | Self-service tenant onboarding (registration, default plan, collector setup) | Must |
| FR-908 | Real-time usage dashboards with drill-down | Must |
| FR-909 | Invoice history with dispute workflow | Must |
| FR-910 | Visual rule builder for virtual tags (GoRules JDM Editor) | Must |

### FR-1000: Security, Compliance, and Multi-Tenancy

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1001 | Multi-tenancy with full data isolation (schema-per-tenant) | Must |
| FR-1002 | Bring-your-own-authentication (BYOA): OIDC, SAML, LDAP | Must |
| FR-1003 | Fine-grained authorization (OpenFGA for ReBAC) | Must |
| FR-1004 | Attribute-based policies (OPA for ABAC) | Must |
| FR-1005 | Immutable audit trail with cryptographic chaining | Must |
| FR-1006 | Data sovereignty zones (configurable per deployment) | Must |
| FR-1007 | GDPR right to erasure (cryptographic erasure for immutable stores) | Must |
| FR-1008 | Separation of duties (2-person approval for rate changes) | Must |
| FR-1009 | Revenue leakage detection (metered vs. rated vs. invoiced reconciliation) | Must |
| FR-1010 | SOC 2 Type II readiness (processing integrity controls) | Must |

### FR-1100: Deployment and Operations

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1101 | On-premises deployment (Helm chart, no cloud dependency) | Must |
| FR-1102 | Single-binary micro profile (PostgreSQL + Go binary, no other services) | Must |
| FR-1103 | Horizontal scaling to >= 1M events/sec | Must |
| FR-1104 | PostgreSQL HA via CloudNativePG (5-10s failover) | Must |
| FR-1105 | Air-gapped deployment support (no internet required) | Must |
| FR-1106 | Backup and restore with point-in-time recovery | Must |
| FR-1107 | GitOps-friendly configuration (rate plans, policies in YAML) | Must |
| FR-1108 | Observability: metrics, logs, traces (Prometheus, OTel) | Must |
| FR-1109 | Zero-downtime upgrades (rolling deployments) | Must |
| FR-1110 | Multi-region deployment with federated view | Should |
| FR-1111 | OpenShift Operator for deployment and lifecycle management | Should |

---

## User Stories

### Metering and Rating

- **US-001:** As a platform engineer, I want to meter CPU, memory, GPU, and
  storage usage across all my OpenShift clusters so that I can allocate costs to
  project teams.

- **US-002:** As a FinOps lead, I want to define custom metering dimensions
  (e.g., "AI inference tokens") without waiting for a code release so that I can
  bill for new services immediately.

- **US-003:** As a sovereign cloud operator, I want to meter bare metal servers
  via IPMI/Redfish so that I can bill tenants for dedicated hardware.

- **US-004:** As a network engineer, I want to meter egress traffic per tenant
  using NetFlow/sFlow so that heavy users pay their fair share of bandwidth costs.

- **US-005:** As a billing administrator, I want to apply time-of-day pricing
  (cheaper at night) so that tenants are incentivized to shift batch workloads
  off-peak.

### Balance and Budget Management

- **US-006:** As a tenant admin, I want my developers to see their real-time
  credit balance so they self-regulate consumption without needing to ask me.

- **US-007:** As a FinOps lead, I want to set a hard budget cap per team that
  actually prevents new pod deployments when exhausted, not just sends an email
  nobody reads.

- **US-008:** As a CTO, I want our organization's 1M monthly credits pooled
  across teams so that seasonal teams can borrow from idle teams' allocation.

- **US-009:** As a finance controller, I want prepaid credits to expire after
  12 months so we don't carry indefinite liability on our balance sheet.

- **US-010:** As a startup CEO, I want to buy a credit top-up with one API call
  when my team runs low, auto-charged to our card on file.

### Cost Allocation and Showback

- **US-011:** As a FinOps lead, I want platform infrastructure costs (control
  plane, monitoring, networking) distributed proportionally across user
  namespaces so that no project appears artificially cheap.

- **US-012:** As a cost analyst, I want to define a virtual tag
  "business_unit" that maps namespaces to BUs using a visual rule builder, so
  that I can group costs without relabeling 500 namespaces.

- **US-013:** As a VP of Engineering, I want a monthly chargeback report per
  team showing their total cost, broken down by rate name, so I can hold teams
  accountable for their infrastructure spend.

### Enterprise Contracts

- **US-014:** As a sales director, I want to model a 3-year ramp deal ($100K,
  $200K, $300K committed spend per year) so I can present it to the customer
  with confidence in the billing system's ability to track it.

- **US-015:** As a finance controller, I want automatic true-up calculations
  at the end of each quarter so I know immediately when a customer has unmet
  commitment and what the shortfall penalty is.

- **US-016:** As an enterprise customer, I want a consolidated invoice across
  my 5 subsidiary accounts, drawing from a shared credit pool, so my finance
  team processes one PO per quarter.

### Optimization and Intelligence

- **US-017:** As a platform engineer, I want anomaly detection to alert me when
  a namespace's CPU cost spikes 10x overnight, so I can investigate before it
  drains the team's budget.

- **US-018:** As a FinOps lead, I want to simulate "what if we increase GPU
  rates by 30%?" on last quarter's data so I can see revenue impact before
  committing to the change.

- **US-019:** As a developer, I want rightsizing recommendations that say
  "reduce your request from 4 CPU to 1.5 CPU, saving 200 credits/day" so I can
  act on them with a single click.

### Deployment and Operations

- **US-020:** As an infrastructure architect, I want to deploy Meteridian on a
  single air-gapped RHEL server with just PostgreSQL and a Go binary -- no
  Kafka, no Redis, no internet -- for a small 50-node cluster.

- **US-021:** As a sovereign cloud operator, I want to guarantee that metering
  data from French tenants never leaves the EU data center, enforced by policy,
  not just documentation.

- **US-022:** As a platform SRE, I want zero-downtime upgrades with automatic
  event replay of anything missed during the rolling restart.

### Integration

- **US-023:** As a billing administrator, I want Meteridian to push rated usage
  to Lago every hour so that Lago generates invoices and handles payment
  collection while Meteridian focuses on metering and rating.

- **US-024:** As a FinOps tool integrator, I want to export our cost data in
  FOCUS v1.4 format to S3 nightly so that our FinOps platform (CloudHealth,
  Apptio, Vantage) can consume it without custom ETL.

- **US-025:** As a telco BSS team, I want Meteridian to expose TM Forum TMF635
  (Usage Management) API so it integrates with our existing OSS/BSS stack
  without custom adapters.

---

## Constraints

1. **Language:** Go (backend), React + TypeScript (frontend)
2. **License:** Apache 2.0 for all Meteridian code
3. **Databases:** PostgreSQL (mandatory), Valkey (mandatory for Standard+), ClickHouse (optional)
4. **Deployment:** Kubernetes (primary), single-binary (micro), bare-metal (supported)
5. **Minimum resource:** Single-binary profile: 2 CPU, 4GB RAM, 50GB disk
6. **No vendor lock-in:** No hard dependency on any cloud provider or proprietary service
7. **Backward compatibility:** Migration path from Koku for existing customers

## Success Metrics

| Metric | Target |
|--------|--------|
| Event ingestion throughput | >= 1M events/sec (Standard profile) |
| Rating latency (p99) | < 60 seconds end-to-end |
| Balance query latency (p99) | < 50 milliseconds |
| Supported infrastructure types | >= 15 at GA |
| Deployment profiles | 4 (Micro, Small, Standard, Enterprise) |
| Time to first metered event (new tenant) | < 15 minutes |
| Test coverage | >= 80% |
