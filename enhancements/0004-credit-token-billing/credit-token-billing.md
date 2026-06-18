# METR-0004: Credit, Prepaid, and Token-Based Billing

- **Status:** provisional
- **Authors:** @pgarciaq, @jordigilh
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18
- **Depends on:** METR-0002 (Extensibility), METR-0003 (Product Catalog)
- **Related:** METR-0005 (Internal Budget Units)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Credit System Architecture](#3-credit-system-architecture)
4. [Committed Spend Contracts](#4-committed-spend-contracts)
5. [Prepaid Balance Management](#5-prepaid-balance-management)
6. [Credit Conversion Rates](#6-credit-conversion-rates)
7. [Bill Shock Prevention](#7-bill-shock-prevention)
8. [API](#8-api)
9. [Integration with Block Runtime](#9-integration-with-block-runtime)
10. [Open Questions](#10-open-questions)
11. [References](#11-references)

---

## 1. Summary

Meteridian supports credit-based and prepaid billing models alongside traditional
pay-as-you-go. This enables committed spend contracts, enterprise credit grants,
prepaid balances with drawdown, and credit-based consumption patterns used by major
cloud and AI platforms (OpenAI API credits, Snowflake credits, Databricks DBUs).

Credit-based billing is a first-class primitive in Meteridian's billing engine. A
tenant's credit balance is a real-time construct backed by Valkey (ADR-0002) for
low-latency reads and an append-only ledger in PostgreSQL for auditability. Usage
events consume credits at rates defined in the product catalog (METR-0003), and the
system supports multiple concurrent credit grants with independent expiration
policies, conversion rates, and priority ordering.

This enhancement defines the data model, APIs, lifecycle rules, and integration
points required to support the full spectrum of credit-based billing — from simple
prepaid top-ups to complex multi-year committed spend contracts with ramp schedules,
true-up invoicing, and burst overage billing.

---

## 2. Motivation

### 2.1 Enterprise Cloud Contracts Are Predominantly Prepaid

The majority of enterprise cloud spend is governed by committed spend agreements —
Enterprise Discount Programs (AWS EDP), Azure MACCs, Google CUDs, and negotiated
contracts with SaaS vendors. Pay-as-you-go is the entry ramp, but committed spend
is where the revenue lives. Without native credit and commitment support, Meteridian
cannot model these contracts and is limited to on-demand billing.

### 2.2 AI Platforms Sell Credits

The explosion of AI-as-a-service has introduced credit-based billing as the dominant
model for API-driven consumption:

- **OpenAI**: Prepaid API credits, consumed per token at model-specific rates.
- **Anthropic**: Prepaid credit balance, usage billed against it.
- **Cohere**: Credit-based API access with tiered pricing.
- **Snowflake**: Compute credits consumed by virtual warehouses.
- **Databricks**: DBU (Databricks Unit) consumption with committed contracts.

These platforms need a metering and billing engine that natively understands credits
as a unit of account, not just a discount mechanism bolted onto invoice line items.

### 2.3 FinOps Maturity Requires Budget Controls

Organizations practicing FinOps need spend controls that operate in real time, not
at invoice time (30 days after the fact). Credit-based billing provides the
foundation for budget guardrails, spend alerts, and automatic throttling — all of
which require a real-time balance that decrements as resources are consumed.

### 2.4 Competitive Landscape

Modern billing platforms (Orb, Metronome, Amberflo) all support some form of
prepaid/credit billing. Stripe Billing recently added credit grants. Meteridian must
match or exceed this functionality to be competitive in the metering/billing
infrastructure market.

---

## 3. Credit System Architecture

### 3.1 Credit Grant

A credit grant is the fundamental unit of credit provisioning. It represents an
amount of credits deposited into a tenant's balance.

**Properties:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique grant identifier |
| `tenant_id` | UUID | Owning tenant |
| `amount` | Decimal | Credit amount granted |
| `currency` | String | Fiat currency code (USD, EUR) or platform unit name |
| `remaining` | Decimal | Unconsumed credits remaining in this grant |
| `grant_date` | Timestamp | When the grant was created |
| `effective_date` | Timestamp | When credits become available (may differ from grant_date) |
| `expiry_date` | Timestamp | When unused credits expire (nullable for non-expiring grants) |
| `source` | Enum | `purchase`, `promotional`, `trial`, `contract`, `adjustment` |
| `priority` | Integer | Consumption order (lower = consumed first) |
| `metadata` | JSONB | Arbitrary key-value pairs (PO number, contract ID, etc.) |
| `voided` | Boolean | Whether the grant has been administratively voided |
| `created_at` | Timestamp | Record creation time |

Credit grants are **append-only**. Once created, a grant is never modified — only
its `remaining` balance changes as credits are consumed. Voiding a grant sets the
`voided` flag and zeroes the remaining balance, recording a corresponding void
transaction in the ledger.

### 3.2 Credit Balance

The credit balance is the real-time aggregate of all active grants for a tenant. It
is maintained as a composite value:

```
available_balance = SUM(remaining) for all active, non-expired, non-voided grants
reserved_balance  = SUM of in-flight reservations (pending usage events)
effective_balance = available_balance - reserved_balance
```

**Valkey representation (ADR-0002):**

The balance is cached in Valkey for sub-millisecond reads. The Valkey data structure
uses a sorted set keyed by grant expiry, enabling efficient FIFO drawdown:

```
credit:balance:{tenant_id}          → available_balance (string, atomic DECR)
credit:reserved:{tenant_id}         → reserved_balance (string)
credit:grants:{tenant_id}           → sorted set (score = expiry_date, member = grant_id)
credit:grant:{grant_id}:remaining   → remaining balance for this grant (string)
```

Valkey operations use `MULTI/EXEC` transactions to ensure atomicity of balance
updates. The PostgreSQL ledger is the source of truth; Valkey is a read-optimized
projection that is reconciled periodically (see Section 9.3).

### 3.3 Credit Ledger

Every mutation to a tenant's credit balance is recorded as an immutable transaction
in the credit ledger (PostgreSQL):

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Transaction identifier |
| `tenant_id` | UUID | Tenant |
| `grant_id` | UUID | Credit grant affected (nullable for aggregate ops) |
| `type` | Enum | `grant`, `consumption`, `expiry`, `void`, `adjustment`, `refund` |
| `amount` | Decimal | Signed amount (positive = credit in, negative = credit out) |
| `balance_after` | Decimal | Running balance after this transaction |
| `event_id` | UUID | Associated usage event (for consumption entries) |
| `description` | String | Human-readable description |
| `created_at` | Timestamp | Transaction timestamp |

The ledger is append-only and provides a complete audit trail for every credit
operation. Ledger entries are partitioned by `tenant_id` and `created_at` for
query performance.

### 3.4 Drawdown Model

When a usage event is processed, credits are consumed according to the following
algorithm:

1. **Rate lookup**: The product catalog (METR-0003) provides the credit cost for
   the usage event based on resource type, tier, and any applicable discounts.
2. **Grant selection**: Active grants are ordered by consumption priority (default:
   FIFO by expiry date — soonest-expiring first).
3. **Drawdown**: Credits are consumed from the highest-priority grant first. If the
   grant's remaining balance is insufficient, the remainder spills to the next grant.
4. **Balance update**: Valkey balance is decremented atomically. Ledger entries are
   written for each grant touched.
5. **Exhaustion handling**: If total available credits are insufficient, behavior
   depends on the tenant's cap configuration (hard cap vs. soft cap — see Section 5).

**Consumption is immediate** — the Valkey balance is updated synchronously during
event processing. Ledger writes and reporting updates are eventually consistent,
batched in micro-intervals for throughput.

### 3.5 Credit Expiry

Credits can be configured to expire. Expiry is processed by a periodic job (Celery
beat) that runs at configurable intervals (default: hourly).

**Expiry strategies:**

| Strategy | Description |
|----------|-------------|
| FIFO | Oldest credits expire first (default) |
| LIFO | Newest credits expire first |
| Priority | Credits with highest priority number expire first |
| Custom | Arbitrary ordering via `priority` field on grants |

**Expiry workflow:**

1. Expiry job scans for grants where `expiry_date <= now()` and `remaining > 0`.
2. For each expired grant, a ledger entry of type `expiry` is created.
3. The grant's `remaining` is set to zero.
4. The Valkey balance is decremented by the expired amount.
5. A `credit.expiring` webhook is fired (or `credit.expired` after the fact).

**Grace periods:** An optional grace period (e.g., 7 days) can be configured per
grant or per tenant. During the grace period, the credits are marked as "expiring"
but remain usable. Notifications are sent at the start of the grace period.

---

## 4. Committed Spend Contracts

### 4.1 Contract Model

A committed spend contract represents a multi-period agreement where a tenant
commits to a minimum spend in exchange for discounted rates.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Contract identifier |
| `tenant_id` | UUID | Tenant |
| `name` | String | Contract name |
| `start_date` | Date | Contract start |
| `end_date` | Date | Contract end |
| `periods` | Array | List of commitment periods (see below) |
| `burst_rate_card_id` | UUID | Rate card for overage billing |
| `rollover_policy` | Enum | `none`, `next_period`, `end_of_contract` |
| `true_up_policy` | Enum | `invoice_shortfall`, `forfeit`, `rollover_as_credit` |
| `status` | Enum | `draft`, `active`, `completed`, `terminated` |

### 4.2 Commitment Periods

Each contract contains one or more commitment periods, enabling ramp deals:

```json
{
  "periods": [
    { "start": "2026-01-01", "end": "2026-12-31", "committed_amount": 100000, "currency": "USD" },
    { "start": "2027-01-01", "end": "2027-12-31", "committed_amount": 200000, "currency": "USD" },
    { "start": "2028-01-01", "end": "2028-12-31", "committed_amount": 300000, "currency": "USD" }
  ]
}
```

At the start of each period, a credit grant is automatically created for the
committed amount. The grant's `source` is `contract` and its `expiry_date` aligns
with the period end (subject to rollover policy).

### 4.3 True-Up

At the end of each commitment period, the system compares actual spend against the
committed amount:

- **`invoice_shortfall`**: If actual spend < committed amount, the shortfall is
  invoiced as a single line item. This is the most common enterprise model.
- **`forfeit`**: Unused commitment is simply lost. No invoice, no rollover.
- **`rollover_as_credit`**: Unused commitment rolls into the next period as a
  credit grant (with the next period's expiry).

True-up is processed by a scheduled job that runs at the end of each commitment
period. The job generates a true-up event that can be reviewed before invoicing.

### 4.4 Burst / Overage

When a tenant's consumption exceeds the committed amount for the period, excess
usage is billed at pay-as-you-go rates. These rates are typically 20-40% higher
than committed rates, defined in the `burst_rate_card_id`.

Burst charges are invoiced separately (or as a line item on the regular invoice)
at the end of each billing cycle. The system tracks committed vs. burst consumption
separately for reporting.

### 4.5 Rollover

Rollover policies determine what happens to unused committed credits at period end:

- **`none`**: No rollover. Unused credits are subject to the true-up policy.
- **`next_period`**: Unused credits carry forward to the next commitment period.
  The rolled-over amount is added to (not replacing) the next period's commitment.
- **`end_of_contract`**: Unused credits accumulate and are available until the
  contract ends. Credits are only forfeited/true-up'd at contract termination.

Rollover is capped by configurable limits (e.g., max rollover of 25% of the
period's commitment) to prevent unbounded accumulation.

---

## 5. Prepaid Balance Management

### 5.1 Top-Up

Tenants can add credits to their balance via manual or automatic top-ups.

**Manual top-up:** A tenant (or admin) initiates a purchase of credits. This
creates a credit grant after payment confirmation. Supported payment methods are
determined by the payment gateway integration.

**Automatic top-up (auto-reload):** Configured with:

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | Boolean | Whether auto-reload is active |
| `threshold` | Decimal | Balance level that triggers reload |
| `reload_amount` | Decimal | Amount to add when triggered |
| `max_reloads_per_period` | Integer | Safety cap on reload frequency |
| `payment_method_id` | UUID | Payment method to charge |

When the effective balance drops below `threshold`, the system automatically
initiates a top-up for `reload_amount`. A `credit.topped_up` webhook is fired
upon successful reload.

### 5.2 Low-Balance Alerts

Configurable alert thresholds relative to the initial balance or a fixed amount:

| Threshold | Default Action |
|-----------|----------------|
| 20% remaining | Notify (email + webhook) |
| 10% remaining | Notify (email + webhook + in-app banner) |
| 5% remaining | Notify (urgent) |
| 0% (exhausted) | Notify + enforce cap policy |

Alert thresholds are evaluated on every balance change. To avoid alert storms,
each threshold fires at most once per reset cycle (balance must recover above the
threshold before the alert can fire again).

### 5.3 Hard Cap vs. Soft Cap

| Mode | Behavior at Zero Balance |
|------|--------------------------|
| **Hard cap** | Usage requests are rejected with HTTP 402. Active workloads may be throttled or terminated (configurable). |
| **Soft cap** | Usage continues; overage is billed on the next invoice. An `credit.exhausted` event is emitted for visibility. |

The cap mode is configurable per tenant and can be overridden per project or
namespace. Hard cap is the default for self-service / SMB tenants; soft cap is
typical for enterprise tenants with contractual SLAs.

### 5.4 Balance Transfer

Within a hierarchical account model, credits can be transferred between
sub-accounts:

- Parent → child: budget allocation (most common)
- Child → parent: budget reclaim
- Sibling → sibling: lateral transfer (requires parent approval or admin override)

Transfers are recorded as paired ledger entries (debit on source, credit on
destination) within a single database transaction. Transfer history is queryable
via the credits API.

---

## 6. Credit Conversion Rates

### 6.1 Platform Credits and Resource Units

Platform credits are an abstraction layer between monetary value and resource
consumption. Conversion rates define how many resource units one credit purchases:

```
1 credit = 10 CPU-hours
1 credit = 50 GB-hours (memory)
1 credit = 100 GB-hours (storage)
1 credit = 0.5 GPU-hours (A100)
1 credit = 1000 API calls (standard)
```

Conversion rates are defined in the product catalog (METR-0003) as entries in the
rate card, linking credit types to resource SKUs.

### 6.2 Effective Dating

Conversion rates are versioned with effective dates. When a rate changes, existing
credits are consumed at the rate that was effective at consumption time, not at
grant time. This means the purchasing power of credits can change over time (for
better or worse).

```json
{
  "resource": "gpu_a100_hour",
  "rates": [
    { "effective_from": "2026-01-01", "credits_per_unit": 2.0 },
    { "effective_from": "2026-07-01", "credits_per_unit": 1.5 }
  ]
}
```

In this example, GPU hours become cheaper (in credit terms) on July 1st. Tenants
with existing credit balances benefit from the price reduction.

### 6.3 Universal vs. Typed Credits

**Universal credits:** A single credit pool that can be consumed against any
resource type. Conversion rates determine the exchange ratio. This is simpler
for tenants but can lead to unexpected budget exhaustion when consuming
expensive resources.

**Typed credits:** Separate credit pools per resource category (compute credits,
storage credits, AI credits). Each pool can only be consumed against its
corresponding resource type. This provides better budget isolation but is more
complex to manage.

Meteridian supports both models. The choice is configured at the tenant level.
Universal credits are the default; typed credits are available for organizations
that need fine-grained budget control.

### 6.4 Credit Valuation

For financial reporting, credits must be valued in fiat currency:

- **At-grant valuation**: Credits are valued at the price paid when the grant was
  created. Used for revenue recognition.
- **Current-rate valuation**: Credits are valued at the current conversion rates.
  Used for balance reporting and budget forecasting.
- **Blended valuation**: Weighted average of all active grant valuations. Used for
  cost allocation in chargeback scenarios.

---

## 7. Bill Shock Prevention

### 7.1 Real-Time Spend Alerts

Budget thresholds trigger notifications as spend accumulates:

| Threshold | Event | Default Channel |
|-----------|-------|-----------------|
| 50% of budget | `budget.warning` | Email |
| 75% of budget | `budget.alert` | Email + Webhook |
| 90% of budget | `budget.critical` | Email + Webhook + PagerDuty |
| 100% of budget | `budget.exceeded` | Email + Webhook + PagerDuty |

Thresholds are configurable per tenant, project, or namespace. Multiple
independent budgets can be active simultaneously (e.g., monthly budget + quarterly
budget + project-specific budget).

### 7.2 Budget Guardrails

Spend limits can be enforced at multiple levels of the tenant hierarchy:

- **Tenant-level**: Overall organizational spend cap
- **Project-level**: Per-project budget with independent thresholds
- **Namespace-level**: Per-Kubernetes-namespace budget (for OCP workloads)
- **User-level**: Individual developer spend limits

Guardrails are evaluated in the balance-check block (Section 9.1). When a
guardrail is hit, the configured action is taken.

### 7.3 Automatic Actions at Thresholds

Each threshold can be configured with one or more actions:

| Action | Description |
|--------|-------------|
| `notify` | Send notification via configured channels |
| `throttle` | Reduce resource allocation (e.g., downgrade instance tier) |
| `block` | Reject new usage requests; existing workloads continue |
| `terminate` | Gracefully shut down workloads (with configurable drain period) |

Actions are composable. A typical configuration:

```json
{
  "thresholds": [
    { "percent": 75, "actions": ["notify"] },
    { "percent": 90, "actions": ["notify", "throttle"] },
    { "percent": 100, "actions": ["notify", "block"] }
  ]
}
```

### 7.4 Forecast-Based Alerts

In addition to threshold-based alerts, the system generates predictive alerts
based on current burn rate:

- "At current burn rate, you will exhaust credits in **N days**."
- "Projected end-of-month spend is **$X**, which is **Y%** over budget."

Forecasting uses a simple linear projection by default, with optional integration
with the forecasting subsystem for more sophisticated models (exponential
smoothing, seasonal decomposition).

Forecast alerts are generated daily and on significant spend events (>5% of
remaining budget consumed in a single event).

---

## 8. API

### 8.1 Credit Grants

**Create a credit grant:**

```
POST /api/v1/credits/grants
```

```json
{
  "tenant_id": "uuid",
  "amount": 10000.00,
  "currency": "USD",
  "effective_date": "2026-07-01T00:00:00Z",
  "expiry_date": "2027-06-30T23:59:59Z",
  "source": "purchase",
  "priority": 1,
  "metadata": {
    "po_number": "PO-2026-001",
    "contract_id": "uuid"
  }
}
```

**List grants for a tenant:**

```
GET /api/v1/credits/grants?tenant_id={uuid}&status=active
```

**Void a grant:**

```
POST /api/v1/credits/grants/{grant_id}/void
```

### 8.2 Credit Balance

**Get real-time balance:**

```
GET /api/v1/credits/balance?tenant_id={uuid}
```

Response:

```json
{
  "tenant_id": "uuid",
  "available_balance": 7500.00,
  "reserved_balance": 150.00,
  "effective_balance": 7350.00,
  "currency": "USD",
  "grants_summary": {
    "active_grants": 3,
    "next_expiry": "2026-12-31T23:59:59Z",
    "expiring_amount": 2000.00
  },
  "as_of": "2026-06-18T14:30:00Z"
}
```

### 8.3 Credit Transactions

**Get transaction ledger:**

```
GET /api/v1/credits/transactions?tenant_id={uuid}&type=consumption&start_date=2026-06-01
```

Response includes paginated ledger entries with running balance.

### 8.4 Spend Simulation

**Simulate spend against a usage forecast:**

```
POST /api/v1/credits/simulate
```

```json
{
  "tenant_id": "uuid",
  "forecast": [
    { "resource": "cpu_core_hour", "quantity": 5000, "period": "2026-07" },
    { "resource": "gpu_a100_hour", "quantity": 200, "period": "2026-07" },
    { "resource": "storage_gb_month", "quantity": 10000, "period": "2026-07" }
  ]
}
```

Response:

```json
{
  "current_balance": 7350.00,
  "projected_consumption": 4200.00,
  "projected_remaining": 3150.00,
  "exhaustion_date": null,
  "breakdown": [
    { "resource": "cpu_core_hour", "credits": 500.00 },
    { "resource": "gpu_a100_hour", "credits": 3000.00 },
    { "resource": "storage_gb_month", "credits": 700.00 }
  ]
}
```

### 8.5 Webhook Events

| Event | Trigger |
|-------|---------|
| `credit.granted` | New credit grant created |
| `credit.low_balance` | Balance drops below configured threshold |
| `credit.exhausted` | Balance reaches zero |
| `credit.expiring` | Credits entering grace period before expiry |
| `credit.expired` | Credits expired and removed from balance |
| `credit.topped_up` | Auto-reload or manual top-up completed |
| `credit.void` | Grant voided by administrator |
| `contract.period_start` | New commitment period begins |
| `contract.true_up` | True-up invoice generated |
| `budget.warning` | Budget threshold crossed |
| `budget.exceeded` | Budget fully consumed |

---

## 9. Integration with Block Runtime

### 9.1 Balance-Check Block

The Meteridian block runtime (METR-0002) processes usage events through a pipeline
of blocks. The credit system introduces a **balance-check block** that is inserted
into the pipeline when credit-based billing is active for a tenant.

**Block behavior:**

1. Receives the usage event with its calculated credit cost (from the rating block).
2. Queries the Valkey balance for the tenant.
3. If `effective_balance >= credit_cost`: approves the event and decrements the
   balance atomically.
4. If `effective_balance < credit_cost`: behavior depends on cap mode:
   - Hard cap: rejects the event with a `402 Payment Required` error.
   - Soft cap: approves the event, marks it as "overage", and emits a
     `credit.exhausted` event if the balance crosses zero.

The balance-check block operates with **optimistic concurrency**. Under high
throughput, multiple events may read the same balance concurrently. Valkey's atomic
`DECRBY` ensures correctness — if the decrement would result in a negative balance
(hard cap mode), the operation is rejected and the event is retried or failed.

### 9.2 Atomic Balance Updates

All balance mutations use Valkey `MULTI/EXEC` transactions:

```
MULTI
  DECRBY credit:balance:{tenant_id} {cost}
  DECRBY credit:grant:{grant_id}:remaining {cost}
  LPUSH credit:ledger:{tenant_id} {transaction_json}
EXEC
```

For drawdown across multiple grants (partial consumption), a Lua script ensures
atomicity:

```lua
-- Drawdown across grants (FIFO by expiry)
local remaining_cost = tonumber(ARGV[1])
local grants = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', '+inf')
for _, grant_id in ipairs(grants) do
    local grant_remaining = tonumber(redis.call('GET', 'credit:grant:' .. grant_id .. ':remaining'))
    local to_consume = math.min(remaining_cost, grant_remaining)
    redis.call('DECRBY', 'credit:grant:' .. grant_id .. ':remaining', to_consume)
    remaining_cost = remaining_cost - to_consume
    if remaining_cost <= 0 then break end
end
redis.call('DECRBY', KEYS[2], tonumber(ARGV[1]) - remaining_cost)
return remaining_cost
```

### 9.3 Reconciliation

Despite Valkey's atomicity guarantees, the real-time balance can drift from the
ledger under edge conditions (network partitions, Valkey failover, application
crashes mid-transaction). A periodic reconciliation job ensures consistency:

1. Runs every 15 minutes (configurable).
2. Computes the expected balance from the PostgreSQL ledger (`SUM(amount)` for
   the tenant's transactions).
3. Compares with the Valkey balance.
4. If the difference exceeds a configurable tolerance (default: 0.01 credits):
   - Logs a reconciliation event with the discrepancy.
   - Resets the Valkey balance to match the ledger.
   - Emits a `credit.reconciled` internal event for monitoring.

Reconciliation is idempotent and safe to run concurrently (uses a distributed
lock via Valkey `SET NX`).

---

## 10. Open Questions

### 10.1 Grant Immutability

Should credit grants be strictly immutable (append-only ledger with no modifications
to existing grants), or should administrative corrections be allowed (e.g., adjusting
a grant's amount after creation)?

**Option A — Strict immutability:** All corrections are modeled as new transactions
(void + re-grant). Simpler audit trail but more verbose.

**Option B — Controlled mutability:** Grants can be adjusted by administrators, with
the adjustment recorded as an audit log entry. More ergonomic but complicates the
audit model.

Current leaning: Option A (strict immutability) for the initial implementation, with
adjustment transactions as the correction mechanism.

### 10.2 Partial Credit Consumption

How should the system handle events where the credit cost exceeds the available
balance in soft-cap mode?

- **Full event**: The entire event is processed; the balance goes negative. Simple
  but allows unbounded negative balances.
- **Split event**: The event is split into a "credited" portion and an "overage"
  portion. More accurate for reporting but adds complexity.
- **Reject and re-rate**: The event is rejected, re-rated at overage rates, and
  billed as pay-as-you-go. Clean separation but adds latency.

### 10.3 Multi-Currency Credit Pools

Should a tenant be able to hold credit grants in multiple currencies simultaneously?

- **Single currency**: All grants are converted to the tenant's base currency at
  grant time. Simple but introduces exchange rate risk.
- **Multi-currency**: Separate balance per currency. The system selects the
  appropriate pool based on the usage event's billing currency. More flexible but
  significantly more complex.

### 10.4 Credit Fungibility Across Products

Can credits granted for Product A be used to pay for Product B? This is particularly
relevant for platform vendors that sell multiple products (e.g., compute and AI
services under the same umbrella).

- **Fully fungible**: Credits are interchangeable across all products. Simpler model.
- **Product-scoped**: Credits are restricted to specific products or product
  categories. Prevents misallocation but adds management overhead.
- **Hybrid**: A portion of credits is fungible ("general credits") and a portion is
  product-scoped ("earmarked credits").

---

## 11. References

### Industry Credit Models

- [OpenAI API Billing](https://platform.openai.com/docs/guides/production-best-practices) —
  Prepaid credit model with per-token pricing by model.
- [Snowflake Credit Model](https://docs.snowflake.com/en/user-guide/credits) —
  Compute credits consumed by virtual warehouses, with capacity and on-demand pricing.
- [Databricks DBU Model](https://www.databricks.com/product/pricing) —
  Databricks Units as a universal consumption metric across workload types.
- [AWS Enterprise Discount Program](https://aws.amazon.com/pricing/enterprise/) —
  Multi-year committed spend with volume discounts.
- [Azure MACC](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/track-consumption-commitment) —
  Microsoft Azure Consumption Commitment for committed spend tracking.

### Billing Platform Implementations

- [Orb Prepaid Billing](https://docs.withorb.com/guides/invoicing/prepaid-credits) —
  Credit grants, drawdown, expiry, and top-up mechanics.
- [Stripe Billing Credit Grants](https://stripe.com/docs/billing/customer/credit-grants) —
  Credit balance management within Stripe's billing infrastructure.
- [Metronome Credits](https://docs.metronome.com/) —
  Committed and prepaid billing with contract management.

### FinOps Guidance

- [FinOps Foundation — Managing Commitment-Based Discounts](https://www.finops.org/framework/capabilities/manage-commitment-based-discounts/) —
  Best practices for committed spend management.
- [FinOps Foundation — Budgeting and Forecasting](https://www.finops.org/framework/capabilities/budget-management/) —
  Budget guardrails and spend alert patterns.
