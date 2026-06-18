# ADR-0013: Two-Layer Data Architecture (Redpanda Connect + Block Runtime)

## Status

**Accepted** — 2026-06-18

## Deciders

- @pgarciaq
- @jordigilh

## Context

Meteridian's data pipeline has two fundamentally different concerns:

1. **Data collection**: Ingesting raw events from diverse sources — cloud
   provider APIs (AWS, Azure, GCP), Prometheus endpoints, Kafka topics, HTTP
   webhooks, S3 buckets, database change streams — and delivering them in a
   normalized format.
2. **Billing processing**: Transforming normalized events through a
   domain-specific pipeline — metering aggregation, rate plan evaluation,
   credit and balance management, invoice generation, distribution, and
   settlement.

These concerns have very different characteristics. Data collection is a
**solved problem** with excellent open-source tooling. Billing processing is
**domain-specific** and requires financial precision, tenant awareness, and
extensibility.

### Redpanda Connect

Redpanda Connect (formerly Benthos) is a stream processing framework with a
declarative YAML configuration language and 200+ connectors. It excels at
protocol translation (Kafka→HTTP, S3→AMQP), format conversion (CSV→JSON→Parquet),
fan-out/fan-in, content-based routing, and retry/error handling.

RC does **not** have: financial precision types, tenant-aware processing, cost
model evaluation, invoice generation, or stateful billing operations.

### Block Runtime

The block-based extensibility model
([METR-0002](../../enhancements/0002-extensibility/extensibility.md)) provides a
processing runtime for billing operations with stateful blocks, tenant isolation
([ADR-0012](0012-tenant-isolated-batches.md)), financial precision via Arrow
Decimal128, extensible DAG pipelines configured via CUE, and at-least-once
delivery with idempotency keys ([ADR-0011](0011-at-least-once-idempotency.md)).

### The Question

What is the boundary between Redpanda Connect and the block runtime? Where does
data collection end and billing processing begin?

## Decision

### Two-Layer Architecture

Meteridian adopts a two-layer data architecture with a clean boundary between
data collection and billing processing.

#### Layer 1 — Redpanda Connect (Data Collection)

Redpanda Connect handles all aspects of data collection:

- **Ingestion**: Reading events from diverse sources using RC's 200+ input
  connectors (Kafka, HTTP, S3, Prometheus, cloud provider APIs, etc.)
- **Normalization**: Converting source-specific formats into CloudEvents
  ([ADR-0004](0004-cloudevents-event-format.md)) with standardized fields
- **Delivery**: Producing normalized CloudEvent batches to the block runtime's
  ingress queue via Arrow IPC

RC pipelines are configured via **YAML** (Redpanda Connect's native
configuration format). Each RC pipeline is a self-contained YAML file defining
an input, optional processors, and an output.

RC is deployed as a **sidecar or standalone service**, separate from the
Meteridian core process. This allows independent scaling, upgrading, and
monitoring of the data collection layer.

#### Layer 2 — Block Runtime (Billing Processing)

The block runtime handles all billing-specific operations:

- **Metering aggregation**: Bucketing raw usage events into metered quantities
- **Rate plan evaluation**: Applying tenant-specific pricing models to usage
- **Credit and balance management**: Tracking prepaid credits and balances
- **Invoice generation**: Assembling line items, calculating taxes
- **Distribution and settlement**: Routing charges to cost centers

Block pipelines are configured via **CUE**
([METR-0002](../../enhancements/0002-extensibility/extensibility.md)). The block
runtime is the Meteridian core process.

### Boundary Definition

The boundary between the two layers is the **CloudEvents format**:

- **RC outputs CloudEvents**. The last step of every RC pipeline produces
  events in CloudEvents format with Arrow serialization.
- **Block runtime inputs CloudEvents**. The first block in every block pipeline
  receives CloudEvent RecordBatches.
- **RC does not know about tenants, rate plans, or cost models.** It is a
  generic data collection layer.
- **The block runtime does not know about Kafka consumer groups, HTTP retry
  policies, or S3 pagination.** It receives pre-normalized events.

| Aspect              | Layer 1 (RC)                 | Layer 2 (Block Runtime)       |
|---------------------|------------------------------|-------------------------------|
| Configuration       | YAML                         | CUE                           |
| Deployment          | Sidecar / standalone service | Core process                  |
| Scaling             | Independent (per-source)     | Independent (per-pipeline)    |
| State               | Stateless (offsets in broker) | Stateful (block state stores) |
| Upgrade cycle       | RC release cadence           | Meteridian release cadence    |

## Consequences

### Positive

- **Leverage existing ecosystem**: RC's 200+ connectors provide immediate
  connectivity to virtually every data source. Adding a new source requires
  only a new YAML configuration, not new Go code.
- **Clean separation of concerns**: Data collection and billing processing
  evolve independently. RC upgrades do not affect billing logic; block runtime
  changes do not affect data ingestion.
- **Independent scaling**: The collection layer scales horizontally per-source
  independently of the billing processing layer.
- **Independent failure domains**: An RC pipeline crash affects only that
  source. A block runtime issue does not affect ingestion — events queue in
  the ingress buffer.
- **Operational simplicity**: RC pipelines are declarative YAML. Operators can
  add or remove data sources without redeploying the core process.

### Negative

- **Additional operational component**: Running RC as a separate service adds
  deployment complexity (separate container, health checks, monitoring).
- **Serialization boundary**: Events cross a serialization boundary between
  RC and the block runtime (CloudEvents over Arrow IPC), adding latency
  compared to an in-process pipeline.
- **Two configuration languages**: Operators must understand YAML (for RC
  pipelines) and CUE (for block pipelines).

### Neutral

- **CloudEvents as the contract**: The CloudEvents format is the API contract
  between the two layers. This contract is stable (CloudEvents 1.0 is a CNCF
  graduated specification) and well-understood.
- **RC is replaceable**: Because the boundary is CloudEvents, the data
  collection layer could be replaced with any system that produces CloudEvents.
  Meteridian is not locked into Redpanda Connect, though there is no reason
  to replace it given its maturity and connector ecosystem.

## Alternatives Considered

### Block Runtime Handles Everything (No Redpanda Connect)

Build all data collection capabilities directly into the block runtime. Create
custom blocks for each data source (Kafka input block, S3 input block, HTTP
webhook block, etc.).

This requires reimplementing functionality that Redpanda Connect already
provides with 200+ battle-tested connectors. Each custom connector needs its
own retry logic, error handling, pagination, authentication, and format
parsing. The engineering effort is enormous, and the result would be less
mature and less well-tested than RC.

**Rejected**: Massive engineering waste reinventing solved infrastructure.

### Redpanda Connect Handles Everything (No Block Runtime)

Use RC's processor pipeline for billing operations. Implement rate plan
evaluation, balance management, and invoice generation as RC processors
configured in YAML.

RC processors are stateless, YAML-configured transforms designed for data
routing and format conversion. They have no concept of durable state stores,
financial precision types, tenant isolation, or complex business logic like
tiered pricing evaluation. Attempting to implement billing logic in RC
processors would require fighting the framework at every step.

**Rejected**: RC processors cannot evaluate rate plans, manage balances, or
generate invoices. Wrong tool for the job.

### Custom Connector Framework

Build a custom connector framework from scratch that handles both data
collection and billing processing in a unified runtime. This is essentially
option 1 (no RC) with the added ambition of creating a general-purpose
connector framework.

This suffers from the same problems as option 1 — reimplementing solved
infrastructure — plus the additional burden of designing and maintaining a
general-purpose connector API. This is a classic case of Not-Invented-Here
syndrome.

**Rejected**: NIH syndrome; no advantage over using RC for its strengths and
the block runtime for its strengths.

## References

- [Redpanda Connect Documentation](https://docs.redpanda.com/redpanda-connect/) — Official documentation for Redpanda Connect (formerly Benthos)
- [Benthos Architecture](https://www.benthos.dev/docs/about) — Original Benthos design philosophy and architecture
- [CloudEvents Specification](https://cloudevents.io/) — CNCF specification for event format interoperability
- [ADR-0004: CloudEvents Envelope](0004-cloudevents-event-format.md)
- [ADR-0011: At-Least-Once Delivery with Idempotency Keys](0011-at-least-once-idempotency.md)
- [ADR-0012: Tenant-Isolated Event Batches](0012-tenant-isolated-batches.md)
- [METR-0002: Block-Based Extensibility](../../enhancements/0002-extensibility/extensibility.md)
