# Meteridian Roadmap

This document tracks planned work, priorities, and deferred decisions for the
Meteridian platform. It complements the enhancement proposals (METRs) and
Architecture Decision Records (ADRs) by providing a high-level view of what
comes when.

## How We Track Work

| Artifact | Purpose | When to Use |
|----------|---------|-------------|
| **METR** (Enhancement) | Detailed design for a significant feature | Well-understood scope, ready for implementation discussion |
| **ADR** | Record of a specific architectural decision | Decision made (or proposed) with alternatives evaluated |
| **ROADMAP.md** (this file) | High-level phases, priorities, deferred items | Tracking what's next, what's deferred, and why |
| **GitHub Issues** | Implementation tasks, bugs, small improvements | Actionable work items for a specific sprint |
| **Draft METR** (status: draft) | Placeholder for future enhancements | Known need, not yet designed in detail |

Rule of thumb: if it needs a design discussion, it's a METR. If it's a
decision with alternatives, it's an ADR. If it's a tracking item or a deferred
decision, it goes here.

---

## Phase 1: Foundation (v1.0)

Target: First usable release with core metering, rating, and billing.

### Enhancements

| METR | Title | Status |
|------|-------|--------|
| [METR-0000](enhancements/0000-requirements/requirements.md) | Requirements | provisional |
| [METR-0001](enhancements/0001-architecture/architecture.md) | Platform Architecture | provisional |
| [METR-0002](enhancements/0002-extensibility/extensibility.md) | Platform Extensibility and Block Marketplace | provisional |
| [METR-0003](enhancements/0003-product-catalog/product-catalog.md) | Product and Service Catalog | provisional |
| [METR-0004](enhancements/0004-credit-token-billing/credit-token-billing.md) | Credit, Prepaid, and Token-Based Billing | provisional |
| [METR-0005](enhancements/0005-internal-token-economy/internal-token-economy.md) | Internal Budget Units and Chargeback | provisional |
| [METR-0006](enhancements/0006-developer-experience/developer-experience.md) | Developer Experience and Tooling | provisional |
| [METR-0007](enhancements/0007-legal-regulatory-compliance/legal-regulatory-compliance.md) | Legal and Regulatory Compliance Framework | draft |
| [METR-0008](enhancements/0008-compliance-as-code/compliance-as-code.md) | Compliance-as-Code and Audit Trail Guarantees | draft |
| [METR-0009](enhancements/0009-e-invoicing-engine/e-invoicing-engine.md) | Native E-Invoicing Engine | draft |
| [METR-0010](enhancements/0010-ai-metering/ai-metering.md) | AI Workload Metering | draft |
| [METR-0011](enhancements/0011-enforcement-integration/enforcement-integration.md) | Enforcement Integration (Closed-Loop via Limitador) | draft |

### Key Decisions (ADRs)

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0001](docs/adr/0001-timescaledb-event-store.md) | TimescaleDB as primary event store | Accepted |
| [ADR-0002](docs/adr/0002-valkey-balance-management.md) | Valkey for real-time balance management | Accepted |
| [ADR-0003](docs/adr/0003-gorules-rating-engine.md) | GoRules ZEN Engine for rating rules | Accepted |
| [ADR-0004](docs/adr/0004-cloudevents-event-format.md) | CloudEvents as canonical event format | Accepted |
| [ADR-0005](docs/adr/0005-arrow-flight-block-data-plane.md) | Arrow Flight as block data plane | Accepted |
| [ADR-0006](docs/adr/0006-hybrid-plugin-runtime.md) | Go SDK + gRPC now, WASM later | Accepted |
| [ADR-0007](docs/adr/0007-block-based-dataflow.md) | Block-based dataflow extensibility | Accepted |
| [ADR-0008](docs/adr/0008-ai-first-extensibility.md) | AI-first interface (MCP + A2A) | Accepted |
| [ADR-0009](docs/adr/0009-slsa-sigstore-provenance.md) | SLSA/Sigstore for marketplace provenance | Accepted |
| [ADR-0010](docs/adr/0010-pluggable-payment-providers.md) | Pluggable payment providers | Accepted |
| [ADR-0011](docs/adr/0011-at-least-once-idempotency.md) | At-least-once + idempotency keys | Accepted |
| [ADR-0012](docs/adr/0012-tenant-isolated-batches.md) | Tenant-isolated event batches | Accepted |
| [ADR-0013](docs/adr/0013-two-layer-data-architecture.md) | Two-layer data architecture (Redpanda Connect + block runtime) | Accepted |
| [ADR-0014](docs/adr/0014-cryptographic-audit-trail.md) | Cryptographic audit trail (hash chains + external anchoring) | Accepted |
| [ADR-0015](docs/adr/0015-compliance-policy-engine.md) | Compliance policy engine (embedded OPA/Rego) | Accepted |
| [ADR-0016](docs/adr/0016-einvoice-canonical-model.md) | E-invoice canonical model (EN 16931) | Accepted |
| [ADR-0017](docs/adr/0017-fedramp-boundary-architecture.md) | FedRAMP boundary architecture and sovereign cloud deployment | Accepted |
| [ADR-0018](docs/adr/0018-closed-loop-enforcement-limitador.md) | Closed-loop enforcement via Limitador (dual-path) | Accepted |

### v1.0 Scope

- Go SDK and gRPC/Arrow Flight block runtimes (no WASM)
- Product catalog with plans, charges, tiers, and multi-currency price books
- Credit grants, prepaid balances, committed spend contracts
- Internal budget unit economy for chargeback and showback
- Redpanda Connect for data collection (layer 1)
- Block runtime for billing processing (layer 2)
- MCP server and A2A endpoint for AI-first extensibility
- CLI and local emulator for block development
- Marketplace with SLSA/Sigstore provenance (Tier 0-2 blocks via gRPC)
- OCP, AWS, Azure, GCP provider support
- AI workload metering: LLM token dimensions, GPU compute metrics, tiered token pricing (METR-0010)
- Closed-loop enforcement integration: Balance Check API and Limitador quota push (METR-0011, ADR-0018)
- Immutable audit trail with cryptographic hash chains (METR-0008, ADR-0014)
- Sequential invoice numbering with gap detection
- PII separation and selective erasure (GDPR-compatible data model)
- Canonical e-invoice model based on EN 16931 (METR-0009, ADR-0016)
- UBL 2.1 and Factur-X format generators
- Avalara tax engine integration block
- Built-in multi-rate VAT/GST calculator for air-gapped deployment
- SOC 2-aligned logging and evidence collection

### v1.0 Explicitly Excluded

- WASM runtime (deferred to v2, see ADR-0006)
- DePIN and blockchain tokenomics (deferred to v2, see below)
- Visual pipeline editor (deferred, AI-first approach covers this)
- Bin-packing optimization for same-tenant batches (deferred, see ADR-0012)

---

## Phase 2: Scale and Ecosystem (v2.0)

Target: Marketplace maturity, advanced features, broader ecosystem.

### Planned Enhancements

| ID | Title | Status | Notes |
|----|-------|--------|-------|
| METR-TBD | WASM Block Runtime | planned | Adds Extism-based WASM runtime for higher block density (ADR-0006) |
| METR-TBD | DePIN and Blockchain Tokenomics | planned | Dimension 3: Burn-and-Mint Equilibrium, on-chain settlement, stablecoin payments. See "Deferred: DePIN" below. |
| METR-TBD | Visual Pipeline Editor | planned | Node-RED-inspired browser UI for pipeline composition |
| METR-TBD | Multi-Region Federation | planned | Catalog sync, event routing, data residency enforcement |
| METR-TBD | Advanced Fraud Detection | planned | ML-based anomaly detection blocks for revenue assurance |

### v1.x Compliance and E-Invoicing Expansion (from METR-0008 and METR-0009)

| Capability | Source | Notes |
|-----------|--------|-------|
| OPA-based compliance policy engine | METR-0008 / ADR-0015 | Inline and continuous evaluation of regulatory policies |
| GDPR, SOX, GoBD regulatory rule sets | METR-0008 | Pre-built policy bundles for major jurisdictions |
| Data subject rights API (access, erasure, portability) | METR-0008 | GDPR Art. 15-22 implementation |
| SOC 2 evidence collector and ISO 27001 artifact generator | METR-0008 | Automated compliance reporting |
| GoBD and FEC audit export blocks | METR-0008 | German and French audit export formats |
| XML-DSig, XAdES, ZATCA CSID digital signing | METR-0009 | Jurisdiction-specific cryptographic signing |
| SEFAZ, IRP, ZATCA tax authority connectors | METR-0009 | Brazil, India, Saudi Arabia clearance |
| NF-e, CFDI 4.0, India GST format generators | METR-0009 | Jurisdiction-specific e-invoice formats |
| Peppol Access Point integration | METR-0009 | EU, AU, and SG e-invoice delivery |
| HSM and KMS integration (PKCS#11, cloud KMS) | METR-0009 | Production key management |
| FedRAMP continuous monitoring artifacts | ADR-0017 | NIST 800-53 control mapping |

### Planned METRs and ADRs

| ID | Title | Type | Notes |
|----|-------|------|-------|
| METR-TBD | Multi-Tenancy and Authorization | METR | Tenant isolation model, OpenFGA/Zanzibar vs RBAC, API-level access control. Prerequisite for production deployment. |
| ADR-TBD | Authorization Model | ADR | Choice of authorization engine (OpenFGA, SpiceDB, or custom), tenant isolation guarantees, integration with block security model. |
| ADR-TBD | Backup and Disaster Recovery Strategy | ADR | RPO and RTO targets, TimescaleDB continuous archiving, Valkey persistence, cross-region replication strategy. |
| ADR-TBD | Arrow IPC over WASM shared memory | ADR | Validate zero-copy feasibility before committing to WASM runtime |
| ADR-TBD | WASI Preview 2 readiness assessment | ADR | Gate WASM launch on WASI stability |
| ADR-TBD | Multi-region catalog consistency | ADR | Eventual vs. strong consistency trade-offs |
| ADR-TBD | Performance benchmark baselines | ADR | Establish measured (not estimated) latency numbers for each runtime |

---

## Deferred: DePIN and Blockchain Tokenomics (Dimension 3)

This is the most speculative area of the roadmap. We are honest about what we
know and what we don't.

### What It Is

DePIN (Decentralized Physical Infrastructure Networks) uses blockchain tokens
to incentivize infrastructure providers. Examples: Helium (wireless), Filecoin
(storage), Render Network (GPU), Akash (compute). The tokenomics model
typically involves:

- **Burn-and-Mint Equilibrium (BME)**: Service consumers burn tokens to pay for
  services. The protocol mints new tokens to reward infrastructure providers.
  The burn rate and mint rate converge toward an equilibrium price.
- **Staking**: Infrastructure providers stake tokens as collateral for quality
  of service guarantees.
- **On-chain settlement**: Usage is metered off-chain, but settlement occurs
  on-chain via smart contracts.
- **Stablecoin integration**: Fiat-pegged tokens (USDC, USDT) for price
  stability in billing contexts.

### Why We Defer

1. **Regulatory uncertainty**: Token classification (utility vs. security) varies
   by jurisdiction. We don't want to build features that may require redesign
   when regulations clarify.
2. **Technical complexity**: On-chain settlement requires blockchain node
   infrastructure, smart contract development, and bridge integration — each of
   these is a major project in itself.
3. **Market validation**: The DePIN market is nascent. We need to validate
   demand with actual customers before investing.
4. **Dimension 1 and 2 first**: Credit and prepaid billing and internal budget unit
   economy serve much larger markets today and share foundational
   infrastructure (balance management, ledger, credit grants) with DePIN.

### What We Build Now (in v1) That Enables DePIN Later

- Pluggable payment providers (ADR-0010) — can plug in crypto payment rails
- Credit and balance management (METR-0004) — same ledger model as BME
- Event-sourced architecture — immutable audit trail needed for on-chain proofs
- Multi-currency price books (METR-0003) — can add token-denominated prices

### When to Revisit

- When we have a concrete customer request for DePIN metering
- When regulatory frameworks (MiCA in EU, US stablecoin legislation) stabilize
- After v1 launch, with real production data on credit and prepaid workloads

---

## Deferred: Performance Benchmark Baselines

The extensibility architecture (METR-0002, ADR-0005, ADR-0006) includes
performance estimates:

| Metric | Estimate | Measured? |
|--------|----------|-----------|
| Go SDK block overhead | ~0 (in-process) | No — need benchmark |
| gRPC/Arrow Flight block overhead | ~1-5ms per call | No — need benchmark |
| WASM block overhead | ~50-200μs per call | No — need benchmark (v2) |
| Arrow RecordBatch batch size sweet spot | 1K-10K rows | No — need benchmark |
| End-to-end pipeline latency (5 blocks) | ~5-25ms (gRPC) | No — need benchmark |

These numbers are based on published benchmarks from Arrow Flight (Dremio),
WASM (Extism), and similar systems. They are NOT measured on Meteridian
workloads.

**Action:** Before v1 GA, establish measured baselines using representative
metering workloads. This will likely become an ADR documenting the actual
performance characteristics and any adjustments to batch size or runtime
recommendations.

---

## Deferred: SDK Expansion

v1 ships with Go SDK and Python SDK. Additional SDKs are planned:

| Language | Priority | Rationale |
|----------|----------|-----------|
| Rust | High | Systems programming, WASM compilation target (v2) |
| TypeScript and Node.js | Medium | Web developer audience, lightweight blocks |
| Java | Medium | Enterprise ecosystem, Kafka integration |
| C# | Low | .NET ecosystem |

SDK development follows the Block Interface Specification (defined in
METR-0006). New SDKs are generated from this specification, not hand-written.

---

## Contribution Guide

To add items to this roadmap:

1. **New enhancement:** Create a METR document in `enhancements/NNNN-short-name/`
2. **New decision:** Create an ADR in `docs/adr/NNNN-short-name.md`
3. **Deferred item:** Add a section to this file under the appropriate phase
4. **Implementation task:** Create a GitHub Issue linking to the relevant METR or ADR

Keep this file updated as METRs and ADRs are created, accepted, or superseded.
