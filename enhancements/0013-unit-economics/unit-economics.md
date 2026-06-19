# METR-0013: Unit Economics and Business Telemetry

**Status:** draft

**Authors:** Meteridian contributors

**Created:** 2026-06-19

**Related:**
- [METR-0002: Platform Extensibility](../0002-extensibility/extensibility.md)
- [METR-0003: Product and Service Catalog](../0003-product-catalog/product-catalog.md)
- [METR-0012: Multi-Cloud and Hybrid Metering](../0012-multi-cloud-metering/multi-cloud-metering.md)
- [ADR-0004: CloudEvents as canonical event format](../../docs/adr/0004-cloudevents-event-format.md)
- [ADR-0021: Unlimited custom dimensions via CloudEvents](../../docs/adr/0021-unlimited-custom-dimensions.md)

---

## Summary

This enhancement adds a unit economics module to Meteridian that enables
organizations to calculate and track cost per business unit — cost per customer,
per feature, per transaction, per API call, per inference request, or any other
business metric.

The core formula is straightforward: `spend ÷ business metric = unit cost`.
What distinguishes Meteridian from competitors like CloudZero is the
**block-based extensibility** of the cost allocation step. Where CloudZero
offers a fixed division formula, Meteridian allows users to insert custom
Transform blocks (METR-0002) with arbitrary allocation logic: weighted splits,
ML-based attribution, multi-factor regression, time-window amortization, and
any algorithm the user can express in Go or Python.

---

## Motivation

### Why Unit Economics Matters

Infrastructure spend is meaningless without business context. Knowing that
"we spent $47,000 on AWS this month" tells a CFO nothing about efficiency.
Knowing that "our cost per active customer dropped from $2.31 to $1.87
while serving 15% more users" tells a story of operational improvement.

Unit economics transforms infrastructure cost data into business intelligence:

- **SaaS companies**: Cost per customer, cost per API call, cost per
  transaction — to ensure margins improve as scale increases
- **AI and ML platforms**: Cost per inference, cost per training run, cost per
  GPU-hour per model — to price AI services and track efficiency
- **Platform teams**: Cost per namespace, cost per deployment, cost per team —
  to drive accountability and optimize resource allocation
- **FinOps practitioners**: Cost per revenue dollar, cost per employee, cost per
  business unit — to benchmark operational efficiency

### Why Block-Based Extensibility Is the Differentiator

CloudZero's unit economics divides spend by a single business metric. This
works for simple cases but fails for:

- **Shared infrastructure**: A Kubernetes cluster serves 12 teams. How do you
  allocate the control plane overhead? Fixed division cannot express
  proportional allocation weighted by CPU and memory usage.
- **Multi-factor attribution**: An API call consumes compute, storage, network,
  and a third-party service. The cost is not simply `total ÷ calls` — it
  requires multi-factor attribution across resource types.
- **Time-window amortization**: A training run costs $50,000 over 3 days but
  benefits the model for 6 months. Unit cost per inference should amortize
  the training cost over the model's lifetime, not spike during the training
  window.
- **ML-based attribution**: When hundreds of microservices contribute to a
  single customer request, statistical attribution models (Shapley values,
  Markov chains) provide more accurate cost allocation than simple division.

Meteridian's block-based dataflow (METR-0002) solves all of these. Users insert
Transform blocks between the spend aggregation and the unit cost calculation.
Each block can implement arbitrary logic — and blocks can be chained,
composed, and shared via the block marketplace.

---

## Design

### 1. Business Telemetry Ingestion

Business metrics (daily active users, orders, API calls, inference counts, etc.)
are ingested as a separate event stream distinct from infrastructure metering
events.

#### Event Format

Business telemetry events use the CloudEvents specification (ADR-0004) with
the `meteridian.business.telemetry` event type:

```json
{
  "specversion": "1.0",
  "type": "meteridian.business.telemetry",
  "source": "//salesforce.example.com/orders",
  "id": "bt-2026-06-19-001",
  "time": "2026-06-19T00:00:00Z",
  "datacontenttype": "application/json",
  "mtrnmetric": "daily_active_users",
  "mtrngranularity": "daily",
  "mtrndimensions": {
    "product": "enterprise",
    "region": "us-east-1",
    "team": "platform"
  },
  "data": {
    "value": 14523,
    "unit": "users"
  }
}
```

The `mtrn`-prefixed extension attributes follow ADR-0021 (unlimited custom
dimensions). The `mtrnmetric` attribute identifies the business metric by name.
The `mtrngranularity` attribute declares the time granularity (hourly, daily,
weekly, monthly).

#### Ingestion Channels

| Channel | Method | Use Case |
|---------|--------|----------|
| **REST API** | `POST /api/v1/telemetry/business` | Real-time metric push from applications |
| **CSV upload** | `POST /api/v1/telemetry/business/upload` | Batch import from spreadsheets and data warehouses |
| **Source block** | Redpanda Connect pipeline | Continuous ingestion from databases, APIs, and message queues |
| **CloudEvents batch** | `POST /api/v1/events` (standard) | Mixed event batches containing both metering and business telemetry |

The CSV upload format accepts columns: `timestamp`, `metric_name`, `value`,
`unit`, and any number of dimension columns. Unknown columns are stored as
custom dimensions automatically.

#### Storage

Business telemetry events are stored in TimescaleDB alongside metering events
but in a dedicated hypertable (`events.business_telemetry`) partitioned by time.
This separation enables independent retention policies and avoids contaminating
the metering event stream.

```sql
CREATE TABLE events.business_telemetry (
    event_id       UUID DEFAULT gen_random_uuid(),
    event_time     TIMESTAMPTZ NOT NULL,
    metric_name    TEXT NOT NULL,
    metric_value   NUMERIC NOT NULL,
    metric_unit    TEXT NOT NULL,
    granularity    TEXT NOT NULL DEFAULT 'daily',
    source         TEXT NOT NULL,
    dimensions     JSONB DEFAULT '{}',
    created_at     TIMESTAMPTZ DEFAULT now()
);

SELECT create_hypertable('events.business_telemetry', 'event_time');

CREATE INDEX idx_bt_metric_time ON events.business_telemetry (metric_name, event_time DESC);
CREATE INDEX idx_bt_dimensions ON events.business_telemetry USING GIN (dimensions);
```

### 2. Unit Cost Calculation Engine

The unit cost engine joins infrastructure spend with business metrics to
produce cost-per-X time series.

#### Core Pipeline

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Spend           │     │  Allocation      │     │  Unit Cost       │
│  Aggregation     │────▶│  Transform       │────▶│  Calculation     │
│                  │     │  (block chain)   │     │                  │
│  SUM(cost) by    │     │  Custom blocks   │     │  allocated_cost  │
│  time + dims     │     │  for advanced    │     │  ÷ metric_value  │
│                  │     │  allocation      │     │  = unit_cost     │
└──────────────────┘     └──────────────────┘     └──────────────────┘
        │                        │                        │
        ▼                        ▼                        ▼
   Cost summaries          Allocated costs          Unit cost series
   (from metering)         (enriched)               (output)
```

1. **Spend aggregation**: The engine queries summarized infrastructure costs
   from metering data, grouped by the requested time granularity and
   dimensions (project, cluster, service, team, etc.).

2. **Allocation transform** (optional): If custom allocation logic is
   configured, the aggregated spend flows through a chain of Transform blocks
   (METR-0002). Each block can redistribute, weight, split, or amortize costs
   before the division step.

3. **Unit cost calculation**: The (optionally transformed) spend is divided by
   the corresponding business metric value for each time period:

   ```
   unit_cost[t] = allocated_cost[t] / metric_value[t]
   ```

#### Default Behavior (No Custom Blocks)

Without custom allocation blocks, the engine performs direct division:

```sql
SELECT
    s.usage_start AS period,
    s.cost_total AS spend,
    bt.metric_value AS metric,
    s.cost_total / NULLIF(bt.metric_value, 0) AS unit_cost
FROM cost_summaries s
JOIN events.business_telemetry bt
    ON s.usage_start = bt.event_time
    AND bt.metric_name = :metric_name
WHERE s.usage_start BETWEEN :start AND :end
ORDER BY s.usage_start;
```

This matches CloudZero's baseline behavior.

### 3. Multi-Dimensional Unit Cost Views

Unit costs can be stacked across any combination of dimensions supported by
the underlying metering and business telemetry data.

#### Dimension Stacking

```
cost_per_customer_per_feature_per_team =
    cost[customer=X, feature=Y, team=Z] / metric[customer=X]
```

Dimensions from infrastructure metering (namespace, cluster, node, service)
and business telemetry (customer, feature, product) can be freely combined.
The unlimited custom dimensions (ADR-0021) ensure that any dimension the user
attaches to events is immediately available for grouping and filtering.

#### Example Queries

**Cost per customer per month:**
```
GET /api/v1/unit-costs/?metric=monthly_active_customers
    &group_by[customer]=*
    &filter[time_scope_units]=month
    &filter[time_scope_value]=-3
```

**Cost per API call by service and region:**
```
GET /api/v1/unit-costs/?metric=api_calls
    &group_by[service]=*
    &group_by[region]=*
    &filter[time_scope_units]=month
    &filter[time_scope_value]=-1
```

**Cost per inference by model and GPU type:**
```
GET /api/v1/unit-costs/?metric=inference_requests
    &group_by[model]=*
    &group_by[gpu_type]=*
    &filter[time_scope_units]=day
    &filter[time_scope_value]=-30
```

### 4. Block-Based Cost Allocation Extensibility

This is the key differentiator. Users create custom Transform blocks (METR-0002)
that plug into the allocation step of the unit cost pipeline.

#### Built-In Allocation Blocks

Meteridian ships with these allocation blocks in the marketplace:

| Block | Category | Description |
|-------|----------|-------------|
| `proportional-split` | Allocation | Distributes cost proportionally by a usage metric (CPU, memory, storage, requests) |
| `weighted-split` | Allocation | Distributes cost by user-defined weights (e.g., 60% to team A, 40% to team B) |
| `even-split` | Allocation | Distributes cost evenly across N consumers |
| `time-window-amortize` | Amortization | Spreads a one-time cost (training run, migration) across a configurable time window |
| `sliding-average` | Smoothing | Replaces point-in-time cost with a sliding window average to reduce spikes |

#### Custom Allocation Block Interface

Custom blocks implement the standard Transform block interface (METR-0002)
with cost allocation-specific input and output schemas:

```go
type AllocationInput struct {
    Period     time.Time              `json:"period"`
    TotalCost  decimal.Decimal        `json:"total_cost"`
    CostType   string                 `json:"cost_type"`
    Dimensions map[string]string      `json:"dimensions"`
    Consumers  []ConsumerUsage        `json:"consumers"`
}

type ConsumerUsage struct {
    ID         string                 `json:"id"`
    Dimensions map[string]string      `json:"dimensions"`
    Usage      map[string]float64     `json:"usage"`
}

type AllocationOutput struct {
    Period      time.Time             `json:"period"`
    Allocations []CostAllocation      `json:"allocations"`
}

type CostAllocation struct {
    ConsumerID    string              `json:"consumer_id"`
    AllocatedCost decimal.Decimal     `json:"allocated_cost"`
    Dimensions    map[string]string   `json:"dimensions"`
    Method        string              `json:"method"`
}
```

#### Advanced Allocation Examples

**ML-Based Attribution (Shapley Values):**

```python
class ShapleyAttributionBlock(TransformBlock):
    """Distributes shared costs using Shapley value attribution.

    Uses historical usage patterns to compute each consumer's marginal
    contribution to shared resource consumption, then allocates costs
    proportionally to Shapley values.
    """

    def transform(self, input: AllocationInput) -> AllocationOutput:
        coalition_costs = self.compute_coalition_costs(input.consumers)
        shapley_values = self.compute_shapley_values(coalition_costs)
        return self.allocate_by_values(input, shapley_values)
```

**Multi-Factor Regression:**

```python
class RegressionAttributionBlock(TransformBlock):
    """Allocates costs using multi-factor linear regression.

    Fits a model: cost = β₁·cpu + β₂·memory + β₃·network + β₄·storage
    to historical data, then uses the coefficients to attribute cost
    to each consumer based on their resource consumption profile.
    """

    def transform(self, input: AllocationInput) -> AllocationOutput:
        coefficients = self.fit_regression(input)
        return self.allocate_by_regression(input, coefficients)
```

**Time-Window Amortization:**

```python
class AmortizationBlock(TransformBlock):
    """Amortizes one-time costs across a configurable time window.

    Useful for training runs, migrations, and setup costs that benefit
    the organization over a longer period than the billing cycle in
    which they occurred.
    """

    config_schema = {
        "amortization_window_days": {"type": "integer", "default": 180},
        "amortization_method": {"type": "string", "enum": ["linear", "declining_balance"]},
    }

    def transform(self, input: AllocationInput) -> AllocationOutput:
        daily_cost = input.total_cost / self.config.amortization_window_days
        return self.spread_cost(input, daily_cost)
```

### 5. Unlimited Custom Dimensions

Unit economics leverages the unlimited custom dimensions capability
(ADR-0021). Business telemetry events carry arbitrary dimensions via
CloudEvents extension attributes with the `mtrn` prefix:

```json
{
  "mtrncustomer": "acme-corp",
  "mtrnfeature": "real-time-analytics",
  "mtrnteam": "platform-eng",
  "mtrncostcenter": "CC-4200",
  "mtrnproject": "project-aurora"
}
```

These dimensions are stored in a `JSONB` column with GIN indexing, enabling
immediate queryability without schema changes or tag-enablement configuration.
Any dimension present on metering events or business telemetry events can be
used as a `group_by` or `filter` parameter in unit cost queries.

This contrasts with competitors:

| Platform | Custom Dimension Approach | Limit |
|----------|--------------------------|-------|
| CloudZero | Cost groups (manual UI config) | ~20 groups |
| Apptio | Fixed tag taxonomy | Limited to cloud tags |
| Kubecost | Kubernetes labels only | K8s label constraints |
| Vantage | Virtual tags (predefined) | ~30 virtual tags |
| **Meteridian** | CloudEvents extension attributes | **Unlimited** |

### 6. Product Catalog Integration

Unit cost metrics are expressible as SQL-based metric definitions in the
Product Catalog (METR-0003). This enables unit costs to drive billing directly:

```sql
-- Metric definition in Product Catalog
-- "Cost per active customer per month"
SELECT
    date_trunc('month', uc.period) AS billing_period,
    uc.dimensions->>'customer' AS customer_id,
    SUM(uc.unit_cost) AS billable_amount
FROM unit_cost_series uc
WHERE uc.metric_name = 'monthly_active_customers'
    AND uc.period BETWEEN :start AND :end
GROUP BY 1, 2
```

This closes the loop from metering → rating → unit economics → billing.
Organizations can bill customers based on their actual unit cost rather
than flat rates, enabling true cost-plus pricing models.

#### Catalog Resource Type

A new resource type `unit_cost_metric` is registered in the Product Catalog
resource type registry:

```yaml
name: unit_cost_metric
description: Derived metric computing cost per business unit
event_type: meteridian.unit_cost.calculated
dimensions:
  - metric_name
  - period
  - granularity
  - (any custom dimension from source events)
measures:
  - unit_cost
  - total_spend
  - metric_value
```

### 7. API Design

#### Business Telemetry API

```
POST   /api/v1/telemetry/business           # Submit business metric events
POST   /api/v1/telemetry/business/upload     # CSV batch upload
GET    /api/v1/telemetry/business/metrics     # List registered metrics
GET    /api/v1/telemetry/business/metrics/:name  # Get metric details and recent values
DELETE /api/v1/telemetry/business/metrics/:name  # Delete metric and its data
```

#### Unit Cost API

```
GET    /api/v1/unit-costs/                   # Query unit cost time series
GET    /api/v1/unit-costs/summary/           # Aggregated unit cost summary
GET    /api/v1/unit-costs/trends/            # Unit cost trends and deltas
GET    /api/v1/unit-costs/compare/           # Compare unit costs across dimensions
```

#### Allocation Pipeline API

```
GET    /api/v1/allocation-pipelines/          # List configured pipelines
POST   /api/v1/allocation-pipelines/          # Create a new pipeline
GET    /api/v1/allocation-pipelines/:id       # Get pipeline details
PUT    /api/v1/allocation-pipelines/:id       # Update pipeline configuration
DELETE /api/v1/allocation-pipelines/:id       # Delete pipeline
POST   /api/v1/allocation-pipelines/:id/dry-run  # Test pipeline with sample data
```

#### Unit Cost Response Format

```json
{
  "meta": {
    "metric": "daily_active_users",
    "granularity": "daily",
    "start": "2026-06-01",
    "end": "2026-06-19",
    "allocation_pipeline": "default",
    "total": {
      "spend": 47230.50,
      "metric_value": 14523,
      "unit_cost": 3.25,
      "unit_cost_delta": -0.12,
      "currency": "USD"
    }
  },
  "data": [
    {
      "date": "2026-06-01",
      "spend": 2485.30,
      "metric_value": 14102,
      "unit_cost": 0.1762,
      "dimensions": {}
    }
  ]
}
```

### 8. Data Model

#### Unit Cost Materialized View

```sql
CREATE MATERIALIZED VIEW unit_cost_series AS
SELECT
    date_trunc('day', s.usage_start) AS period,
    s.dimensions,
    bt.metric_name,
    SUM(s.cost_total) AS total_spend,
    AVG(bt.metric_value) AS metric_value,
    SUM(s.cost_total) / NULLIF(AVG(bt.metric_value), 0) AS unit_cost,
    s.currency
FROM cost_summaries s
JOIN events.business_telemetry bt
    ON date_trunc('day', s.usage_start) = date_trunc('day', bt.event_time)
WHERE bt.metric_value > 0
GROUP BY 1, 2, 3, 7;

CREATE UNIQUE INDEX idx_ucs_period_metric_dims
    ON unit_cost_series (period, metric_name, dimensions);
```

The materialized view is refreshed on a configurable schedule (default: hourly).
For real-time unit costs, the API falls back to a direct query against the
underlying tables.

#### Allocation Pipeline Configuration

```sql
CREATE TABLE catalog.allocation_pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    description     TEXT,
    metric_name     TEXT NOT NULL,
    cost_source     TEXT NOT NULL DEFAULT 'all',
    block_chain     JSONB NOT NULL DEFAULT '[]',
    schedule        TEXT NOT NULL DEFAULT '@hourly',
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
```

The `block_chain` field stores an ordered list of block references with
their configuration:

```json
[
  {
    "block": "marketplace://meteridian/proportional-split@1.0",
    "config": {
      "weight_metric": "cpu_core_hours",
      "fallback": "even-split"
    }
  },
  {
    "block": "marketplace://acme-corp/training-amortizer@2.1",
    "config": {
      "amortization_window_days": 180,
      "amortization_method": "linear"
    }
  }
]
```

### 9. Implementation Approach

#### Phase 1 (v1.0)

- Business telemetry ingestion (REST API and CSV upload)
- Direct unit cost calculation (no custom blocks)
- Multi-dimensional grouping and filtering
- Unit cost API with time series response
- Product Catalog integration (resource type registration)
- Built-in allocation blocks: `proportional-split`, `weighted-split`,
  `even-split`

#### Phase 2 (v1.1)

- Source block for continuous business telemetry ingestion
- Custom allocation block pipeline
- Block chaining and composition
- `time-window-amortize` and `sliding-average` blocks
- Dry-run and simulation mode
- Unit cost alerting (thresholds and anomaly detection)

#### Phase 3 (v2.0)

- ML-based attribution blocks (Shapley values)
- Multi-factor regression allocation
- Unit cost forecasting (Augurs integration)
- Unit cost-based billing in Product Catalog
- Visual pipeline editor for allocation chains

---

## Open Questions

1. **Granularity alignment**: When infrastructure costs are hourly but business
   metrics are daily, should the engine interpolate or aggregate? Current
   proposal: aggregate spend to match the coarser business metric granularity.

2. **Backfill semantics**: When a business metric is submitted retroactively
   (e.g., corrected DAU count for last week), should previously computed unit
   costs be recalculated? Proposal: yes, with versioned unit cost snapshots.

3. **Zero-metric periods**: When the business metric is zero for a period (e.g.,
   no API calls on a holiday), the unit cost is undefined. Should the API return
   `null`, `Infinity`, or omit the period? Proposal: return `null` with a
   warning field.

4. **Cross-tenant metrics**: Can a business telemetry metric span multiple
   tenants (e.g., a platform team tracking cost per internal customer across
   org units)? Current thinking: metrics are tenant-scoped; cross-tenant
   requires a federation layer.

5. **Block ordering sensitivity**: The order of blocks in an allocation pipeline
   affects the result. Should the API validate or warn about potentially
   problematic orderings (e.g., amortization before proportional split vs. after)?

---

## References

- [METR-0002: Platform Extensibility](../0002-extensibility/extensibility.md) —
  Block-based dataflow architecture enabling custom allocation logic
- [METR-0003: Product and Service Catalog](../0003-product-catalog/product-catalog.md) —
  SQL-based metric definitions and resource type registry
- [ADR-0004: CloudEvents as canonical event format](../../docs/adr/0004-cloudevents-event-format.md) —
  Event format for business telemetry
- [ADR-0021: Unlimited custom dimensions](../../docs/adr/0021-unlimited-custom-dimensions.md) —
  Custom dimension approach for multi-dimensional unit cost views
- CloudZero unit economics documentation — baseline capability to match and exceed
- Shapley value attribution in cost allocation — academic foundation for
  ML-based allocation blocks
