# ADR-0020: FOCUS v1.4 as Canonical Cost Data Interchange Format

- **Status:** accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq, @jordigilh
- **Related:** [ADR-0004](0004-cloudevents-event-format.md), [ADR-0013](0013-two-layer-data-architecture.md), [ADR-0019](0019-multi-cloud-cost-normalization.md), [METR-0003](../../enhancements/0003-product-catalog/product-catalog.md), [METR-0012](../../enhancements/0012-multi-cloud-metering/multi-cloud-metering.md)

---

## Context and Problem Statement

Meteridian must ingest cost and usage data from an unbounded set of cloud, SaaS,
and infrastructure providers. Building and maintaining native integrations for
every provider does not scale — each integration requires ongoing maintenance as
provider APIs and billing formats evolve.

We need a vendor-neutral interchange format that:

1. Allows any provider to onboard by exporting a standardized file (no custom
   integration required).
2. Maps cleanly to Meteridian's internal CloudEvents schema and Product Catalog.
3. Supports both ingestion (import) and export (emit Meteridian data for
   external FinOps tools).
4. Is backed by an industry consortium with broad vendor adoption.

## Decision Drivers

- **Provider coverage without engineering effort.** Any provider exporting FOCUS
  data should be immediately usable without writing a custom source block.
- **Industry adoption.** AWS, Azure, GCP, Oracle Cloud, and others already
  export billing data in FOCUS format.
- **Interoperability.** Enterprises use multiple FinOps tools (CloudHealth,
  Apptio, Vantage). Emitting FOCUS data ensures Meteridian plays well in
  heterogeneous toolchains.
- **Future-proofing.** The FinOps Foundation (under the Linux Foundation)
  maintains FOCUS with broad vendor participation, ensuring long-term viability.
- **Schema completeness.** FOCUS v1.4 covers cloud, SaaS, AI, and data center
  cost dimensions — matching Meteridian's scope.

## Considered Options

1. **FOCUS v1.4** — FinOps Open Cost and Usage Specification (ratified
   June 4, 2026).
2. **Custom Meteridian interchange format** — define our own CSV and JSON schema.
3. **CloudEvents-only** — require all providers to emit CloudEvents directly.
4. **FinOps FOCUS v1.0** — earlier version with fewer columns and less vendor
   adoption.

## Decision Outcome

**Chosen option: FOCUS v1.4**, because it provides the broadest vendor adoption,
covers all required cost dimensions, and eliminates the need for per-provider
custom integrations for basic cost visibility.

### Consequences

**Positive:**

- Any provider that exports FOCUS-format billing data is immediately supported
  via CSV upload or API ingestion (Tier Custom in METR-0012).
- Meteridian can emit billing data in FOCUS format, enabling interoperability
  with external FinOps tools without custom export adapters.
- Reduces the number of native integrations needed to cover the market — native
  integrations are reserved for providers where real-time metering or deeper
  metadata is required.
- Aligns with an industry standard backed by AWS, Microsoft, Google, Oracle,
  and the FinOps Foundation.

**Negative:**

- FOCUS is a cost reporting schema, not a metering schema. Real-time metering
  still requires native integrations or CloudEvents.
- FOCUS v1.4 may not cover all dimensions needed for advanced use cases (e.g.,
  detailed GPU utilization metrics). Native integrations supplement FOCUS for
  these cases.
- Schema evolution (future FOCUS versions) requires migration tooling.

**Neutral:**

- FOCUS data arrives as periodic CSV exports (daily or hourly), not as
  real-time event streams. This is acceptable for cost management use cases
  where near-real-time (hourly) granularity is sufficient.

## Implementation

### Ingestion Pipeline

FOCUS data enters Meteridian through two paths:

1. **CSV upload** — operators upload FOCUS-format CSV files via the
   `/api/v1/ingress/focus` endpoint or via S3-compatible bucket polling.
2. **API ingestion** — providers push FOCUS-format JSON records via the
   `/api/v1/ingress/focus/stream` endpoint.

The FOCUS ingestion source block (a Redpanda Connect component) performs:

1. **Schema validation** — validates incoming records against the FOCUS v1.4
   column specification.
2. **Column mapping** — maps FOCUS columns to Meteridian CloudEvents attributes
   and Product Catalog resource types.
3. **Deduplication** — uses `BillingAccountId + ChargePeriodStart +
   ChargePeriodEnd + ResourceId + ChargeCategory` as a composite deduplication
   key.
4. **Emit CloudEvents** — transformed records are emitted as CloudEvents into
   the standard Meteridian pipeline for rating, distribution, and reporting.

### FOCUS to CloudEvents Mapping

| FOCUS column | CloudEvents attribute | Notes |
|--------------|----------------------|-------|
| `BillingAccountId` | `tenantid` (extension) | Tenant resolution |
| `ServiceName` | `type` (prefix: `focus.cost.`) | Event type routing |
| `ServiceCategory` | `subject` | Event subject |
| `ResourceId` | `source` | Event source identifier |
| `ChargePeriodStart` | `time` | Event timestamp |
| `BilledCost` | `data.billed_cost` | Event payload |
| `EffectiveCost` | `data.effective_cost` | Event payload |
| `PricingQuantity` | `data.quantity` | Event payload |
| `PricingUnit` | `data.unit` | Maps to resource type registry |
| `Provider` | `data.provider` | Provider identification |
| `Region` | `data.region` | Geographic dimension |
| `ResourceType` | `data.resource_type` | Maps to catalog resource type |
| `Tags` | `data.tags` | Preserved as key-value map |

### FOCUS to Product Catalog Mapping

| FOCUS column | Catalog entity | Field |
|--------------|---------------|-------|
| `ServiceName` | `catalog.products` | `name` |
| `ServiceCategory` | `catalog.products` | `category` |
| `ResourceType` | `catalog.resource_types` | `name` |
| `PricingUnit` | `catalog.resource_types` | `unit_of_measure` |
| `ListUnitPrice` | `catalog.tiers` or `catalog.charges` | `unit_price` |
| `ChargeCategory` | `catalog.charges` | `charge_model` (derived) |

The `resource_type_mappings` table supports a `provider = 'focus'` entry that
defines how FOCUS columns map to each canonical resource type.

### Export Pipeline

Meteridian emits billing data in FOCUS v1.4 format via:

1. **Scheduled export** — daily or monthly FOCUS-format CSV files written to an
   S3-compatible bucket.
2. **API export** — `/api/v1/export/focus` endpoint returns FOCUS-format JSON
   for a specified time range and tenant.
3. **Streaming export** — Redpanda Connect sink block that continuously emits
   rated events in FOCUS format to an external system.

### Schema Versioning

- The ingestion pipeline declares a `focus_version` field (default: `1.4`).
- When FOCUS publishes new versions, a migration adapter maps older columns to
  the new schema.
- Meteridian stores the original FOCUS version alongside ingested records for
  auditability.
- Export always emits the latest supported FOCUS version unless a specific
  version is requested via the `?focus_version=` query parameter.

## Links

- [FOCUS Specification v1.4](https://focus.finops.org/) — FinOps Foundation
- [FOCUS GitHub Repository](https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec)
- [FinOps Foundation](https://www.finops.org/)
- [ADR-0004: CloudEvents as canonical event format](0004-cloudevents-event-format.md)
- [ADR-0019: Multi-cloud cost normalization](0019-multi-cloud-cost-normalization.md)
- [METR-0003: Product and Service Catalog](../../enhancements/0003-product-catalog/product-catalog.md)
- [METR-0012: Multi-Cloud and Hybrid Metering](../../enhancements/0012-multi-cloud-metering/multi-cloud-metering.md)
