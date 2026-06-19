# ADR-0012: Tenant-Isolated Event Batches

## Status

**Accepted** — 2026-06-18

## Deciders

- @pgarciaq
- @jordigilh

## Context

Meteridian is a multi-tenant metering and billing platform. Apache Arrow
RecordBatches flow between blocks in the processing pipeline (see
[METR-0002](../../enhancements/0002-extensibility/extensibility.md)). Arrow's
columnar format provides excellent throughput for analytical workloads, but
introduces a fundamental architectural question: should a single RecordBatch
contain data from multiple tenants or from exactly one tenant?

This is a tension between performance and security:

- **Multi-tenant batches** maximize throughput. Larger batches amortize the
  fixed overhead of Arrow IPC serialization, schema validation, and block
  invocation. Packing events from all tenants into a single batch yields the
  highest possible throughput.
- **Single-tenant batches** maximize isolation. Each batch contains data from
  exactly one tenant, making it impossible for a buggy or malicious block to
  leak data across tenant boundaries.

For a billing platform handling financial data subject to regulatory
requirements (GDPR, SOC 2, data residency), the security implications of
cross-tenant data mixing are severe. A single bug in a block's filtering logic
could expose one tenant's usage data, pricing, or invoice details to another
tenant.

### Additional Considerations

- **Field-level security projections** are applied per-tenant with different
  field visibility rules based on subscription tier or regulatory jurisdiction.
- **Rate plans and cost models** differ per tenant. Multi-tenant batches require
  per-row rate plan lookups, adding complexity and error surface.
- **Data residency requirements** (GDPR Art. 25) may require that certain
  tenants' data never leaves a specific processing boundary.
- **Blast radius**: A bug in a multi-tenant batch affects all tenants in that
  batch. Single-tenant batches limit the blast radius to one tenant.

## Decision

### Single-Tenant RecordBatches

Each Arrow RecordBatch flowing through the block pipeline contains data from
exactly **one** tenant. The `tenant_id` column is present in every batch but is
uniform — all rows share the same value.

This is a **security and data safety requirement**, not a performance
optimization. The marginal throughput gain from multi-tenant batches does not
justify the risk of cross-tenant data leakage in a billing system.

### Tenant Identity Validation

The `tenant_id` is validated at pipeline ingress (the first block in the DAG).
The ingress block verifies that:

1. The `tenant_id` is present and non-empty.
2. The `tenant_id` corresponds to a known, active tenant.
3. All rows in the incoming batch belong to the same tenant.

Downstream blocks can trust the `tenant_id` without re-validation, reducing
per-block overhead.

### Batch Size Targets

To amortize Arrow IPC overhead while maintaining reasonable latency:

| Parameter          | Target          | Rationale                                |
|--------------------|-----------------|------------------------------------------|
| Rows per batch     | 1,000–10,000    | Balances IPC overhead vs. processing latency |
| Max batch size     | 10,000 rows     | Caps memory usage per batch              |
| Min batch size     | 1 row           | Supports low-volume tenants              |
| Batch flush timeout| 5 seconds       | Ensures low-volume tenants don't wait indefinitely |

For high-volume tenants generating millions of events per hour, the ingress
block partitions the incoming event stream into multiple batches of up to 10,000
rows each, all tagged with the same `tenant_id`.

For low-volume tenants, the batch flush timeout ensures that events are not
buffered indefinitely waiting for the batch to fill. After 5 seconds without new
events for a tenant, the current batch is flushed regardless of size.

### Future Optimization: Same-Tenant Bin-Packing

A future optimization (tracked in ROADMAP.md) may implement bin-packing of
same-tenant events into larger batches when events arrive individually or in
small groups. This amortizes per-batch overhead without crossing tenant
boundaries. The bin-packing strategy:

- Groups queued events by `tenant_id`.
- Packs same-tenant events into batches up to the target size.
- Respects the flush timeout to bound latency.

This optimization is deferred because the current batch size targets provide
acceptable performance for the expected event volumes at launch.

### Schema Enforcement

Every RecordBatch includes the following metadata columns enforced by the
platform:

| Column        | Type      | Description                              |
|---------------|-----------|------------------------------------------|
| `tenant_id`   | `utf8`    | Tenant identifier (uniform within batch) |
| `event_id`    | `utf8`    | CloudEvents `id` (idempotency key, see ADR-0011) |
| `event_time`  | `timestamp[us, tz=UTC]` | Event timestamp               |
| `event_type`  | `utf8`    | CloudEvents `type`                       |

Blocks may add additional columns but must not remove or modify these platform
columns.

## Consequences

### Positive

- **Tenant isolation by construction**: It is structurally impossible for a
  block to accidentally process or emit data belonging to a different tenant.
  The isolation guarantee is enforced by the data format itself, not by
  per-block logic.
- **Simplified block logic**: Block authors do not need to filter by
  `tenant_id`, look up per-tenant configuration for multiple tenants in a
  single invocation, or worry about partial failures affecting other tenants.
  Every block invocation operates on exactly one tenant's data.
- **Simplified field-level security**: Per-tenant field projections can be
  applied once at ingress rather than per-row within each block.
- **Regulatory compliance**: Data residency and processing boundary requirements
  are trivially enforced by routing batches based on the uniform `tenant_id`.
- **Failure isolation**: A block crash or error affects only the tenant whose
  batch was being processed. Other tenants' data continues flowing.

### Negative

- **Reduced batch efficiency for small tenants**: Tenants with low event volumes
  produce small batches (potentially single-row), where the fixed overhead of
  Arrow IPC serialization dominates. The bin-packing optimization (deferred)
  mitigates this.
- **Higher batch count**: The total number of batches in the system is higher
  than in a multi-tenant model, increasing scheduling and coordination
  overhead. For N tenants each producing M events per second, the system
  processes at least N batches per flush interval rather than 1.
- **Memory overhead**: Each batch carries its own Arrow schema metadata. With
  many small batches, the aggregate schema metadata overhead is higher than
  with fewer large multi-tenant batches. In practice, Arrow schemas are small
  (hundreds of bytes) and this is negligible.

### Neutral

- **No impact on block API**: The block processing interface is the same
  regardless of whether batches are single-tenant or multi-tenant. Blocks
  receive a RecordBatch and emit zero or more RecordBatches. This decision is
  transparent to block authors beyond the guarantee that `tenant_id` is
  uniform.
- **Monitoring and observability**: Per-tenant metrics (throughput, latency,
  error rates) are naturally available because each batch is already
  tenant-scoped. This is equally achievable with multi-tenant batches via
  per-row grouping, but single-tenant batches make it trivial.

## Alternatives Considered

### Multi-Tenant Batches with Column-Based Isolation

Pack events from multiple tenants into a single RecordBatch. Blocks filter by
the `tenant_id` column to process each tenant's data separately. This maximizes
batch sizes and throughput.

However, the isolation guarantee depends entirely on every block correctly
filtering by `tenant_id`. A single bug — a missing filter, an incorrect join, a
logging statement that dumps the full batch — leaks data across tenant
boundaries. For financial billing data, this risk is unacceptable.

**Rejected**: One bug = cross-tenant data leak; unacceptable for billing data.

### Single-Event Processing (No Batching)

Process each event individually, eliminating batch-level concerns entirely. This
provides perfect isolation and simplicity but sacrifices throughput by
100–1000x. Arrow's columnar format provides no benefit for single-row batches;
the serialization overhead dominates. Billing pipelines routinely process
millions of events per hour, making single-event processing infeasible at scale.

**Rejected**: Correct but 100–1000x slower; infeasible at production scale.

### Tenant-Sharded Pipelines

Deploy a separate, independent pipeline instance for each tenant. This provides
perfect isolation (separate processes, separate memory spaces) but requires
O(N) resources for N tenants. For a platform serving thousands of tenants, this
does not scale. It also complicates deployment, monitoring, and upgrades
(rolling out a block update requires updating N pipeline instances).

**Rejected**: O(N) resource usage; does not scale beyond hundreds of tenants.

## References

- [Apache Arrow RecordBatch Specification](https://arrow.apache.org/docs/format/Columnar.html) — Columnar format and IPC details
- [GDPR Article 25](https://gdpr-info.eu/art-25-gdpr/) — Data protection by design and by default
- [Stripe's Tenant Isolation Architecture](https://stripe.com/blog/engineering) — Industry reference for tenant isolation in financial systems
- [ADR-0004: CloudEvents Envelope](0004-cloudevents-event-format.md)
- [ADR-0011: At-Least-Once Delivery with Idempotency Keys](0011-at-least-once-idempotency.md)
- [METR-0002: Block-Based Extensibility](../../enhancements/0002-extensibility/extensibility.md)
