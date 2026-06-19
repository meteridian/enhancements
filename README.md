# Meteridian Enhancements

This repository contains design documents and enhancement proposals for
[Meteridian](https://github.com/meteridian), an open-source billing-grade
metering and rating platform for hybrid infrastructure.

## What Is Meteridian?

Meteridian is a **metering and rating kernel with a pluggable block
architecture**. It covers the full spectrum of infrastructure: Kubernetes, bare
metal, hypervisors, cloud providers, network devices, mainframes, OpenStack, and
AI/ML workloads. It provides financial-grade rating accuracy with real-time
balance management, and deploys on-premises or in the cloud.

Users can swap, extend, or replace almost any pipeline component by plugging in
their own blocks. The platform provides the financial-grade plumbing (data
plane, serialization, credit ledger, product catalog, rating engine, trust
framework) and lets users assemble the pipeline that fits their business. See
[METR-0002 (Extensibility)](enhancements/0002-extensibility/extensibility.md)
for the canonical statement of this philosophy.

**Examples of "bring your own block":**

- **Invoicing:** Use Meteridian's native invoicing or plug in Zuora, Lago, etc. as a Sink block
- **Cost allocation:** Use standard formulas or write a custom Transform block with arbitrary math
- **AI metering:** Configure a Source block for MaaS CloudEvents or any other AI platform
- **Payment processing:** Plug in Stripe, Paddle, Adyen, or your own payment provider as a block
- **ERP integration:** Write a Sink block for SAP, Oracle, or NetSuite journal entry export
- **Subscription management:** Handle natively or connect external billing tools as Sink blocks

### Target Market

| Segment | Use Case |
|---------|----------|
| **Internal FinOps** | Chargeback and showback across hybrid infrastructure |
| **Managed Service Providers (MSPs)** | Billing customers for hosted services |
| **Neoclouds and Sovereign Clouds** | BMaaS, VMaaS, DBaaS, GPUaaS, MaaS, and similar XaaS |
| **B2B and B2C subscription management** | Native or via integrated external tools (Zuora, Lago, Stripe Billing) |

Meteridian is not targeting telco BSS (no 3GPP, no CDR mediation, no Diameter)
— though architecturally capable of slice-based metering as a dimension.

## Repository Structure

Each enhancement lives in its own numbered directory under `enhancements/`:

```
enhancements/
  0001-architecture/     # Foundational architecture design
  0002-feature-name/     # Future enhancements
  ...
```

Each enhancement directory contains a `README.md` with the full proposal.

## Enhancement Lifecycle

| Status | Meaning |
|--------|---------|
| `provisional` | Under active discussion, not yet accepted |
| `implementable` | Accepted and ready for implementation |
| `implemented` | Fully implemented in the codebase |
| `deferred` | Postponed to a future release |
| `rejected` | Not accepted |
| `withdrawn` | Withdrawn by the author |
| `replaced` | Superseded by another enhancement |

## Contributing

1. Fork this repository
2. Create a new directory: `enhancements/NNNN-short-title/`
3. Write your proposal in `README.md` following the template
4. Open a pull request for discussion

## License

Apache License 2.0
