# ADR-0015: Compliance Policy Engine Architecture

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq
- **Related:** METR-0008 (Compliance-as-Code), METR-0002 (Extensibility)

## Context

METR-0008 defines a compliance policy engine that evaluates regulatory
requirements expressed as code against Meteridian's live system state. The
engine must support three evaluation modes:

1. **Inline (synchronous):** Evaluate before committing a billing event or
   configuration change; reject if non-compliant; <5ms latency budget
2. **Continuous (async):** Periodic background sweep of system state; generate
   alerts for policy violations
3. **On-demand:** Auditor requests a compliance check; generate a report

Three approaches were evaluated:

1. **Embedded OPA (Open Policy Agent)** — embed the OPA engine as a Go library
   (`github.com/open-policy-agent/opa/rego`) within Meteridian's process
2. **External OPA server** — run OPA as a sidecar or separate service,
   communicate via REST API
3. **Custom DSL** — design a Meteridian-specific policy language

## Decision

**Embed OPA as a Go library (Option 1) for inline and continuous evaluation.
Support external OPA server (Option 2) as an optional deployment mode for
operators who want to share OPA infrastructure across services.**

The primary deployment mode is embedded OPA:

```go
import "github.com/open-policy-agent/opa/rego"

func (e *PolicyEngine) Evaluate(ctx context.Context, input interface{}) (*PolicyResult, error) {
    query, err := rego.New(
        rego.Query("data.meteridian.compliance"),
        rego.Module("policy.rego", e.compiledPolicy),
    ).PrepareForEval(ctx)
    if err != nil {
        return nil, fmt.Errorf("policy preparation: %w", err)
    }
    results, err := query.Eval(ctx, rego.EvalInput(input))
    // ...
}
```

Policies are loaded as OPA bundles (tarballs of Rego files + data files) from
local storage or the block marketplace.

For operators running OPA as a shared infrastructure service (common in
Kubernetes environments), Meteridian also supports delegating policy evaluation
to an external OPA server via REST:

```
POST http://opa:8181/v1/data/meteridian/compliance
Content-Type: application/json

{"input": { ... }}
```

The evaluation mode is configured per policy category: latency-sensitive
policies (data residency, billing integrity) use embedded OPA; latency-tolerant
policies (configuration compliance, operational checks) can use either mode.

## Rationale

### Why Not External OPA Only (Option 2)?

External OPA adds ~1-5ms of network latency per evaluation. For inline
evaluation on the billing event path (target: <5ms total), this consumes the
entire latency budget. Embedded OPA evaluates in <1ms for typical policies
(measured: 100-500μs for policies with <50 rules).

### Why Not Custom DSL (Option 3)?

A custom DSL offers maximum control over syntax and performance but:

- **No existing ecosystem:** OPA/Rego has a large community, extensive
  documentation, and production use at scale (Netflix, Goldman Sachs, Atlassian,
  Pinterest, and others)
- **No existing tooling:** Rego has IDE support, testing framework (`opa test`),
  profiling, and debugging tools
- **Engineering cost:** Designing, implementing, and maintaining a policy
  language is a multi-year effort that diverts resources from core billing
  functionality
- **Adoption barrier:** Operators would need to learn a Meteridian-specific
  language; Rego skills are transferable across the cloud-native ecosystem

### Why OPA Bundles for Distribution?

OPA bundles are the standard OPA distribution mechanism. They are:

- Versioned tarballs containing Rego files and data files
- Loadable from local filesystem, HTTP servers, or S3-compatible storage
- Supported by OPA's built-in bundle management (signature verification,
  polling, caching)
- Compatible with Meteridian's marketplace (blocks are also versioned tarballs
  with provenance verification via SLSA/Sigstore — ADR-0009)

## Consequences

### Positive

- **Sub-millisecond inline evaluation:** Embedded OPA meets the <5ms latency
  budget with room to spare
- **Ecosystem leverage:** Operators familiar with OPA can reuse existing
  policies and tooling
- **Testability:** Policies are unit-testable with `opa test` before deployment
- **No external dependency for core path:** Embedded mode requires no network
  calls on the billing event path
- **Marketplace integration:** OPA bundles fit naturally into the block
  marketplace distribution model

### Negative

- **OPA dependency:** Meteridian takes a dependency on the OPA Go library.
  OPA is Apache 2.0 licensed and CNCF graduated — low risk of licensing or
  governance issues.
- **Rego learning curve:** Rego's logic programming paradigm (Datalog-based)
  is unfamiliar to developers accustomed to imperative languages. Mitigation:
  ship well-documented example policies and a policy development guide.
- **Memory overhead:** Embedded OPA consumes memory proportional to the number
  of loaded policies and data. For typical compliance rule sets (< 1000 rules),
  this is <100MB. For very large rule sets, the external OPA server mode allows
  independent scaling.
- **Policy hot-reload complexity:** Swapping compiled policies in an embedded
  engine without restart requires careful synchronization. The OPA Go library
  supports this via `rego.PrepareForEval` with re-preparation on bundle update.

## Implementation Notes

- Policy bundles are stored in the same artifact registry as blocks
  (S3-compatible storage)
- Bundle signature verification uses the same Sigstore/SLSA pipeline as
  marketplace blocks (ADR-0009)
- The policy engine exposes Prometheus metrics: evaluations/sec, latency
  histogram, violations/sec, bundle reload events
- Policy evaluation results are recorded in the audit trail (METR-0008 §5.1)
  for compliance reporting
- The `/api/v1/compliance/policies/{id}/evaluate` endpoint wraps the on-demand
  evaluation mode and returns structured results
