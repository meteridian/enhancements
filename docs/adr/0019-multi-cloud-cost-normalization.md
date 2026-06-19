# ADR-0019: Multi-Cloud Cost Normalization at Ingestion Boundary

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq, @jordigilh
- **Related:** METR-0012 (Multi-Cloud and Hybrid Metering), ADR-0004 (CloudEvents), ADR-0013 (Two-Layer Data Architecture)

## Context and Problem Statement

Meteridian needs to support unified billing for AI workloads that span multiple
cloud providers (AWS, Azure, GCP) and on-premise infrastructure. Each cloud
provider reports costs, metrics, and usage in different formats, units,
currencies, and time granularities. The platform must normalize these
heterogeneous data sources into a single, consistent model for rating, billing,
and reporting.

The key design question is: **where in the data pipeline should normalization
occur?**

## Decision Drivers

- **Query performance** — Reports spanning multiple clouds must be fast
- **Data freshness** — Real-time balance management needs immediate cost estimates
- **Accuracy** — Billing-grade accuracy requires reconciliation with authoritative data
- **Extensibility** — Adding a new cloud provider should not require core changes
- **Auditability** — Original cloud-specific data must be preserved for disputes

## Considered Options

### Option 1: Normalize at Ingestion (Source Block Boundary)

Each cloud source block (Redpanda Connect pipeline) transforms provider-specific
data into Meteridian's canonical CloudEvents format before it enters the core
pipeline. Normalization logic lives in the source block configuration (Bloblang
mappings).

### Option 2: Normalize at Query Time (Reporting Layer)

Store raw cloud-specific data in TimescaleDB with provider-specific schemas.
Normalize on-the-fly during report generation using SQL views or query-time
transformations.

### Option 3: Normalize in a Dedicated Middleware Layer

Insert a "normalization service" between source blocks and the core pipeline.
This service maintains provider-specific adapters and emits normalized events.

## Decision Outcome

**Chosen option: Option 1 — Normalize at Ingestion (Source Block Boundary)**

### Rationale

1. **Consistency with existing architecture** — The two-layer architecture
   (ADR-0013) already defines source blocks as the transformation boundary.
   Cloud normalization is a natural extension of this pattern.

2. **Downstream simplicity** — The rating engine, credit system, and reporting
   layer never see cloud-specific formats. They operate on a single canonical
   schema. This eliminates cloud-specific branching in core business logic.

3. **Performance** — Normalized data is queryable without runtime transformation.
   Cross-cloud aggregation queries are simple GROUP BYs on standard columns,
   not complex UNION queries with per-provider normalization.

4. **Extensibility** — Adding a new cloud provider means adding a new Redpanda
   Connect YAML configuration file. No core code changes. No new database schemas.

5. **Auditability** — Original cloud-specific data is preserved in the
   CloudEvent's `data` payload alongside normalized fields. The `source` and
   `meteridiancloud` attributes identify provenance.

### Trade-offs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Performance | Fast queries, no runtime transformation | Slightly larger event payloads (both raw and normalized data) |
| Accuracy | Immediate normalized costs for enforcement | May need reconciliation when authoritative billing data arrives later |
| Extensibility | New providers via YAML configuration | Each provider's source block must implement full normalization |
| Auditability | Original data preserved in CloudEvent | Duplicate data stored (raw + normalized) |
| Maintenance | Core is cloud-agnostic | Normalization logic distributed across source block configs |

### Consequences

**Positive:**
- Rating engine remains cloud-agnostic (METR-0003)
- Credit deduction works identically for all clouds (METR-0004)
- Cross-cloud reports are standard SQL aggregations
- New cloud providers do not require schema migrations

**Negative:**
- Source block configurations must be maintained per-provider
- Exchange rate changes require re-processing if historical accuracy is needed
- Large schema changes in cloud APIs require source block updates

**Mitigations:**
- Provider-specific source blocks are YAML configurations, not code — updates
  are low-risk and do not require redeployment of Meteridian core
- Daily reconciliation process compares ingestion-time estimates with
  authoritative billing data and adjusts if variance exceeds threshold
- Exchange rate history is maintained to support retroactive corrections

## Links

- [METR-0012: Multi-Cloud and Hybrid Metering](../../enhancements/0012-multi-cloud-metering/multi-cloud-metering.md)
- [ADR-0004: CloudEvents as Canonical Event Format](0004-cloudevents-event-format.md)
- [ADR-0013: Two-Layer Data Architecture](0013-two-layer-data-architecture.md)
