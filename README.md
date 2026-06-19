# Meteridian Enhancements

This repository contains design documents and enhancement proposals for
[Meteridian](https://github.com/meteridian), an open-source billing-grade
metering and rating platform for hybrid infrastructure.

## What Is Meteridian?

Meteridian is a metering, rating, and cost management platform that covers the
full spectrum of infrastructure: Kubernetes, bare metal, hypervisors, cloud
providers, network devices, mainframes, OpenStack, and AI and ML workloads. It
provides telco-grade rating accuracy with real-time balance management, and
deploys on-premises or in the cloud.

## Repository Structure

```
enhancements/                          # Enhancement proposals (METRs)
  0000-requirements/requirements.md    # METR-0000: Product requirements (separate branch)
  0001-architecture/architecture.md    # METR-0001: Platform architecture (separate branch)
  0002-extensibility/extensibility.md  # METR-0002: Extensibility and marketplace
  0003-product-catalog/                # METR-0003: Product and service catalog
  0004-credit-token-billing/           # METR-0004: Credit, prepaid, and token billing
  0005-internal-token-economy/         # METR-0005: Internal Budget Units and Chargeback
  0006-developer-experience/           # METR-0006: Developer experience and tooling
  0007-legal-regulatory-compliance/    # METR-0007: Legal and regulatory compliance
  0008-compliance-as-code/             # METR-0008: Compliance-as-code and audit trails
  0009-e-invoicing-engine/             # METR-0009: Native e-invoicing engine
  0010-ai-metering/                    # METR-0010: AI workload metering
  0011-enforcement-integration/        # METR-0011: Enforcement integration (Limitador)
docs/
  adr/                                 # Architecture Decision Records (ADRs)
  competitive/                         # Competitive research and market analysis
ROADMAP.md                             # Phases, priorities, deferred items
```

## Enhancement Lifecycle

| Status | Meaning |
|--------|---------|
| `draft` | Initial draft, not yet ready for formal discussion |
| `provisional` | Under active discussion, not yet accepted |
| `implementable` | Accepted and ready for implementation |
| `implemented` | Fully implemented in the codebase |
| `deferred` | Postponed to a future release |
| `rejected` | Not accepted |
| `withdrawn` | Withdrawn by the author |
| `replaced` | Superseded by another enhancement |

## Documents

| Type | What | Where |
|------|------|-------|
| **METR** | Enhancement proposals | `enhancements/NNNN-short-title/` |
| **ADR** | Architectural decisions | [`docs/adr/`](docs/adr/index.md) |
| **ROADMAP** | Phases and deferred items | [`ROADMAP.md`](ROADMAP.md) |

## Terminology

The following terms have specific, non-overlapping meanings in Meteridian:

| Term | Meaning | Where defined |
|------|---------|---------------|
| **Credits** | Platform-level prepaid balance (e.g., "100 credits per core-hour"). Used for committed spend, prepaid packages, drawdown. Industry standard (cf. Snowflake credits, Google Cloud credits). | METR-0004 |
| **Budget units** | Internal allocation units for chargeback and showback within an organization. Departments "spend" budget units; finance "allocates" them. | METR-0005 |
| **Tokens** | Reserved for blockchain and DePIN settlement (v2). Burn-and-Mint Equilibrium, on-chain settlement, stablecoin integration. | ROADMAP (deferred) |

Avoid using "tokens" when you mean "credits" or "budget units." This prevents
confusion between internal billing concepts and blockchain tokenomics.

## Contributing

1. Fork this repository
2. Create a new directory: `enhancements/NNNN-short-title/`
3. Write your proposal as `short-title.md` (not `README.md` — avoids filename conflicts across branches)
4. Open a pull request for discussion

## License

Apache License 2.0
