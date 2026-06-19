# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the Meteridian platform.

ADRs document significant architectural decisions, including the context, decision, consequences, and alternatives considered. They follow the [MADR](https://adr.github.io/madr/) format.

## Index

### Platform Architecture

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0001](0001-timescaledb-event-store.md) | TimescaleDB as primary event store | Accepted |
| [ADR-0002](0002-valkey-balance-management.md) | Valkey for real-time balance management | Accepted |
| [ADR-0003](0003-gorules-rating-engine.md) | GoRules ZEN Engine for rating rules | Accepted |
| [ADR-0004](0004-cloudevents-event-format.md) | CloudEvents as canonical event format | Accepted |

### Extensibility

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0005](0005-arrow-flight-block-data-plane.md) | Arrow Flight as block data plane | Accepted |
| [ADR-0006](0006-hybrid-plugin-runtime.md) | Go SDK + gRPC now, WASM later | Accepted |
| [ADR-0007](0007-block-based-dataflow.md) | Block-based dataflow extensibility | Accepted |
| [ADR-0008](0008-ai-first-extensibility.md) | AI-first interface (MCP + A2A) | Accepted |
| [ADR-0009](0009-slsa-sigstore-provenance.md) | SLSA/Sigstore for marketplace provenance | Accepted |
| [ADR-0010](0010-pluggable-payment-providers.md) | Pluggable payment providers | Accepted |

### Data Processing

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0011](0011-at-least-once-idempotency.md) | At-least-once delivery + idempotency keys | Accepted |
| [ADR-0012](0012-tenant-isolated-batches.md) | Tenant-isolated event batches | Accepted |
| [ADR-0013](0013-two-layer-data-architecture.md) | Two-layer data architecture (Redpanda Connect + block runtime) | Accepted |

### Compliance and E-Invoicing

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0014](0014-cryptographic-audit-trail.md) | Cryptographic audit trail (hash chains + external anchoring) | Accepted |
| [ADR-0015](0015-compliance-policy-engine.md) | Compliance policy engine (embedded OPA/Rego) | Accepted |
| [ADR-0016](0016-einvoice-canonical-model.md) | E-invoice canonical model (EN 16931) | Accepted |
| [ADR-0017](0017-fedramp-boundary-architecture.md) | FedRAMP boundary architecture and sovereign cloud deployment | Accepted |

## Creating New ADRs

1. Copy the template from any existing ADR
2. Number sequentially (next: ADR-0018)
3. Use kebab-case filenames: `NNNN-short-title.md`
4. Set status to `Proposed` until reviewed, then `Accepted`
5. Never delete or renumber ADRs; supersede with a new one
