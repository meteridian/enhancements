# METR-0015: Order Management and Service Activation

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-25
- **Last Updated:** 2026-06-25
- **Depends on:** METR-0003 (Product Catalog), METR-0004 (Credit and Token Billing — §4.0 Quote, §4.1 Contracts)
- **Related:** METR-0001 (Architecture — UDLM lifecycle), METR-0009 (E-Invoicing — billing trigger), METR-0011 (Enforcement — suspend/terminate), METR-0012 (Multi-Cloud Metering — provisioned resource tracking)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Order Data Model](#3-order-data-model)
4. [Order Lifecycle FSM](#4-order-lifecycle-fsm)
5. [Order API](#5-order-api)
6. [Provisioning Integration](#6-provisioning-integration)
7. [Metering Activation](#7-metering-activation)
8. [Enforcement Bridge](#8-enforcement-bridge)
9. [Fluxo Workflow Orchestration](#9-fluxo-workflow-orchestration)
10. [Non-Goals](#10-non-goals)
11. [Open Questions](#11-open-questions)
12. [Related Documents](#12-related-documents)

---

## 1. Summary

Order Management bridges the gap between a **commercial agreement** (quote
acceptance or contract signing) and **operational reality** (a running service
that Meteridian meters and bills). It provides:

- A lightweight `Order` entity with a well-defined lifecycle FSM
- CloudEvent emission on state transitions for external provisioning systems
- Webhook callbacks for provisioning confirmation
- Automatic metering pipeline activation when an order becomes active
- Integration with enforcement (METR-0011) for order suspension/termination

**Key principle:** Meteridian does NOT provision infrastructure. It signals
that provisioning should happen and waits for confirmation. The actual
provisioning is owned by external systems (K8s operators, Terraform,
Crossplane, HEAT, manual processes).

---

## 2. Motivation

### 2.1 The Missing Link

Today, Meteridian's flow has a gap between "contract signed" and "metering
starts":

```
Catalog → Quote → Contract → [???] → Metering → Rating → Billing
```

Without Order Management, operators must:
1. Manually configure metering for each new customer
2. Track "which services are active for which tenant" in spreadsheets
3. Rely on implicit activation (first usage event = service exists)
4. Have no clean mechanism for "we sold it but haven't deployed it yet"

### 2.2 Use Cases

| Scenario | Without Orders | With Orders |
|----------|---------------|-------------|
| MSP onboards new customer | Manually create metering config, hope billing catches up | Accept quote → order created → provisioning webhook → metering auto-activates |
| Customer upgrades plan | Ad-hoc change; billing inconsistency during transition | Modify order → reprovisioning signal → seamless metering switch |
| Customer churns | Disable metering manually; risk billing ghost resources | Terminate order → enforcement suspend → metering stops → final invoice |
| Trial-to-paid conversion | Custom logic per deployment | Trial order (time-limited) → convert to paid order on acceptance |

### 2.3 UDLM Alignment

The architecture (METR-0001 §18) already describes UDLM lifecycle events:

> If DCM (UDLM realization) is the provisioning layer, Meteridian subscribes
> to lifecycle events (`entity.created`, `entity.decommissioned`) to trigger
> metering start/stop.

This enhancement formalizes that pattern: the Order is the internal entity
that tracks the lifecycle; UDLM/CloudEvents are the signaling mechanism.

---

## 3. Order Data Model

```go
type Order struct {
    ID              uuid.UUID        `json:"id"`
    TenantID        uuid.UUID        `json:"tenant_id"`
    QuoteID         *uuid.UUID       `json:"quote_id,omitempty"`
    ContractID      *uuid.UUID       `json:"contract_id,omitempty"`
    Status          OrderStatus      `json:"status"`
    Items           []OrderItem      `json:"items"`
    ActivatedAt     *time.Time       `json:"activated_at,omitempty"`
    SuspendedAt     *time.Time       `json:"suspended_at,omitempty"`
    TerminatedAt    *time.Time       `json:"terminated_at,omitempty"`
    ProvisioningRef string           `json:"provisioning_ref,omitempty"`
    Metadata        map[string]any   `json:"metadata,omitempty"`
    CreatedAt       time.Time        `json:"created_at"`
    UpdatedAt       time.Time        `json:"updated_at"`
}

type OrderItem struct {
    ID              uuid.UUID        `json:"id"`
    ProductID       uuid.UUID        `json:"product_id"`
    PlanID          uuid.UUID        `json:"plan_id"`
    Quantity        int              `json:"quantity"`
    ResourceRef     string           `json:"resource_ref,omitempty"`
    MeteringConfig  *MeteringConfig  `json:"metering_config,omitempty"`
}

type MeteringConfig struct {
    CollectorType   string           `json:"collector_type"`
    TargetEndpoint  string           `json:"target_endpoint,omitempty"`
    Labels          map[string]string `json:"labels,omitempty"`
    ActivateOnOrder bool             `json:"activate_on_order"`
}

type OrderStatus string

const (
    OrderPending       OrderStatus = "pending"
    OrderProvisioning  OrderStatus = "provisioning"
    OrderActive        OrderStatus = "active"
    OrderSuspended     OrderStatus = "suspended"
    OrderTerminated    OrderStatus = "terminated"
    OrderFailed        OrderStatus = "failed"
)
```

---

## 4. Order Lifecycle FSM

```
     create (from quote acceptance or API)
                    │
                    ▼
              ┌──────────┐
              │ PENDING  │
              └─────┬────┘
                    │ trigger provisioning
                    ▼
           ┌──────────────────┐
           │  PROVISIONING    │
           └───────┬──────┬───┘
                   │      │
            success│      │failure
                   ▼      ▼
            ┌────────┐  ┌────────┐
            │ ACTIVE │  │ FAILED │
            └───┬────┘  └────────┘
                │
         ┌──────┼───────┐
         │      │       │
    suspend  terminate  modify
         │      │       │
         ▼      ▼       │
   ┌───────────┐ ┌────────────┐  │
   │ SUSPENDED │ │ TERMINATED │  │
   └─────┬─────┘ └────────────┘  │
         │                        │
    resume│          ┌────────────┘
         │           │ (re-provision with new config)
         ▼           ▼
      ┌────────┐  ┌──────────────────┐
      │ ACTIVE │  │  PROVISIONING    │
      └────────┘  └──────────────────┘
```

### Transition Rules

| From | To | Trigger | Side Effects |
|------|-----|---------|-------------|
| — | `pending` | Quote accepted / API call | Emit `order.created` CloudEvent |
| `pending` | `provisioning` | Automatic (or manual trigger) | Emit `order.provisioning` CloudEvent; call provisioning webhook |
| `provisioning` | `active` | Provisioning callback confirms success | Emit `order.activated`; start metering (§7) |
| `provisioning` | `failed` | Provisioning callback reports failure or timeout | Emit `order.failed`; alert operator |
| `active` | `suspended` | Payment failure (METR-0011 §7.4) / manual / policy | Emit `order.suspended`; pause metering; enforcement block |
| `active` | `terminated` | Customer churn / contract end / write-off | Emit `order.terminated`; stop metering; final invoice trigger |
| `suspended` | `active` | Payment recovered / manual resume | Emit `order.resumed`; restart metering; enforcement restore |
| `failed` | `provisioning` | Operator retries provisioning | Re-emit `order.provisioning` |

---

## 5. Order API

```
POST   /api/v1/orders                           # Create order (from quote or direct)
GET    /api/v1/orders                           # List orders (filter by tenant, status, product)
GET    /api/v1/orders/{id}                      # Get order details
POST   /api/v1/orders/{id}/provision            # Trigger provisioning (if not auto)
POST   /api/v1/orders/{id}/activate             # Mark as active (provisioning callback)
POST   /api/v1/orders/{id}/suspend              # Suspend order
POST   /api/v1/orders/{id}/resume               # Resume suspended order
POST   /api/v1/orders/{id}/terminate            # Terminate order
POST   /api/v1/orders/{id}/modify               # Change plan/quantity (triggers re-provision)
GET    /api/v1/orders/{id}/events               # Audit trail of state transitions
```

### Provisioning Callback Endpoint

External provisioning systems call back to confirm completion:

```
POST   /api/v1/orders/{id}/callbacks/provisioned
```

```json
{
  "status": "success",
  "resource_ref": "namespace/tenant-bob-gpu-pool",
  "provisioned_at": "2026-06-25T10:30:00Z",
  "details": {
    "cluster": "prod-west-2",
    "namespace": "tenant-bob",
    "quota_cpu": "64",
    "quota_gpu": "8"
  }
}
```

On success, the order transitions to `active` and metering begins. On failure
(`"status": "failed"`), the order transitions to `failed` with error details
stored for operator review.

---

## 6. Provisioning Integration

Meteridian signals provisioning but does NOT execute it. Three integration
patterns are supported:

### 6.1 Webhook Push (Default)

On `pending` → `provisioning` transition, Meteridian calls a configured
webhook URL with the order details:

```json
{
  "event": "order.provisioning",
  "order_id": "ord-abc123",
  "tenant_id": "tenant-bob",
  "items": [
    {
      "product_id": "gpu-compute-a100",
      "plan_id": "committed-8gpu-monthly",
      "quantity": 1,
      "metering_config": {
        "collector_type": "nvidia_dcgm",
        "labels": { "tenant": "bob", "tier": "premium" }
      }
    }
  ],
  "callback_url": "https://meteridian.internal/api/v1/orders/ord-abc123/callbacks/provisioned"
}
```

### 6.2 CloudEvent Bus (Event-Driven)

For Kubernetes-native environments, provisioning signals are emitted as
CloudEvents on the NATS/Kafka bus:

```json
{
  "specversion": "1.0",
  "type": "billing.order.provisioning",
  "source": "meteridian/orders",
  "subject": "ord-abc123",
  "data": { /* same as webhook payload */ }
}
```

K8s operators, ArgoCD ApplicationSets, or Crossplane Compositions subscribe
to these events and create the requested resources.

### 6.3 TMF Forum TMF622 (Product Ordering API)

For telco/enterprise environments using TM Forum Open APIs, orders can be
emitted in TMF622 format for integration with existing OSS/BSS stacks.

---

## 7. Metering Activation

When an order transitions to `active`, Meteridian automatically activates
metering for the ordered items:

1. **Collector configuration:** If `MeteringConfig.activate_on_order` is true,
   deploy or enable the specified collector for the provisioned resource.
2. **Metering filter:** Add the order's `resource_ref` and labels to the
   metering pipeline's inclusion filter so events from this resource are
   processed.
3. **Billing linkage:** Associate metered events with the order's contract
   (§4.1) and rate plan for correct pricing.
4. **UDLM event:** Emit `entity.realized` (METR-0001 §18) to signal that
   this resource is now in the metering lifecycle.

On `suspended`: metering collection continues (for audit/reconciliation) but
rated charges are held pending. On `terminated`: metering stops after a grace
period (configurable, default 24h for final usage flush).

---

## 8. Enforcement Bridge

Orders integrate bidirectionally with METR-0011 enforcement:

### Inbound (enforcement → order)

| Enforcement Signal | Order Effect |
|-------------------|-------------|
| `block` (payment failure / budget exhausted) | Order transitions to `suspended` |
| `terminate` (write-off / policy violation) | Order transitions to `terminated` |
| `restore` (payment recovered) | Order transitions back to `active` |

### Outbound (order → enforcement)

| Order Transition | Enforcement Effect |
|-----------------|-------------------|
| `active` → `suspended` (manual) | Push Limitador block for tenant |
| `suspended` → `active` (resume) | Remove Limitador block |
| `terminated` | Remove all tenant enforcement config |

This creates a clean feedback loop: payment failure → enforcement block →
order suspension → metering pause → no new charges → customer pays → restore.

---

## 9. Fluxo Workflow Orchestration

Complex order fulfillment (multi-item, multi-step provisioning) uses Fluxo
workflows:

```yaml
order_fulfillment:
  name: "multi-resource-provisioning"
  trigger: "order.created"
  steps:
    - name: "validate_capacity"
      action: "check_resource_availability"
      params: { cluster: "{{ order.metadata.target_cluster }}" }
      on_failure: "fail_order"

    - name: "provision_namespace"
      action: "webhook"
      url: "{{ config.provisioning_webhook }}"
      payload: "{{ order.items | where: collector_type == 'kubernetes' }}"
      timeout: "5m"

    - name: "provision_gpu"
      action: "webhook"
      url: "{{ config.gpu_provisioning_webhook }}"
      payload: "{{ order.items | where: collector_type == 'nvidia_dcgm' }}"
      timeout: "10m"
      depends_on: "provision_namespace"

    - name: "activate"
      action: "activate_order"
      depends_on: ["provision_namespace", "provision_gpu"]
```

---

## 10. Non-Goals

This enhancement explicitly does NOT cover:

| Non-Goal | Rationale | Alternative |
|----------|-----------|-------------|
| **Infrastructure provisioning** | Meteridian is not Terraform/Crossplane | Delegate via webhook/CloudEvent |
| **Inventory management** | Tracking available capacity is an infrastructure concern | Provisioning systems own capacity |
| **Approval workflows for orders** | Approval lives in the quote stage (METR-0004 §4.0.6) | Orders are post-approval by definition |
| **Shopping cart / storefront** | UI concern for the Customer Portal | Portal calls Order API after checkout |
| **Service dependency graphs** | Complex topology management | External CMDB/service catalog |

---

## 11. Open Questions

1. **Provisioning timeout** — What is the default timeout before an order in
   `provisioning` state is marked `failed`? Proposed: configurable per product,
   default 30 minutes.

2. **Partial provisioning** — If an order has 3 items and only 2 provision
   successfully, should the order be `active` (partial) or `failed`? Proposed:
   `active` with a `partial` flag and alert.

3. **Order amendments vs. new orders** — When a customer changes quantity or
   plan mid-contract, should it be a modification of the existing order or a
   new order? Proposed: modification (preserves audit trail).

4. **Metering gap during provisioning** — If the resource starts generating
   usage before the provisioning callback arrives, should those events be
   buffered or discarded? Proposed: buffer with retroactive rating on
   activation.

---

## 12. Related Documents

### Enhancement Proposals

- [METR-0003: Product and Service Catalog](../0003-product-catalog/product-catalog.md) —
  defines the products and plans that can be ordered
- [METR-0004: Credit, Prepaid, and Token Billing](../0004-credit-token-billing/credit-token-billing.md) —
  §4.0 Quote Generation (pre-order), §4.1 Contracts (commercial backing for orders)
- [METR-0009: Native E-Invoicing Engine](../0009-e-invoicing-engine/e-invoicing-engine.md) —
  order termination triggers final invoice generation
- [METR-0011: Enforcement Integration](../0011-enforcement-integration/enforcement-integration.md) —
  bidirectional enforcement bridge (§7.4 payment failures suspend orders)
- [METR-0012: Multi-Cloud Metering](../0012-multi-cloud-metering/multi-cloud-metering.md) —
  orders may trigger activation of cloud metering pipelines

### Architecture

- [METR-0001: Platform Architecture](../0001-architecture/architecture.md) —
  §10 Consumption Control, §13 Portals (place orders), §18 UDLM lifecycle

### External References

- [TM Forum TMF622: Product Ordering API](https://www.tmforum.org/resources/specification/tmf622-product-ordering-api-rest-specification-r19-0-0/) —
  industry standard order API for telco/enterprise
- [UDLM Specification](https://udlm.org) —
  resource lifecycle management concepts adopted by Meteridian
- [Crossplane Compositions](https://docs.crossplane.io/latest/concepts/compositions/) —
  example of a provisioning system that can consume order events
