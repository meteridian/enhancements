# Meteridian Enhancements

This repository contains design documents and enhancement proposals for
[Meteridian](https://github.com/meteridian), an open-source billing-grade
metering and rating platform for hybrid infrastructure.

## What Is Meteridian?

Meteridian is a metering, rating, and cost management platform that covers the
full spectrum of infrastructure: Kubernetes, bare metal, hypervisors, cloud
providers, network devices, mainframes, OpenStack, and AI/ML workloads. It
provides telco-grade rating accuracy with real-time balance management, and
deploys on-premises or in the cloud.

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
