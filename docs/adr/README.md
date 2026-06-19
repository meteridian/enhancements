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

### Extensibility (see extensibility PR)

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0005](0005-arrow-unified-serialization.md) | Apache Arrow as unified serialization | Accepted |
| [ADR-0006](0006-hybrid-plugin-runtime.md) | Hybrid plugin runtime (Go + WASM + gRPC) | Accepted |
| [ADR-0007](0007-block-dataflow-model.md) | Block-based dataflow extensibility | Accepted |
| [ADR-0008](0008-ai-first-extensibility.md) | AI-first interface (MCP + A2A) | Accepted |
| [ADR-0009](0009-slsa-sigstore-provenance.md) | SLSA/Sigstore for marketplace provenance | Accepted |
| [ADR-0010](0010-pluggable-payment-providers.md) | Pluggable payment providers | Accepted |

## Creating New ADRs

1. Copy the template from any existing ADR
2. Number sequentially (next: ADR-0011)
3. Use kebab-case filenames: `NNNN-short-title.md`
4. Set status to `Proposed` until reviewed, then `Accepted`
5. Never delete or renumber ADRs; supersede with a new one
