# METR-0005: Internal Budget Units and Chargeback

- **Status:** provisional
- **Authors:** @pgarciaq, @jordigilh
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18
- **Depends on:** METR-0001 (Platform Architecture), METR-0003 (Product Catalog), METR-0004 (Credit, Prepaid, and Token Billing)
- **Related:** METR-0002 (Platform Extensibility)

> **Terminology note:** This document uses "budget units" for internal
> allocation units used in chargeback/showback. The term "tokens" is reserved
> for blockchain and DePIN settlement (v2). See the
> [README terminology table](../../README.md#terminology) for the full glossary.

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Internal Budget Unit Model](#3-internal-budget-unit-model)
4. [Chargeback vs. Showback](#4-chargeback-vs-showback)
5. [Cost Allocation Rules](#5-cost-allocation-rules)
6. [Hierarchical Account Model](#6-hierarchical-account-model)
7. [Internal Marketplace](#7-internal-marketplace)
8. [FinOps Integration](#8-finops-integration)
9. [API](#9-api)
10. [Open Questions](#10-open-questions)
11. [References](#11-references)

---

## 1. Summary

Meteridian supports internal budget unit systems where organizations allocate budgets
to departments and teams as budget units, track consumption against those
allocations, and produce chargeback and showback reports. This enables FinOps
practices within large enterprises by providing real-time cost visibility,
accountability, and budget governance — without requiring teams to interact
directly with cloud provider billing systems.

Budget units are **not** blockchain or cryptocurrency tokens. They are an
organizational unit of account — an abstraction that decouples internal cost
allocation from external cloud pricing. Organizations define their own budget
unit types, set internal exchange rates, and allocate budgets in these units. As
teams consume cloud resources, their budget unit balances are debited according to
internal rate cards that may differ from actual cloud costs.

This enhancement defines the data model for budget unit types, budget
allocations, internal rate cards, chargeback/showback reporting, hierarchical
account structures, and the optional internal marketplace. It builds on the
credit system architecture defined in METR-0004 and generalizes the cost
distribution patterns proven at scale in production cost management systems.

---

## 2. Motivation

### 2.1 The Cost Allocation Problem

Large enterprises run thousands of workloads across multiple cloud providers and
on-premise infrastructure. The cloud bill arrives as a single invoice to IT or
Finance, but the costs were incurred by dozens of business units, hundreds of
teams, and thousands of individual developers. Allocating these costs accurately
and in real time is one of the hardest problems in FinOps.

### 2.2 Current Approaches Are Fragile

Today, most organizations rely on one or more of these approaches:

- **Cloud provider cost allocation tags**: Require discipline to apply consistently.
  Tags are often missing, misspelled, or stale. Tag coverage rates of 60-70% are
  considered good; the remaining 30-40% is "untagged" cost that gets allocated
  arbitrarily.
- **Manual spreadsheets**: Finance teams manually split invoices using rules of
  thumb ("Engineering gets 60%, Marketing gets 20%, ..."). Delayed by weeks,
  inaccurate, and not actionable.
- **Cloud provider tools**: AWS Cost Explorer, Azure Cost Management, and GCP
  Billing Reports provide some allocation features, but only within their own
  cloud. Multi-cloud organizations must manually aggregate across providers.

None of these approaches provide real-time visibility, and none enable proactive
budget governance (alerting teams before they overspend, not weeks after).

### 2.3 Internal Budget Units Solve This

An internal budget unit system provides:

- **Unified unit of account**: A single metric (budget units) that spans all
  cloud providers and on-premise infrastructure.
- **Real-time budget tracking**: Teams see their remaining budget in real time,
  not on next month's invoice.
- **Accountability**: Teams own their consumption. Budget overruns are visible
  immediately, not retroactively.
- **Incentive alignment**: Internal rate cards can be designed to incentivize
  efficient behavior (e.g., higher rates for on-demand instances, lower rates
  for reserved capacity).

### 2.4 Prior Art in Cost Distribution

Sophisticated cost distribution logic — platform cost distribution, worker
unallocated distribution, storage, network, and GPU overhead allocation — has
been proven in production at scale. These patterns map directly to the internal
budget unit model:

| Proven Pattern | Meteridian Generalization |
|---|---|
| Platform distributed costs | Shared overhead allocation |
| Worker unallocated costs | Idle resource chargeback |
| Cost model rate types | Internal rate cards |
| Namespace-level cost rollup | Team-level budget tracking |

Meteridian generalizes these patterns into a configurable, multi-tenant framework
that works across providers and organizational structures.

---

## 3. Internal Budget Unit Model

### 3.1 Budget Unit Type

A budget unit type is an organizational unit of account. Each organization can define
one or more budget unit types.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Budget unit type identifier |
| `org_id` | UUID | Owning organization |
| `name` | String | Display name (e.g., "Engineering Credits", "IT Units") |
| `symbol` | String | Short symbol (e.g., "EC", "ITU", "CT") |
| `description` | String | Human-readable description |
| `fiat_exchange_rate` | Decimal | Budget units per base currency unit (e.g., 1 USD = 100 budget units) |
| `fiat_currency` | String | Base currency (USD, EUR, etc.) |
| `divisibility` | Integer | Decimal places allowed (0 = whole units only, 2 = cent-level) |
| `created_at` | Timestamp | Creation time |

**Examples:**

- "Compute Credits" (1 USD = 100 CC): Used to track compute consumption.
- "AI Units" (1 USD = 10 AIU): Higher-value units for GPU and ML workloads.
- "Platform Units" (1 USD = 1 PU): One-to-one mapping with currency for simplicity.

Most organizations will define a single budget unit type for simplicity. Multiple budget
unit types are supported for organizations that want to track different cost categories
independently (similar to typed credits in METR-0004).

### 3.2 Budget Allocation

Finance or IT leadership allocates budget units to organizational units per
budget period.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Allocation identifier |
| `budget_unit_type_id` | UUID | Budget unit type being allocated |
| `source_account_id` | UUID | Account allocating budget units (parent in hierarchy) |
| `target_account_id` | UUID | Account receiving budget units |
| `amount` | Decimal | Number of budget units allocated |
| `period_start` | Date | Budget period start |
| `period_end` | Date | Budget period end |
| `rollover_policy` | Enum | `forfeit`, `rollover`, `rollover_capped` |
| `rollover_cap_percent` | Integer | Max rollover as percent of allocation (if capped) |
| `status` | Enum | `draft`, `active`, `frozen`, `closed` |
| `approved_by` | UUID | User who approved the allocation |
| `created_at` | Timestamp | Creation time |

Allocations are created at the start of each budget period (monthly, quarterly,
or annually). They can be adjusted mid-period by authorized users, with all
adjustments recorded in an audit log.

### 3.3 Consumption Tracking

As cloud resources are consumed, budget units are debited from the team's
allocation. Consumption tracking integrates with the Meteridian usage event
pipeline:

1. A usage event enters the block runtime (METR-0002).
2. The event is rated using the product catalog (METR-0003) to determine its
   cost in the organization's base currency.
3. The internal rate card (Section 3.4) converts the base currency cost to
   budget units.
4. The team's budget unit balance is debited.
5. A consumption record is written to the chargeback ledger.

Consumption is tracked in real time via Valkey (same infrastructure as METR-0004
credit balances) and persisted to PostgreSQL for reporting.

### 3.4 Internal Rate Card

An internal rate card maps resource consumption to budget unit costs. Rate
cards are distinct from external pricing (product catalog) — they represent the
organization's internal cost policy.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Rate card identifier |
| `org_id` | UUID | Owning organization |
| `name` | String | Rate card name (e.g., "2026 H2 Standard Rates") |
| `effective_from` | Date | When this rate card takes effect |
| `effective_to` | Date | When this rate card expires (nullable) |
| `entries` | Array | List of rate card entries |

Each entry in the rate card:

| Field | Type | Description |
|-------|------|-------------|
| `resource_type` | String | Resource SKU or category |
| `budget_unit_type_id` | UUID | Budget unit type |
| `budget_units_per_unit` | Decimal | Budget units per resource unit |
| `multiplier` | Decimal | Optional adjustment factor (default: 1.0) |
| `notes` | String | Explanation of the rate |

**Why internal rates differ from cloud costs:**

- **Subsidization**: The organization may subsidize certain workloads (e.g., R&D
  gets cheaper rates to encourage experimentation).
- **Penalty pricing**: Inefficient patterns (e.g., on-demand instead of reserved)
  may carry a surcharge to incentivize better practices.
- **Overhead loading**: Rates include a loading factor for shared infrastructure
  costs (networking, security, platform team salaries).
- **Simplification**: Internal rates can be rounded or simplified compared to the
  complex, per-second cloud pricing models.
- **Budget predictability**: Fixed internal rates provide more predictable budgets
  than volatile cloud spot pricing.

---

## 4. Chargeback vs. Showback

### 4.1 Chargeback

In chargeback mode, budget unit consumption results in actual budget transfers
between internal cost centers. The consuming team's budget is debited, and the
shared services / platform team's budget is credited.

**Characteristics:**
- Teams have real financial accountability for their cloud consumption.
- Budget overruns require formal approval (budget increase, reallocation).
- Incentivizes cost optimization because overspending has tangible consequences.
- Requires mature financial processes and organizational buy-in.

### 4.2 Showback

In showback mode, consumption is tracked and reported, but no actual budget
transfer occurs. Teams see what they *would* be charged, but their budgets are
not affected.

**Characteristics:**
- Lower organizational friction — teams see costs without financial consequences.
- Used as a stepping stone toward full chargeback.
- Helps establish baseline consumption patterns before setting budgets.
- No enforcement mechanism — teams may ignore showback reports.

### 4.3 Hybrid Mode

Meteridian supports a hybrid approach where different cost categories use
different modes:

- **Compute and storage**: Full chargeback (teams control these directly).
- **Networking and security**: Showback only (teams have limited control).
- **Platform overhead**: Allocated proportionally, showback only.

The mode is configurable per organization, per cost category, and per level in
the account hierarchy. A common progression:

1. **Month 1-3**: Showback for all costs (establish baselines).
2. **Month 4-6**: Chargeback for direct costs, showback for shared costs.
3. **Month 7+**: Full chargeback with shared cost allocation.

### 4.4 Implementation

Both modes use the same underlying data model and event processing. The
difference is in the enforcement layer:

| Aspect | Chargeback | Showback |
|--------|------------|----------|
| Balance deducted | Yes | No (shadow balance tracked) |
| Budget alerts | Enforced (can block usage) | Advisory only |
| Reports | "Charged" language | "Attributed" language |
| Period-end settlement | Invoice or journal entry | Report only |
| Disputes | Formal dispute process | Informal correction |

---

## 5. Cost Allocation Rules

> **Pluggable allocation blocks (METR-0002):** Cost allocation logic is
> implemented as Transform blocks in the Meteridian pipeline. Organizations can
> use the built-in allocation methods below, or plug in custom Transform blocks
> with arbitrary allocation logic — weighted splits, ML-based attribution
> (Shapley values), multi-factor regression, time-window amortization, or any
> algorithm expressible in Go or Python. Custom allocation blocks can be shared
> via the block marketplace. See also
> [METR-0013 (Unit Economics)](../0013-unit-economics/unit-economics.md) for
> the full block-based cost allocation extensibility model.

### 5.1 Direct Allocation

The simplest allocation model: costs follow metadata tags directly to the
responsible team.

**Allocation keys:**
- Kubernetes namespace → team mapping
- Cloud provider tags (team, project, cost_center)
- Application identifiers
- User or service account ownership

Direct allocation works well for resources that are exclusively owned by a single
team. For shared resources, other allocation methods are needed.

### 5.2 Shared Cost Distribution

Shared costs (platform infrastructure, networking, security tools, monitoring) must
be distributed across consuming teams. Meteridian supports configurable distribution
strategies:

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **CPU-weighted** | Proportional to CPU consumption | Compute-heavy workloads |
| **Memory-weighted** | Proportional to memory consumption | Memory-heavy workloads |
| **Blended** | Weighted average of CPU + memory | General purpose |
| **Headcount** | Equal share per team member | Non-technical overhead |
| **Equal split** | Equal share per team | Small organizations |
| **Custom weights** | Manually defined percentages | Negotiated allocations |
| **Usage-based** | Proportional to any metered dimension | Flexible |

This maps directly to proven cost distribution patterns:

| Distribution Pattern | Meteridian Equivalent |
|---|---|
| Platform overhead (by CPU and memory) | Shared cost distribution (blended) |
| Worker unallocated (by CPU and memory) | Idle resource allocation |
| Unattributed storage | Shared storage distribution |
| Unattributed network | Shared network distribution |
| `gpu_distributed` | Specialized resource distribution |

### 5.3 Allocation Rules Engine

Allocation rules are defined as an ordered list of rules evaluated per cost item:

```json
{
  "rules": [
    {
      "name": "Direct namespace allocation",
      "match": { "has_tag": "namespace" },
      "allocate": { "method": "direct", "key": "namespace" }
    },
    {
      "name": "Platform overhead",
      "match": { "tag_equals": { "cost_center": "platform" } },
      "allocate": { "method": "cpu_weighted", "scope": "all_teams" }
    },
    {
      "name": "Untagged costs",
      "match": { "no_tags": true },
      "allocate": { "method": "equal_split", "scope": "all_teams" }
    }
  ]
}
```

Rules are evaluated top-to-bottom. The first matching rule determines the
allocation. A catch-all rule at the bottom ensures no costs are left unallocated.

### 5.4 Amortization

Some costs are one-time or periodic but should be spread over time:

- **Reserved instance purchases**: Amortized over the reservation term.
- **License costs**: Amortized over the license period.
- **One-time setup fees**: Amortized over a configurable period.

Amortization creates a smoother cost profile for teams and avoids budget spikes
from large one-time purchases.

---

## 6. Hierarchical Account Model

### 6.1 Account Hierarchy

Organizations model their structure as a tree of accounts:

```
Organization (Acme Corp)
├── Business Unit: Engineering
│   ├── Department: Backend
│   │   ├── Team: API Team
│   │   │   ├── Project: Payment Service
│   │   │   └── Project: Auth Service
│   │   └── Team: Data Team
│   │       └── Project: Analytics Pipeline
│   └── Department: Frontend
│       └── Team: Web Team
├── Business Unit: Marketing
│   └── Department: Digital
│       └── Team: Campaign Team
└── Business Unit: Operations
    └── Department: SRE
        └── Team: Platform Team
```

Each node in the hierarchy has:

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Account identifier |
| `org_id` | UUID | Organization |
| `parent_id` | UUID | Parent account (null for root) |
| `name` | String | Account name |
| `type` | Enum | `organization`, `business_unit`, `department`, `team`, `project` |
| `cost_center_code` | String | Financial system cost center code |
| `owner_id` | UUID | Account owner (user) |
| `budget_approver_id` | UUID | User who approves budget changes |

### 6.2 Budget Cascading

Budgets flow top-down through the hierarchy:

1. Finance allocates 1,000,000 budget units to the Organization.
2. The Organization allocates to Business Units: Engineering (600K), Marketing
   (200K), Operations (200K).
3. Engineering allocates to Departments: Backend (400K), Frontend (200K).
4. Backend allocates to Teams: API Team (250K), Data Team (150K).

**Constraints:**
- The sum of child allocations must not exceed the parent's allocation.
- Unallocated budget at any level is held as a "reserve" at that level.
- Reserves can be used for mid-period reallocation without going to the parent.

### 6.3 Budget Governance

| Action | Who Can Do It | Approval Required |
|--------|---------------|-------------------|
| Allocate budget to child | Account owner | No (within their allocation) |
| Increase own budget | Account owner | Parent's approver |
| Decrease own budget | Account owner | No |
| Transfer between siblings | Either owner | Parent's approver |
| Freeze an account | Parent owner or admin | No |
| Close period | Finance admin | No |

### 6.4 Reporting at Any Level

Reports can be generated at any level of the hierarchy, rolling up all child
accounts:

- **Organization level**: Total cloud spend, budget vs. actual, top cost drivers.
- **Business Unit level**: BU-specific costs, cross-department comparisons.
- **Team level**: Detailed resource consumption, per-project breakdown.
- **Project level**: Granular resource usage, cost per API call, cost per request.

Roll-up reports automatically aggregate child account data. Drill-down is supported
from any level to its children.

---

## 7. Internal Marketplace

### 7.1 Allocation Transfers

Teams that anticipate underutilizing their budget can transfer unused budget units to
teams that need more:

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Transfer identifier |
| `source_account_id` | UUID | Giving team |
| `target_account_id` | UUID | Receiving team |
| `amount` | Decimal | Budget units to transfer |
| `reason` | String | Justification |
| `status` | Enum | `pending`, `approved`, `completed`, `rejected` |
| `approved_by` | UUID | Approving user (parent in hierarchy) |

Transfers require approval from the common parent in the hierarchy (e.g., a
transfer between two teams in the same department requires department head
approval).

### 7.2 Transfer Pricing

When teams provide services to other teams (e.g., a data platform team providing
a shared analytics service), transfer pricing determines the internal cost:

- **Cost-based**: The providing team charges the consuming team at actual cost
  (no profit or loss).
- **Market-based**: The providing team charges at market rates (comparable
  external service pricing). This tests whether internal services are competitive.
- **Negotiated**: Teams negotiate a price, approved by management.

Transfer pricing records are maintained in the chargeback ledger alongside
resource consumption records.

### 7.3 Reservation System

Teams can reserve capacity (in budget units) for future use:

- **Purpose**: Guarantee budget availability for planned initiatives.
- **Mechanism**: Reserved budget units are deducted from available balance but not
  consumed. They appear as "committed" in budget reports.
- **Release**: Reservations can be released if plans change, returning budget units
  to available balance.
- **Expiry**: Reservations expire at the end of the budget period unless renewed.

---

## 8. FinOps Integration

### 8.1 Unit Economics Reporting

Meteridian calculates unit economics by dividing total cost by business metrics:

| Metric | Calculation | Use Case |
|--------|-------------|----------|
| Cost per API call | Total compute cost / API call count | API pricing decisions |
| Cost per customer | Total cost / active customer count | Customer profitability |
| Cost per transaction | Total cost / transaction count | Transaction efficiency |
| Cost per GB processed | Total cost / data volume | Data pipeline efficiency |
| Cost per ML inference | Total GPU cost / inference count | AI workload optimization |

Unit economics are calculated in real time using the consumption data from the
chargeback ledger and business metrics ingested via the usage event pipeline.

### 8.2 Budget vs. Actual Dashboards

Real-time dashboards compare allocated budgets against actual consumption:

- **Burn rate**: Current consumption rate (budget units per day or hour).
- **Projected spend**: Extrapolated end-of-period consumption.
- **Budget utilization**: Percentage of budget consumed vs. elapsed time.
- **Variance analysis**: Breakdown of over- or under-budget by resource type.

Dashboard data is served from pre-aggregated materialized views, updated in
near-real-time (< 5 minute lag).

### 8.3 Anomaly Detection

The system monitors consumption patterns and alerts on anomalies:

- **Spike detection**: Sudden increase in consumption (>2 standard deviations
  from 7-day rolling average).
- **Trend detection**: Gradual but persistent increase that will lead to budget
  overrun.
- **Pattern deviation**: Consumption at unusual times (e.g., large compute
  usage at 3 AM when the team has no scheduled jobs).
- **New resource types**: Team consuming a resource type they haven't used before
  (potential misconfiguration or misattribution).

Anomaly alerts are delivered via the standard notification channels (email,
webhook, in-app) and include:
- What anomaly was detected
- Magnitude of deviation
- Affected team and resources
- Suggested investigation steps

### 8.4 Optimization Recommendations

Based on consumption patterns, the system generates optimization suggestions:

- "Team X is 40% over budget; top cost driver is GPU compute in namespace
  `ml-training`. Consider using spot instances or scheduling jobs during
  off-peak hours."
- "Team Y has used only 15% of their storage allocation. Consider reducing
  the allocation and reallocating to Team Z, which is at 90%."
- "Department A has 5 idle reserved instances. Transfer the reservation to
  Department B, which is running on-demand instances for similar workloads."

Recommendations are generated by a periodic analysis job and prioritized by
potential savings impact.

---

## 9. API

### 9.1 Budget Unit Types

**Create a budget unit type:**

```
POST /api/v1/chargeback/budget-unit-types
```

```json
{
  "name": "Engineering Credits",
  "symbol": "EC",
  "description": "Internal unit for engineering cost allocation",
  "fiat_exchange_rate": 100.0,
  "fiat_currency": "USD",
  "divisibility": 2
}
```

**List budget unit types:**

```
GET /api/v1/chargeback/budget-unit-types?org_id={uuid}
```

### 9.2 Budget Allocations

**Create or modify an allocation:**

```
POST /api/v1/chargeback/allocations
```

```json
{
  "budget_unit_type_id": "uuid",
  "source_account_id": "uuid",
  "target_account_id": "uuid",
  "amount": 250000,
  "period_start": "2026-07-01",
  "period_end": "2026-09-30",
  "rollover_policy": "rollover_capped",
  "rollover_cap_percent": 10
}
```

**List allocations for an account:**

```
GET /api/v1/chargeback/allocations?account_id={uuid}&period=2026-Q3
```

### 9.3 Rate Cards

**Create a rate card:**

```
POST /api/v1/chargeback/rate-cards
```

```json
{
  "name": "2026 H2 Standard Rates",
  "effective_from": "2026-07-01",
  "entries": [
    { "resource_type": "cpu_core_hour", "budget_unit_type_id": "uuid", "budget_units_per_unit": 10 },
    { "resource_type": "memory_gb_hour", "budget_unit_type_id": "uuid", "budget_units_per_unit": 5 },
    { "resource_type": "gpu_a100_hour", "budget_unit_type_id": "uuid", "budget_units_per_unit": 500 },
    { "resource_type": "storage_gb_month", "budget_unit_type_id": "uuid", "budget_units_per_unit": 2 }
  ]
}
```

### 9.4 Chargeback Reports

**Get chargeback/showback report:**

```
GET /api/v1/chargeback/reports?account_id={uuid}&period=2026-06&granularity=daily
```

Response:

```json
{
  "account": { "id": "uuid", "name": "API Team" },
  "period": { "start": "2026-06-01", "end": "2026-06-30" },
  "allocation": 250000,
  "consumed": 187500,
  "remaining": 62500,
  "utilization_percent": 75.0,
  "projected_end_of_period": 225000,
  "breakdown": [
    { "resource_type": "cpu_core_hour", "quantity": 12000, "budget_units": 120000 },
    { "resource_type": "memory_gb_hour", "quantity": 8000, "budget_units": 40000 },
    { "resource_type": "gpu_a100_hour", "quantity": 50, "budget_units": 25000 },
    { "resource_type": "storage_gb_month", "quantity": 1250, "budget_units": 2500 }
  ],
  "daily": [
    { "date": "2026-06-01", "consumed": 6250, "cumulative": 6250 },
    { "date": "2026-06-02", "consumed": 5800, "cumulative": 12050 }
  ]
}
```

### 9.5 Unit Economics

**Get unit cost calculations:**

```
GET /api/v1/chargeback/unit-economics?account_id={uuid}&period=2026-06
```

Response:

```json
{
  "metrics": [
    {
      "name": "cost_per_api_call",
      "total_cost_budget_units": 120000,
      "total_units": 15000000,
      "cost_per_unit": 0.008,
      "trend": "stable",
      "change_vs_last_period": -0.02
    },
    {
      "name": "cost_per_customer",
      "total_cost_budget_units": 187500,
      "total_units": 5000,
      "cost_per_unit": 37.5,
      "trend": "increasing",
      "change_vs_last_period": 0.05
    }
  ]
}
```

### 9.6 Webhook Events

| Event | Trigger |
|-------|---------|
| `allocation.created` | New budget allocation |
| `allocation.modified` | Budget allocation adjusted |
| `allocation.exhausted` | Team consumed 100% of allocation |
| `chargeback.threshold` | Consumption crosses configured threshold |
| `chargeback.anomaly` | Anomalous consumption pattern detected |
| `transfer.requested` | Budget unit transfer requested |
| `transfer.approved` | Budget unit transfer approved |
| `transfer.completed` | Budget unit transfer executed |
| `recommendation.generated` | Optimization recommendation created |

---

## 10. Open Questions

### 10.1 Disputed Charges

How should the system handle disputed charges? A team may claim they did not use
a particular resource, or that a resource was misattributed to their namespace.

**Options:**

- **Formal dispute workflow**: Team files a dispute; a finance admin investigates
  and either upholds or reverses the charge. The charge remains on the team's
  ledger (with a "disputed" flag) until resolved.
- **Automatic reversal**: If a team disputes within a grace period (e.g., 5
  business days), the charge is automatically reversed and placed in a shared
  "disputed" pool for manual resolution.
- **Arbitration**: Disputes above a threshold amount are escalated to a designated
  arbitrator (e.g., VP of Engineering).

### 10.2 Budget Unit Tradability

Should budget units be tradeable between teams, or only allocated top-down?

- **Top-down only**: Budgets flow from parent to child. Teams cannot transfer
  budget units laterally. Simpler but less flexible.
- **Lateral transfers with approval**: Teams can transfer budget units to siblings with
  approval from their common parent. Moderate complexity.
- **Free market**: Teams can trade budget units freely. Maximum flexibility but
  introduces market dynamics that may be undesirable (hoarding, speculation,
  gaming).

Current leaning: lateral transfers with approval (Option B) as the default, with
free market as an opt-in feature for organizations that want it.

### 10.3 Shared Kubernetes Namespaces

How should costs be allocated when multiple teams share a Kubernetes namespace?

- **Sub-namespace allocation**: Use pod labels or annotations to attribute costs
  within a shared namespace. Requires consistent labeling discipline.
- **Proportional by pod count**: Divide namespace costs proportionally by the
  number of pods each team runs. Simple but ignores resource consumption
  differences.
- **Proportional by resource consumption**: Divide by actual CPU and memory usage
  per team within the namespace. Most accurate but requires fine-grained metering.
- **Ownership-based**: The namespace owner bears all costs, regardless of who
  runs pods in it. Simplest but may create perverse incentives.

### 10.4 Internal Rate Card Governance

Who should be able to set and modify internal rate cards?

- **Centralized**: Finance and FinOps team sets rates for the entire organization.
  Ensures consistency but may not reflect local realities.
- **Federated**: Each business unit can set its own rates within guardrails.
  More flexible but risks inconsistency.
- **Hybrid**: Organization-wide base rates with BU-level adjustments (multipliers).

### 10.5 Retrospective Adjustments

When cloud provider invoices arrive with unexpected charges (e.g., data transfer
fees, support charges), how should these be allocated retroactively?

- Reopen the closed period and recalculate all allocations.
- Apply the adjustment to the current period as a one-time charge.
- Create an adjustment entry that corrects the historical record without
  reopening the period.

---

## 11. References

### FinOps Standards

- [FinOps Foundation — Cost Allocation](https://www.finops.org/framework/capabilities/cost-allocation/) —
  Best practices for allocating cloud costs to business units.
- [FinOps Foundation — Chargeback and Showback](https://www.finops.org/framework/capabilities/chargeback/) —
  Guidance on implementing chargeback and showback models.
- [FinOps Foundation — Unit Economics](https://www.finops.org/framework/capabilities/measure-unit-costs/) —
  Measuring cost efficiency through unit economics.

### Industry Frameworks

- [TBM (Technology Business Management) Taxonomy](https://www.tbmcouncil.org/) —
  Standard taxonomy for IT cost categorization and allocation.
- [ITIL Financial Management](https://www.axelos.com/best-practice-solutions/itil) —
  IT service management framework including financial management practices.

### Existing Implementations

- [Red Hat Cost Management](https://www.redhat.com/en/technologies/cloud-computing/cost-management) —
  Platform and worker cost distribution, namespace-level cost allocation.
  Meteridian generalizes these patterns with pluggable allocation blocks.
- [Apptio / IBM Turbonomic](https://www.ibm.com/products/turbonomic) —
  Enterprise IT financial management and cost optimization platform.
- [VMware Aria Cost (CloudHealth)](https://www.vmware.com/products/aria-cost.html) —
  Multi-cloud cost management with chargeback and showback reporting.
- [Kubecost](https://www.kubecost.com/) —
  Kubernetes cost monitoring with namespace-level allocation and chargeback.

### Academic References

- Kaplan, R.S. and Anderson, S.R. (2007). *Time-Driven Activity-Based Costing*.
  Harvard Business School Press. — Foundation for activity-based cost allocation.
- Sharman, P. and Vikas, K. (2004). "Lessons from German Cost Accounting."
  *Strategic Finance*. — Transfer pricing and internal cost allocation models.
