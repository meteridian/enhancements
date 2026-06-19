# METR-0011: Closed-Loop Enforcement Integration

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0010 (AI Workload Metering), METR-0004 (Credit and Token Billing)
- **Related:** METR-0002 (Platform Extensibility), ADR-0002 (Valkey Balance Management), ADR-0018 (Closed-Loop Enforcement via Limitador)
- **Motivated by:** [MB-002, MB-006 Gap Analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Enforcement Architecture — Closed-Loop Pattern](#3-enforcement-architecture--closed-loop-pattern)
4. [Integration Path A: Balance Check API (Request-Path)](#4-integration-path-a-balance-check-api-request-path)
5. [Integration Path B: Limitador Quota Push (Async)](#5-integration-path-b-limitador-quota-push-async)
6. [Defense-in-Depth: Dual-Path Enforcement](#6-defense-in-depth-dual-path-enforcement)
7. [Enforcement Actions and Signals](#7-enforcement-actions-and-signals)
8. [Failure Modes and Resilience](#8-failure-modes-and-resilience)
9. [Latency Budget](#9-latency-budget)
10. [Implementation — Enforcement Sink Block](#10-implementation--enforcement-sink-block)
11. [Non-RHOAI Deployments](#11-non-rhoai-deployments)
12. [PoC Scope](#12-poc-scope)
13. [Open Questions](#13-open-questions)
14. [Related Documents](#14-related-documents)

---

## 1. Summary

This enhancement defines the **closed-loop enforcement integration** between
Meteridian (metering and billing) and Red Hat Connectivity Link's Limitador
(rate limiting and enforcement). It addresses gaps MB-002 (Orchestrator
Integration) and MB-006 (Real-Time Guardrails and Enforcement) from the
[AI Grid PoC gap analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md).

The core design principle is: **Meteridian meters, evaluates, and signals;
Limitador enforces.** Meteridian does NOT sit in the inference request path.
Instead, it provides two complementary enforcement integration paths:

1. **Balance Check API (request-path, pull)** — The MaaS external metering
   plugin queries Meteridian's balance API before allowing inference requests.
   Meteridian responds with allow or deny based on the tenant's current
   balance. This leverages the metering plugin's existing `meteringURL`
   configuration.

2. **Limitador Quota Push (async, push)** — When a tenant's balance crosses
   an enforcement threshold, Meteridian proactively pushes updated rate-limit
   configurations to Limitador via its API. This ensures enforcement takes
   effect even between request-path balance checks.

Together, these form a **defense-in-depth** model where enforcement is
guaranteed regardless of timing: the balance check catches requests
immediately after budget exhaustion, while the Limitador push handles
cross-request budget transitions and provides persistent enforcement state.

**Key constraint:** Enforcement must be **fail-open** — if Meteridian or
Limitador is unreachable, inference requests proceed. Billing accuracy is
never sacrificed, but brief over-budget usage is tolerable and reconciled
after the fact.

---

## 2. Motivation

### 2.1 The Enforcement Gap

Meteridian is a billing-grade metering platform. It:
- Ingests usage events (CloudEvents from the MaaS gateway)
- Rates events against the Product Catalog
- Deducts from credit balances in real-time (Valkey, sub-ms)
- Fires threshold alerts (bill shock prevention, METR-0004 §7)

What Meteridian does NOT do (and should not do):
- Sit in the inference request path
- Accept or reject AI inference requests
- Act as a proxy, gateway, or admission controller

The gap is the **feedback loop**: how does Meteridian's billing state (budget
exhausted, threshold crossed) translate into enforcement actions (block
requests, throttle, downgrade model tier)?

### 2.2 Why Limitador Is the Right Enforcement Point

Red Hat Connectivity Link's **Limitador** is already deployed in the MaaS
gateway pipeline as part of the AI Inference Gateway stack. It:

- Operates at the **data plane** (Envoy integration via gRPC Rate Limit
  Service protocol) with sub-millisecond enforcement latency
- Is already configured to enforce `TokenRateLimitPolicy` (per-user
  token-per-minute limits) for RHOAI MaaS
- Provides a **well-defined API** for counter manipulation and limit updates
- Supports multiple storage backends (in-memory, Redis, disk) for counter
  persistence
- Is managed via Kubernetes CRDs (`RateLimitPolicy`) for declarative
  configuration

The MaaS external metering plugin (METR-0010 §3.2) already performs a
**request-phase balance check** against a configurable metering backend. This
means the integration surface already exists — Meteridian simply needs to
serve as that metering backend.

### 2.3 Design Principles

1. **Separation of concerns**: Meteridian owns billing state. Limitador owns
   enforcement execution. Neither duplicates the other's logic.
2. **Fail-open**: Network or service failures must not block inference.
   Over-budget usage is reconciled, not prevented at all costs.
3. **Eventually consistent enforcement**: Enforcement catches up within
   seconds (not microseconds). The latency budget is ~1-5 seconds from
   balance change to enforcement active.
4. **Idempotent signals**: Enforcement updates are idempotent — sending the
   same signal twice has no additional effect.
5. **Observable**: Every enforcement action is logged, metered, and traceable
   in both Meteridian's audit trail and Limitador's telemetry.

---

## 3. Enforcement Architecture — Closed-Loop Pattern

The enforcement integration forms a closed loop:

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                        CLOSED LOOP                                │
  │                                                                   │
  │   ① METER         ② EVALUATE          ③ ENFORCE                  │
  │                                                                   │
  │   CloudEvent ──► Meteridian ──► Enforcement ──► Limitador         │
  │   (token usage)  (balance      Signal          (rate limit        │
  │                   update,      (push quota     update or          │
  │                   threshold     change)         balance check     │
  │                   evaluation)                   response)         │
  │                                                                   │
  │                                       │                           │
  │                                       ▼                           │
  │                               MaaS Gateway                        │
  │                               (Envoy + IPP)                       │
  │                                       │                           │
  │                                       │ ④ RESULT                  │
  │                                       ▼                           │
  │                               Inference Request                   │
  │                               ALLOWED or DENIED                   │
  │                                                                   │
  └──────────────────────────────────────────────────────────────────┘
```

**Step-by-step flow:**

1. **Meter**: The MaaS external metering plugin emits a CloudEvent with token
   usage after each inference response. Meteridian receives and processes it.
2. **Evaluate**: Meteridian deducts tokens from the tenant's credit balance
   (Valkey). If the balance crosses a threshold (75%, 90%, 100%), it triggers
   an enforcement evaluation.
3. **Enforce**: Meteridian pushes an enforcement signal — either updating
   Limitador's counters/quotas directly (Path B) or preparing a denial
   response for the next balance check (Path A).
4. **Result**: The next inference request from this tenant is either allowed
   (balance OK) or denied (budget exhausted) at the gateway level.

---

## 4. Integration Path A: Balance Check API (Request-Path)

### 4.1 How It Works

The MaaS external metering plugin already queries a metering backend before
allowing inference requests (METR-0010 §3.2, request phase). The plugin's
configuration includes a `meteringURL` that points to the balance-check
endpoint. **Meteridian serves as this metering backend.**

```
  Inference Request
       │
       ▼
  ┌─────────────────────────────────┐
  │  MaaS External Metering Plugin  │
  │  (IPP request phase)            │
  │                                 │
  │  1. Extract user identity       │
  │  2. Query balance check ────────┼──► Meteridian Balance API
  │  3. If denied → reject request  │         │
  │  4. If allowed → proceed        │         │
  │                                 │    ┌────┴────┐
  └─────────────────────────────────┘    │ Valkey  │
                                         │ Balance │
                                         │ Store   │
                                         └─────────┘
```

### 4.2 API Contract

Meteridian exposes a balance-check endpoint compatible with the MaaS metering
plugin's expected interface:

```
GET /api/v1/customers/{customer_id}/entitlements/{feature_key}/value
```

**Request parameters:**
- `customer_id`: Tenant identifier (from Authorino identity, `x-maas-username`
  or `x-maas-subscription`)
- `feature_key`: The metered dimension (e.g., `tokens`, `gpu_hours`)

**Response (allow):**

```json
{
  "hasAccess": true,
  "currentUsage": 450000,
  "entitledQuantity": 1000000,
  "remainingQuantity": 550000,
  "overage": false
}
```

**Response (deny — budget exhausted):**

```json
{
  "hasAccess": false,
  "currentUsage": 1000000,
  "entitledQuantity": 1000000,
  "remainingQuantity": 0,
  "overage": true,
  "resetAt": "2026-07-01T00:00:00Z"
}
```

### 4.3 MaaS Plugin Configuration

The MaaS external metering plugin is configured to use Meteridian as its
balance-check backend:

```yaml
upstreamBbr:
  bbr:
    plugins:
      - type: external-metering
        name: external-metering
        json:
          meteringURL: "http://meteridian-api.metering.svc:8080"
          failOpen: true
          timeoutSeconds: 3
          featureKey: "tokens"
          source: "maas-gateway"
```

### 4.4 Characteristics

| Property | Value |
|----------|-------|
| Enforcement latency | ~3-8ms (HTTP round-trip within cluster + Valkey read) |
| Enforcement granularity | Per-request (every inference request is checked) |
| Failure mode | Fail-open (if Meteridian unreachable, request proceeds) |
| State source | Valkey balance (real-time, sub-ms reads) |
| Configuration | MaaS plugin `meteringURL` pointing to Meteridian |

### 4.5 Advantages

- **Zero additional infrastructure**: Uses the metering plugin's existing
  balance-check mechanism — no new components deployed.
- **Per-request accuracy**: Every request is checked against the current
  balance. No enforcement delay.
- **Simple integration**: Meteridian implements a REST endpoint that returns
  a JSON response. No Kubernetes operators, no CRD manipulation.

### 4.6 Limitations

- **Request-path dependency**: Adds ~3-8ms latency to every inference request.
  The `failOpen: true` configuration mitigates availability risk but introduces
  a brief window where over-budget requests may proceed during Meteridian
  outages.
- **Per-request overhead**: Every inference request triggers a balance read.
  For high-throughput workloads (thousands of requests per second per tenant),
  this may require Meteridian to scale its balance-check endpoint.
- **No proactive enforcement**: Only enforces when a request arrives. If a
  long-running request causes budget exhaustion (e.g., a 100K-token streaming
  response), the NEXT request catches the denial, not the current one.

---

## 5. Integration Path B: Limitador Quota Push (Async)

### 5.1 How It Works

When Meteridian detects that a tenant's balance has crossed an enforcement
threshold, it proactively pushes updated rate-limit configuration to Limitador.
This is an async, push-based signal that takes effect for all subsequent
requests without requiring a per-request balance check.

```
  Meteridian                     Limitador                   MaaS Gateway
  ┌───────────────────┐         ┌──────────────────┐       ┌───────────────┐
  │ Balance Monitor   │         │ Rate Limit       │       │ Envoy + RLS   │
  │                   │         │ Service          │       │               │
  │ Detects threshold │         │                  │       │ Next request: │
  │ crossing:         │─ HTTP ─►│ Update counter   │──RLS─►│ DENIED        │
  │ 100% exhausted    │  PATCH  │ or limit config  │       │ (429 or 403)  │
  │                   │         │                  │       │               │
  │ Detects replenish │         │                  │       │ Next request: │
  │ (top-up):         │─ HTTP ─►│ Reset counter    │──RLS─►│ ALLOWED       │
  │ Balance restored  │  PATCH  │ or restore limit │       │               │
  └───────────────────┘         └──────────────────┘       └───────────────┘
```

### 5.2 Limitador API Integration

Limitador exposes a gRPC and HTTP API for rate-limit evaluation and counter
management. Meteridian uses the **HTTP API** for pushing enforcement updates:

**Option 1: Counter manipulation (preferred for PoC)**

Set the tenant's token counter to the maximum limit, causing subsequent
requests to be rate-limited:

```
PATCH /limitador/namespaces/{namespace}/limits/{limit_name}/counters
Content-Type: application/json

{
  "conditions": ["user_id == alice", "feature_key == tokens"],
  "delta": 999999999,
  "max_value": 1000000,
  "seconds": 3600
}
```

This effectively sets the counter to its maximum value, causing Limitador to
deny subsequent requests until the counter resets or is cleared.

**Option 2: Dynamic limit configuration via Kubernetes CRD**

For persistent enforcement (block until manual intervention or top-up), patch
the `RateLimitPolicy` CRD:

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: tenant-alice-enforcement
  namespace: ai-gateway
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: inference-route
  limits:
    "budget-enforcement":
      rates:
        - limit: 0           # Block all requests
          window: 1s
      when:
        - predicate: "request.auth.identity.user == 'alice'"
```

Setting `limit: 0` with a matching predicate blocks all requests from the
specified tenant.

**Option 3: Restore on top-up**

When a tenant replenishes their balance (prepaid top-up, new billing cycle,
admin override), Meteridian reverses the enforcement:

```
DELETE /limitador/namespaces/{namespace}/limits/{limit_name}/counters
Content-Type: application/json

{
  "conditions": ["user_id == alice", "feature_key == tokens"]
}
```

Or for CRD-based enforcement, delete the blocking `RateLimitPolicy`.

### 5.3 Characteristics

| Property | Value |
|----------|-------|
| Enforcement latency | ~1-5 seconds (event processing + HTTP push) |
| Enforcement granularity | Per-threshold-crossing (not per-request) |
| Failure mode | Fail-open (if Limitador unreachable, balance check still active via Path A) |
| State source | Valkey balance + threshold configuration |
| Configuration | Meteridian enforcement sink block config |

### 5.4 Advantages

- **No per-request overhead**: Limitador enforcement is evaluated at the Envoy
  data-plane level with sub-millisecond latency. No HTTP round-trip to
  Meteridian for each request.
- **Proactive**: Takes effect as soon as the threshold is crossed, not waiting
  for the next request.
- **Persistent**: Once the rate limit is applied, it persists across Limitador
  restarts (Redis-backed counters) and does not depend on Meteridian being
  available for subsequent enforcement.
- **Graduated**: Supports multiple thresholds (throttle at 90%, block at 100%)
  by pushing different limit values at each threshold.

### 5.5 Limitations

- **Latency**: ~1-5 seconds from balance change to enforcement active. During
  this window, requests may proceed over-budget.
- **Complexity**: Requires managing Limitador API interactions, counter TTLs,
  and CRD lifecycle.
- **Coupling**: Creates a dependency on Limitador's API stability.

---

## 6. Defense-in-Depth: Dual-Path Enforcement

The two paths are complementary, not alternatives. Deploying both provides
defense-in-depth:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  Inference Request Flow                                          │
  │                                                                  │
  │  Request ──► Envoy ──► Rate Limit Check (Limitador) ──┐         │
  │                                                        │         │
  │              Path B applied?                           │         │
  │              ├── YES (limit=0) → 429 DENIED            │         │
  │              └── NO → continue                         │         │
  │                                                        │         │
  │                         ┌──────────────────────────────┘         │
  │                         ▼                                        │
  │              MaaS Plugin Balance Check (Meteridian) ──┐          │
  │                                                        │         │
  │              Path A response?                          │         │
  │              ├── hasAccess=false → 403 DENIED           │         │
  │              └── hasAccess=true → PROCEED to inference  │         │
  │                                                        │         │
  └─────────────────────────────────────────────────────────────────┘
```

**When each path "wins":**

| Scenario | Path A (Balance Check) | Path B (Limitador Push) |
|----------|----------------------|------------------------|
| Balance just exhausted (same request) | Catches it immediately | Not yet applied (1-5s delay) |
| Balance exhausted, next request arrives 10s later | Catches it | Also catches it (redundant) |
| Meteridian is down | Fail-open (request proceeds) | Still enforcing (Limitador has persisted state) |
| Limitador is down | Still enforcing (plugin denies based on balance) | Not active |
| Both are down | Fail-open (request proceeds, reconciled later) | Not active |

**The dual-path model ensures enforcement survives any single component
failure.** Only a simultaneous failure of both Meteridian AND Limitador allows
over-budget requests, and even then, the usage is accurately metered (the
metering plugin still emits CloudEvents) for after-the-fact reconciliation.

---

## 7. Enforcement Actions and Signals

### 7.1 Threshold-Based Actions

Meteridian evaluates enforcement actions at configurable balance thresholds:

| Balance Level | Action | Path A Behavior | Path B Behavior |
|---------------|--------|----------------|----------------|
| > 25% remaining | None | `hasAccess: true` | No Limitador update |
| 25% remaining | Notify | `hasAccess: true` + warning header | Webhook to billing admin |
| 10% remaining | Throttle | `hasAccess: true` + throttle hint | Push reduced rate limit (e.g., 50% of normal) |
| 0% (exhausted) | Block | `hasAccess: false` | Push rate limit = 0 |
| Replenished (top-up) | Restore | `hasAccess: true` | Delete enforcement limit or reset counter |

### 7.2 Enforcement Signal Schema

Meteridian emits enforcement signals as CloudEvents for observability and
integration with non-Limitador enforcement points:

```json
{
  "specversion": "1.0",
  "id": "enf-a1b2c3d4",
  "type": "billing.enforcement.action",
  "source": "meteridian/enforcement",
  "time": "2026-06-19T14:30:00Z",
  "subject": "tenant-alice",
  "datacontenttype": "application/json",
  "data": {
    "tenant_id": "tenant-alice",
    "action": "block",
    "reason": "budget_exhausted",
    "feature_key": "tokens",
    "current_balance": 0,
    "entitled_quantity": 1000000,
    "threshold_pct": 0,
    "effective_at": "2026-06-19T14:30:00Z",
    "expires_at": "2026-07-01T00:00:00Z",
    "enforcement_target": "limitador",
    "limitador_action": {
      "type": "set_counter_to_max",
      "namespace": "ai-gateway",
      "limit_name": "tenant-alice-tokens"
    }
  }
}
```

### 7.3 Graduated Enforcement (Model Tier Downgrade)

Beyond binary allow and block, enforcement supports graduated responses:

| Signal | Limitador Implementation | Effect |
|--------|------------------------|--------|
| `throttle_50pct` | Reduce `rates[].limit` by 50% | Slower inference, not blocked |
| `downgrade_model_tier` | Apply different limit per model class (block premium models, allow base models) | Cost-conscious fallback |
| `block` | Set `rates[].limit` to 0 | Full denial |
| `burst_penalty` | Apply 2x multiplied rate limit during burst window | Penalize burst, don't block |
| `restore` | Remove enforcement limit or reset counter | Return to normal operation |

---

## 8. Failure Modes and Resilience

### 8.1 Meteridian Unreachable (Path A Failure)

- **MaaS plugin behavior**: `failOpen: true` — request proceeds without
  balance check.
- **Mitigation**: Path B (Limitador) still enforces if a limit was previously
  pushed.
- **Recovery**: When Meteridian recovers, the next balance check resumes
  enforcement.
- **Billing impact**: Usage during the outage is still metered (CloudEvents
  are emitted by the metering plugin regardless of balance check). Overage is
  reconciled on the next invoice.

### 8.2 Limitador Unreachable (Path B Failure)

- **Push behavior**: Meteridian retries with exponential backoff (base 1s,
  max 30s, 3 attempts). After max retries, the enforcement signal is written
  to a dead-letter queue for manual review.
- **Mitigation**: Path A (balance check) still enforces on every request.
- **Recovery**: When Limitador recovers, Meteridian replays any pending
  enforcement signals from the dead-letter queue.

### 8.3 Both Unreachable (Full Failure)

- **Behavior**: Inference requests proceed (fail-open on both paths).
- **Billing impact**: All usage is still metered by the MaaS plugin.
  Overage is billed on the next invoice.
- **SLA expectation**: The platform guarantees metering accuracy (100%) but
  not enforcement SLA (best-effort, target 99.9% of transitions enforced
  within 5 seconds).

### 8.4 Split-Brain Between Paths

If Path A allows but Path B denies (or vice versa), the MORE RESTRICTIVE
path wins. Limitador's deny (429) returns before the metering plugin's
balance check runs, so Path B takes priority when active. If Path B is not
active (no enforcement limit pushed yet), Path A is the sole enforcer.

### 8.5 Idempotency

Enforcement signals are idempotent:
- Pushing the same "block" signal to Limitador multiple times has no
  additional effect (counter already at max).
- Pushing "restore" multiple times is safe (deleting a non-existent counter
  is a no-op).
- The balance-check API is stateless — every call reads current balance.

---

## 9. Latency Budget

### 9.1 End-to-End Enforcement Latency

From the moment a CloudEvent causes budget exhaustion to the moment the next
inference request is denied:

| Step | Latency | Cumulative |
|------|---------|------------|
| CloudEvent received by Meteridian | ~1-5ms (HTTP or Kafka) | 1-5ms |
| Balance deduction (Valkey SET) | ~0.1ms | 1-5ms |
| Threshold evaluation | ~0.1ms | 1-5ms |
| Path A: Next request balance check | ~3-8ms (when request arrives) | Immediate on next request |
| Path B: Push to Limitador API | ~5-20ms (HTTP within cluster) | 5-25ms |
| Path B: Limitador applies new limit | ~0.1ms (in-memory update) | 5-25ms |

**Path A enforcement latency**: 0ms (instantaneous — denial happens on
next request regardless of when it arrives).

**Path B enforcement latency**: ~5-25ms from balance crossing to enforcement
active in Limitador. In practice, this means the very next request after
budget exhaustion may proceed (if it arrives within this window), but all
subsequent requests are denied.

### 9.2 PoC Latency Target

For the AI Grid PoC, the enforcement latency target is:
- **Path A**: < 10ms response time for balance-check API (p99)
- **Path B**: < 5 seconds from threshold crossing to Limitador enforcement
  active (p99)

These targets are achievable with:
- Valkey balance reads: ~0.1ms (in-cluster, same-node)
- Limitador HTTP API: ~5-20ms (in-cluster)
- No external dependencies in the enforcement path

---

## 10. Implementation — Enforcement Sink Block

### 10.1 Block Architecture

The enforcement integration is implemented as a Meteridian **Sink Block**
(METR-0002 §3.2) called `limitador-enforcement-sink`. It subscribes to
balance-change events from the credit system and pushes enforcement signals
to Limitador.

```
  ┌──────────────────────────────────────────────────────────┐
  │  Meteridian Pipeline                                      │
  │                                                           │
  │  ┌────────────┐   ┌──────────┐   ┌────────────────────┐  │
  │  │ Balance    │──►│ Threshold│──►│ Enforcement Sink   │  │
  │  │ Change    │   │ Evaluator│   │ (limitador-        │  │
  │  │ Events    │   │          │   │  enforcement-sink) │  │
  │  └────────────┘   └──────────┘   └─────────┬──────────┘  │
  │                                             │             │
  └─────────────────────────────────────────────┼─────────────┘
                                                │
                                                ▼
                                         ┌──────────────┐
                                         │  Limitador   │
                                         │  HTTP API    │
                                         └──────────────┘
```

### 10.2 Block Manifest

```toml
[block]
name = "limitador-enforcement-sink"
version = "1.0.0"
description = "Pushes enforcement signals to Limitador based on billing state"
author = "meteridian"
license = "Apache-2.0"
runtime = "go"

[input]
schema = [
    { name = "tenant_id", type = "utf8", required = true },
    { name = "action", type = "utf8", required = true },
    { name = "feature_key", type = "utf8", required = true },
    { name = "current_balance", type = "decimal128(15,2)" },
    { name = "entitled_quantity", type = "decimal128(15,2)" },
    { name = "threshold_pct", type = "int32" },
]

[capabilities]
network = ["limitador-service:8080"]
state = { enabled = true, max_size_mb = 10 }

[config]
schema = "config.schema.json"

[health]
endpoint = "/healthz"
interval = "10s"
timeout = "5s"
```

### 10.3 Block Configuration

```yaml
blocks:
  - name: limitador-enforcement
    type: meteridian/limitador-enforcement-sink
    config:
      limitador_url: "http://limitador-service.kuadrant-system:8080"
      timeout_ms: 3000
      retry_max: 3
      retry_backoff_base_ms: 1000
      retry_backoff_max_ms: 30000
      fail_open: true

      thresholds:
        - level: 75
          action: notify
        - level: 90
          action: throttle
          throttle_pct: 50
        - level: 100
          action: block

      enforcement_namespace: "ai-gateway"
      counter_ttl_seconds: 3600
```

### 10.4 Balance Check API Implementation

The balance-check endpoint is a lightweight HTTP handler in the Meteridian
API service (not a separate deployment):

```go
// GET /api/v1/customers/{customer_id}/entitlements/{feature_key}/value
func (h *EnforcementHandler) CheckBalance(w http.ResponseWriter, r *http.Request) {
    customerID := chi.URLParam(r, "customer_id")
    featureKey := chi.URLParam(r, "feature_key")

    balance, err := h.balanceStore.GetBalance(r.Context(), customerID, featureKey)
    if err != nil {
        // Fail-open: if balance store is unreachable, allow the request
        h.writeAllowResponse(w, customerID, featureKey)
        return
    }

    if balance.Remaining <= 0 {
        h.writeDenyResponse(w, balance)
        return
    }

    h.writeAllowResponse(w, customerID, featureKey)
}
```

---

## 11. Non-RHOAI Deployments

For deployments without Red Hat Connectivity Link (no Limitador, no MaaS
gateway), Meteridian provides alternative enforcement mechanisms:

### 11.1 Webhook-Based Enforcement

Meteridian emits enforcement signals as webhook events (CloudEvents format).
The target system implements a listener that acts on these signals:

```yaml
enforcement:
  mode: webhook
  webhook_url: "https://orchestrator.example.com/api/v1/enforcement"
  webhook_headers:
    Authorization: "Bearer ${ORCH_TOKEN}"
  retry_max: 3
  timeout_ms: 5000
```

### 11.2 Kubernetes Operator (Planned, Post-PoC)

A Kubernetes operator watches Meteridian's balance state and patches
orchestrator CRDs (e.g., `ResourceQuota`, `NetworkPolicy`, provider-specific
admission policies):

```yaml
enforcement:
  mode: k8s-operator
  watched_resource: "credits.meteridian.io/v1/Balance"
  target_resources:
    - apiVersion: "v1"
      kind: "ResourceQuota"
      namespace: "{{ .tenant_namespace }}"
      patch:
        spec.hard.requests.nvidia.com/gpu: "0"
```

### 11.3 API Gateway Integration (Generic)

For non-Envoy gateways (Kong, Nginx, Traefik, AWS API Gateway), Meteridian
exposes a standard rate-limit API that the gateway can query:

```
GET /api/v1/enforcement/rate-limit?tenant_id=alice&feature_key=tokens
```

Response follows the standard Rate Limit Service protocol headers:

```
X-RateLimit-Limit: 1000000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1719849600
```

---

## 12. PoC Scope

### 12.1 What's In Scope for the AI Grid PoC (July 31, 2026)

| Capability | Implementation | Effort |
|-----------|---------------|--------|
| Balance Check API (Path A) | REST endpoint serving MaaS plugin balance queries | 2-3 days |
| Limitador counter push (Path B, basic) | HTTP client pushing counter updates on budget exhaustion | 2-3 days |
| Threshold configuration | Configurable thresholds (75%, 90%, 100%) with actions | 1 day |
| Enforcement restore on top-up | Reset Limitador counters when balance is replenished | 1 day |
| Integration testing | Test with MaaS metering simulator and Limitador dev instance | 2 days |

**Total PoC effort: ~8-10 person-days (~2 weeks, 1 engineer)**

### 12.2 What's Deferred to Post-PoC

| Capability | Reason |
|-----------|--------|
| CRD-based enforcement (RateLimitPolicy patching) | Requires Kubernetes operator; counter manipulation is simpler for PoC |
| Model tier downgrade enforcement | Requires per-model routing configuration in MaaS gateway |
| Graduated throttling (multiple throttle levels) | Binary allow and block sufficient for PoC |
| Kubernetes operator for non-Limitador enforcement | Post-PoC, for environments without Red Hat Connectivity Link |
| Multi-cluster enforcement federation | Post-PoC, requires cross-cluster communication |

---

## 13. Open Questions

1. **Limitador API stability** — Limitador's HTTP API is not yet formally
   versioned with stability guarantees. Should we abstract behind an
   interface to allow for API changes, or depend directly on the current API?

2. **Counter TTL vs CRD-based limits** — Counter manipulation is ephemeral
   (counters expire after TTL). CRD-based limits are persistent. Should
   budget-exhaustion enforcement use persistent limits (CRD) or ephemeral
   counters with periodic refresh? The PoC uses counters for simplicity.

3. **Multi-feature enforcement** — If a tenant has multiple feature keys
   (tokens AND gpu_hours), should budget exhaustion on one feature block
   all features or only the exhausted one? Current design: per-feature
   enforcement.

4. **Enforcement audit and compliance** — Enforcement actions must be
   auditable for dispute resolution. Should enforcement signals be written
   to the same cryptographic audit trail (ADR-0014) as billing events, or
   a separate enforcement log?

5. **Rate-limit granularity** — Limitador can enforce at different
   granularities (per-user, per-group, per-subscription, per-API-key).
   Which granularity does the AI Grid PoC require? Current assumption:
   per-user (matching MaaS `x-maas-username` header).

---

## 14. Related Documents

### Enhancement Proposals

- [METR-0004](../0004-credit-token-billing/credit-token-billing.md) — Credit,
  Prepaid, and Token Billing (balance management, bill shock prevention)
- [METR-0010](../0010-ai-metering/ai-metering.md) — AI Workload Metering
  (MaaS plugin, CloudEvents, Limitador context)
- [METR-0002](../0002-extensibility/extensibility.md) — Platform Extensibility
  (block model for enforcement sink)

### Architecture Decision Records

- [ADR-0002](../../docs/adr/0002-valkey-balance-management.md) — Valkey for
  real-time balance management
- [ADR-0018](../../docs/adr/0018-closed-loop-enforcement-limitador.md) —
  Closed-loop enforcement via Limitador (architectural decision)

### Prospect Analysis

- [AI Grid PoC Gap Analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md)
  — MB-002, MB-006 enforcement gaps

### External References

- [Limitador (Kuadrant)](https://github.com/Kuadrant/limitador) — Generic rate
  limiter with Envoy integration and Redis-backed counters
- [Limitador HTTP API](https://docs.kuadrant.io/limitador/) — API reference for
  counter manipulation and limit management
- [Red Hat Connectivity Link](https://www.redhat.com/en/technologies/cloud-computing/connectivity-link) —
  Kubernetes-native gateway, auth, and rate limiting framework
- [RateLimitPolicy CRD](https://docs.kuadrant.io/kuadrant-operator/doc/reference/ratelimitpolicy/) —
  Kubernetes-native rate limit configuration
- [MaaS External Metering Plugin](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320) —
  IPP plugin with request-phase balance check
