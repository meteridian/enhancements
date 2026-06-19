# ADR-0018: Closed-Loop Enforcement via Limitador

- **Status:** Proposed
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq, @jordigilh
- **Related:** [METR-0011](../../enhancements/0011-enforcement-integration/enforcement-integration.md), [METR-0010](../../enhancements/0010-ai-metering/ai-metering.md), [ADR-0002](0002-valkey-balance-management.md)

---

## Context and Problem Statement

Meteridian meters AI workload usage and manages billing state (credit
balances, budget thresholds). However, it does **not** sit in the inference
request path and cannot directly accept or reject AI inference requests. The
AI Grid PoC prospect requires real-time enforcement — hard caps on token and
compute budgets with automatic suspension when budgets are exhausted (gap
analysis MB-002 and MB-006).

How should Meteridian close the loop between metering (what it does) and
enforcement (what the AI platform does)?

## Decision Drivers

- Meteridian must NOT become a gateway or proxy (separation of concerns)
- Enforcement must be fail-open (never block inference due to billing system
  outage)
- Enforcement latency must be < 5 seconds from balance change to denial
- The solution must work with the existing RHOAI MaaS infrastructure (Red Hat
  Connectivity Link, Limitador, MaaS external metering plugin)
- The solution must be observable and auditable

## Considered Options

### Option 1: Meteridian as Balance-Check Backend (Pull Model)

The MaaS external metering plugin already queries a metering backend before
allowing inference. Meteridian serves as this backend, responding with allow
or deny based on current balance.

- **Pro:** Zero additional infrastructure; uses existing plugin mechanism
- **Pro:** Per-request accuracy (every request is checked)
- **Pro:** Simple REST API implementation
- **Con:** Adds ~3-8ms latency per inference request
- **Con:** Only enforces when a request arrives (no proactive blocking)
- **Con:** Meteridian availability directly impacts enforcement

### Option 2: Push Enforcement Signals to Limitador (Push Model)

When balance crosses a threshold, Meteridian proactively pushes updated
rate-limit configuration to Limitador via its HTTP API.

- **Pro:** No per-request overhead on Meteridian (Limitador evaluates locally)
- **Pro:** Proactive enforcement (takes effect before next request)
- **Pro:** Persistent enforcement survives Meteridian restarts
- **Con:** ~1-5 second enforcement latency (async)
- **Con:** Dependency on Limitador API stability
- **Con:** More complex implementation (counter management, TTLs)

### Option 3: Kubernetes Operator Patching CRDs

A Kubernetes operator watches Meteridian's balance state and patches
`RateLimitPolicy` or `MaaSSubscription` CRDs.

- **Pro:** Kubernetes-native, declarative
- **Pro:** Persistent configuration changes
- **Con:** Slowest enforcement latency (~10-30 seconds for CRD reconciliation)
- **Con:** Requires a separate operator deployment
- **Con:** Complex failure modes (operator crash, CRD conflicts)

### Option 4: Webhook to Orchestrator (Fire-and-Forget)

Meteridian fires enforcement webhooks to the orchestrator, which is
responsible for implementing enforcement.

- **Pro:** Loose coupling (webhook is a simple HTTP POST)
- **Con:** Requires the orchestrator to implement an enforcement listener
- **Con:** No guarantee of enforcement (webhook may be dropped)
- **Con:** No standard contract (every orchestrator has different APIs)

## Decision

**Adopt a dual-path defense-in-depth model combining Option 1 (Balance Check)
and Option 2 (Limitador Push).**

Both paths are implemented:
1. **Path A (Pull)**: Meteridian serves as the MaaS plugin's balance-check
   backend. Every inference request is checked against current balance.
2. **Path B (Push)**: On threshold crossings, Meteridian pushes enforcement
   signals to Limitador's HTTP API to set or clear rate limits.

The dual approach ensures enforcement survives any single component failure:
- If Meteridian is down: Path B (Limitador) still enforces previously pushed
  limits.
- If Limitador is down: Path A (balance check) still denies over-budget
  requests.
- If both are down: Fail-open; usage is metered and reconciled later.

Option 3 (Kubernetes operator) is deferred to post-PoC for persistent
enforcement use cases and non-Limitador environments.

Option 4 (webhooks) is available as a fallback for non-RHOAI deployments but
is not the primary enforcement mechanism.

## Consequences

### Positive

- **No single point of failure** for enforcement
- **Sub-second enforcement** via Path A (immediate on next request)
- **Proactive enforcement** via Path B (doesn't wait for next request)
- **Fail-open guarantee** — billing system outages never block inference
- **Leverages existing infrastructure** (MaaS plugin balance check, Limitador)
- **Observable** — every enforcement action emitted as a CloudEvent

### Negative

- **Two integration surfaces** to maintain (balance-check API and Limitador
  client)
- **Eventual consistency** — brief window (~1-5s) where over-budget requests
  may proceed before Path B takes effect
- **Dependency on Limitador API** — API changes require Meteridian updates
- **Additional operational complexity** — two enforcement paths to monitor and
  troubleshoot

### Neutral

- Enforcement accuracy depends on event delivery latency from the MaaS
  gateway (not on Meteridian's processing speed)
- The balance-check endpoint may need horizontal scaling for high-throughput
  tenants (thousands of requests per second)

## Implementation Notes

- Path A is implemented as a REST endpoint in the Meteridian API service
- Path B is implemented as a `limitador-enforcement-sink` block in the
  Meteridian pipeline (METR-0002 block model)
- Both paths use the same Valkey-backed balance store as the source of truth
- See [METR-0011](../../enhancements/0011-enforcement-integration/enforcement-integration.md)
  for the full implementation specification
