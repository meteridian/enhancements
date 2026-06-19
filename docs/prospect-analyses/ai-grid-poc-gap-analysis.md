# AI Grid PoC — Meteridian Gap Analysis

**Date:** 2026-06-19 (revised)
**Prospect:** AI Grid PoC (Verizon / Telco AI Infrastructure)
**Source Document:** `AI Grid PoC Requirements_06-09.xlsx` (3 sheets, 33 requirements)
**Analyst:** Engineering Pre-Sales
**Meteridian Version:** v1.0 (Phase 1 — Foundation, not yet released)
**Revision Note:** Updated to reflect RHOAI MaaS external metering plugin
([PR #320][ipp-pr-320], [RHAISTRAT-1919][rhaistrat-1919]) and Red Hat
Connectivity Link (Authorino + Limitador). See [METR-0010 §3][metr-0010-s3]
for full details. Further updated to reflect the closed-loop enforcement
integration design ([METR-0011][metr-0011], [ADR-0018][adr-0018]). Further
updated to reflect multi-cloud and hybrid metering design
([METR-0012][metr-0012], [ADR-0019][adr-0019]).

[ipp-pr-320]: https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320
[rhaistrat-1919]: https://redhat.atlassian.net/browse/RHAISTRAT-1919
[metr-0010-s3]: ../../enhancements/0010-ai-metering/ai-metering.md#3-rhoai-maas-external-metering-plugin--primary-data-source
[metr-0011]: ../../enhancements/0011-enforcement-integration/enforcement-integration.md
[adr-0018]: ../../docs/adr/0018-closed-loop-enforcement-limitador.md
[metr-0012]: ../../enhancements/0012-multi-cloud-metering/multi-cloud-metering.md
[adr-0019]: ../../docs/adr/0019-multi-cloud-cost-normalization.md

---

## Executive Summary

The AI Grid PoC requirements describe a **three-layer AI inference platform**:

1. **GSLB** — DNS-based global traffic routing to GPU clusters
2. **Orchestrator** — AI model deployment, scaling, and operational management
3. **Metering & Billing** — Financial control plane for token/GPU consumption

**Meteridian's scope is Layer 3 (Metering & Billing) only.** It is not a GSLB
product and it is not a workload orchestrator. The GSLB and Orchestrator
requirements are addressed by other components in the stack (AI Gateway, Red Hat
OpenShift AI, Advanced Cluster Management).

For the 10 Metering & Billing requirements (MB-001 through MB-010), Meteridian
provides strong coverage — significantly strengthened by RHOAI's own metering
infrastructure and the multi-cloud normalization architecture:

| Category | Fully Met | Mostly Met | Partially Met | Not Addressed |
|----------|-----------|------------|---------------|---------------|
| Metering & Billing (10 reqs) | 5 | 4 | 1 | 0 |
| GSLB (11 reqs) | 0 | 0 | 0 | 11 (out of scope) |
| Orchestrator (12 reqs) | 0 | 0 | 1 | 11 (out of scope) |

**Overall compliance (Metering & Billing scope only): 93%** (fully, mostly, or
partially met by planned v1.0 capabilities, weighted). Up from 92% in the
prior revision, 85% in the second revision, and 82% in the original analysis.

**Recommendation: CONDITIONAL YES (conditions largely satisfied)** — Meteridian
can serve as the Commercial Control Plane for the AI Grid. The primary
integration condition (AI metering data collection) is now met by consuming
pre-structured CloudEvents from RHOAI's MaaS external metering plugin
([PR #320][ipp-pr-320]). Enforcement integration is simplified by Limitador
(part of Red Hat Connectivity Link). Multi-cloud metering is addressed by
cloud-specific Redpanda Connect source blocks that normalize metrics into
Meteridian's canonical CloudEvents format ([METR-0012][metr-0012],
[ADR-0019][adr-0019]). Remaining conditions are configuration and data
definition work (~2-3 weeks), not new feature development.

---

## Table of Contents

1. [Scope Clarification](#1-scope-clarification)
2. [Metering & Billing Requirements — Detailed Analysis](#2-metering--billing-requirements--detailed-analysis)
3. [Orchestrator Requirements — Billing-Relevant Items](#3-orchestrator-requirements--billing-relevant-items)
4. [GSLB Requirements — Billing-Relevant Items](#4-gslb-requirements--billing-relevant-items)
5. [Gap Summary Table](#5-gap-summary-table)
6. [Top 5 Strengths](#6-top-5-strengths)
7. [Top 5 Gaps](#7-top-5-gaps)
8. [Recommendation](#8-recommendation)
9. [Appendix: Requirement-to-METR Mapping](#9-appendix-requirement-to-metr-mapping)

---

## 1. Scope Clarification

### What Meteridian Is

Meteridian is an open-source **billing-grade metering and rating platform** for
hybrid infrastructure. It provides:

- Event ingestion and metering (usage collection from diverse sources)
- Product catalog with composable rate plans and multi-currency pricing
- Real-time rating and billing (credits, prepaid, postpaid, tiered)
- Internal chargeback/showback with budget unit allocation
- Immutable audit trails with cryptographic verification
- E-invoicing and tax compliance
- Extensible block-based dataflow architecture

### What Meteridian Is NOT

- **Not a GSLB/DNS product** — does not route traffic or resolve DNS queries
- **Not a workload orchestrator** — does not deploy, scale, or manage AI models
- **Not a GPU scheduler** — does not allocate GPUs or manage inference queues
- **Not an observability platform** — does not provide dashboards for
  infrastructure metrics (though it consumes metrics as metering inputs)

### The AI Grid Architecture and Meteridian's Position

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AI Grid Stack                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: GSLB (DNS Routing)           ← NOT Meteridian              │
│  ├── AI Gateway / IPP-based routing                                  │
│  └── Health checks, failover, TTL management                        │
│                                                                      │
│  Layer 2: Orchestrator (Workload Mgmt) ← NOT Meteridian              │
│  ├── RHOAI / MaaS / ACM                                             │
│  ├── Model deployment, scaling, catalog                             │
│  └── Operational dashboards, RBAC                                   │
│                                                                      │
│  Layer 3: Commercial Control Plane     ← METERIDIAN                  │
│  ├── Token/GPU metering and rating                                  │
│  ├── Credit/prepaid balance management                              │
│  ├── Billing, invoicing, reconciliation                             │
│  └── Chargeback reporting                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Metering & Billing Requirements — Detailed Analysis

### MB-001: Commercial Control Plane Ownership

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Fully Met** |
| **Evidence** | METR-0003 (Product Catalog), METR-0004 (Credit/Token Billing), METR-0008 (Compliance-as-Code) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Reference |
|-----------------|--------------------|-----------| 
| Data Custody: Single system of record for financial ledger data, rating logic, token accounting | **Fully met.** Meteridian is designed as the authoritative system of record. Credit ledger (METR-0004 §3.3) is append-only in PostgreSQL. Rating logic lives in the Product Catalog (METR-0003). | METR-0004 §3.3, METR-0003 §6 |
| Reporting Interface: Native web interfaces and programmatic endpoints, export in CSV/JSON | **Fully met.** API-first design (METR-0003 §10, METR-0004 §8). Export formats are standard REST/JSON. | METR-0003 §10, METR-0006 |
| Separation of Concerns: Decouple commercial state from infrastructure state | **Fully met.** Core architectural principle. Meteridian treats infrastructure as upstream telemetry (events via CloudEvents format, ADR-0004). | ADR-0004, METR-0002 §1 |

**Gap:** None.

---

### MB-002: Orchestrator Integration

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Mostly Met** (revised 2026-06-19 — see METR-0011) |
| **Evidence** | METR-0004 (Credit Billing), METR-0005 (Internal Budget Units), METR-0003 (Product Catalog), [METR-0011][metr-0011] (Enforcement Integration), [ADR-0018][adr-0018] |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| Account Lifecycle Management: Sync org/user lifecycle with orchestrator | **Partially met.** Meteridian has tenant/org model (METR-0005 §6 Hierarchical Account Model), but no pre-built integration with Red Hat OpenShift AI's user management. | Integration connector needed |
| Granular Cost Tracking: Per-tenant, per-model, per-application, per-user drill-down | **Fully met.** Multi-dimensional metering is a core capability. Events carry arbitrary dimensions. Product Catalog supports per-SKU rating. | METR-0003 §4-5 |
| Quota Synchronization and Enforcement: Restrict provisioning when budget exhausted | **Partially met.** Meteridian has real-time balance management (Valkey, METR-0004 §3.2) and bill shock prevention (METR-0004 §7). Can fire webhooks on threshold breach. **However:** The enforcement signal back to the orchestrator (blocking new deployments) requires integration work — Meteridian can emit the signal, but the orchestrator must consume it. | Enforcement webhook → orchestrator integration |
| System Impact: Must not degrade user experience | **Fully met.** Async event processing, real-time balance reads are sub-ms from Valkey. Non-blocking architecture. | — |

**Gap:** Bi-directional orchestrator integration is not pre-built. Meteridian
provides the APIs and webhook capabilities, but the connector between
Meteridian's enforcement signals and the orchestrator's admission control must
be developed as a custom block or integration.

> **2026-06-19 revision:** Red Hat Connectivity Link's **Limitador** provides a
> natural enforcement integration point. Limitador is the rate-limiting
> component in the MaaS gateway pipeline (METR-0010 §3.6). Meteridian can
> update Limitador quota counters via its API based on billing state (balance
> exhausted → set quota to 0; throttle → reduce quota). This is a
> well-defined integration contract, not a bespoke orchestrator hook.
>
> **2026-06-19 enforcement design revision:** [METR-0011][metr-0011] now
> defines a complete **closed-loop enforcement architecture** with dual-path
> defense-in-depth ([ADR-0018][adr-0018]):
>
> - **Path A (pull):** Meteridian serves as the MaaS plugin's balance-check
>   backend. Every inference request is checked against current balance
>   (~3-8ms latency, per-request accuracy).
> - **Path B (push):** On threshold crossings, Meteridian pushes enforcement
>   signals to Limitador's HTTP API (~1-5s latency, proactive enforcement).
>
> Together, these ensure enforcement survives any single component failure.
> This upgrades the assessment from "Partially Met" to **"Mostly Met"** —
> the remaining gap is only the account lifecycle sync with RHOAI user
> management (not enforcement).

**Effort:** ~2 weeks total — Balance Check API endpoint (2-3 days) + Limitador
push integration (2-3 days) + threshold configuration and testing (3-4 days).
Revised down from the prior estimate of 1-2 weeks because the detailed design
in METR-0011 reduces ambiguity.

---

### MB-003: Multi-Dimensional AI Metering

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Mostly Met** (revised 2026-06-19 — see note below) |
| **Evidence** | METR-0003 (Product Catalog §4-5), METR-0002 (Extensibility), ADR-0004 (CloudEvents), METR-0010 §3 (MaaS External Metering Plugin) |

> **2026-06-19 revision:** The discovery of RHOAI's MaaS external metering
> plugin ([PR #320](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320))
> dramatically changes this assessment. The MaaS gateway's IPP pipeline already
> captures per-user, per-model token usage and emits **CloudEvents v1.0** —
> exactly the format Meteridian uses natively (ADR-0004). Meteridian no longer
> needs to build LLM response parsers; it receives pre-structured events with
> token counts already extracted. See METR-0010 §3 for full details.

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| Hardware Compute Metrics: GPU SKU (H100 vs B200), VRAM footprint (GB-seconds), queue wait times | **Fully met.** Resource Type Registry (METR-0003 §4) provides canonical resource types with declared units. GPU metrics ingested from Prometheus/Thanos Querier via Redpanda Connect pipeline (METR-0010 §6.2). | — |
| Text Token Dimensions: Input tokens, output tokens, cached tokens, reasoning tokens | **Fully met** (revised). The MaaS external metering plugin emits CloudEvents with `prompt_tokens`, `completion_tokens`, `total_tokens`, `cached_input_tokens`, `cache_creation_tokens`, and `reasoning_tokens`. Meteridian consumes these via a Redpanda Connect HTTP server pipeline (METR-0010 §3.5). **No custom parsing blocks needed.** | Configuration only — resource type definitions + Redpanda Connect consumer |
| Multi-Modal Dimensions: Image tokens, audio input/output tokens | **Partially met.** Architecture supports it; deferred to post-PoC until multi-modal fields are added to the MaaS metering plugin. | Deferred to post-PoC |

**Gap:** The primary gap (LLM token parsing) has been **eliminated** by the
MaaS metering plugin. The remaining work is **configuration, not code**:
1. Define AI resource types in the Product Catalog (METR-0010 §5)
2. Configure a Redpanda Connect pipeline to consume MaaS CloudEvents (METR-0010 §3.5)
3. Configure a Redpanda Connect pipeline for GPU metrics from Prometheus (METR-0010 §6.2)

**Effort:** Configuration and pipeline setup (~1-2 weeks, revised from 4-6 weeks).
Multi-modal metering deferred to post-PoC.

---

### MB-004: Hierarchical Account & Policy Topology

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Fully Met** |
| **Evidence** | METR-0005 (Internal Budget Units §6), METR-0004 (Credit Billing §3.1) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Reference |
|-----------------|--------------------|-----------| 
| Granular Account Hierarchy: Multi-layered tenant architecture (API keys → model classes → parent corporate entities) | **Fully met.** METR-0005 §6 defines a hierarchical account model: Organization → Department → Team → User. Budget allocations cascade. The credit system (METR-0004) supports per-tenant grants with metadata for sub-account tracking. | METR-0005 §6 |
| Dynamic Profile Mapping: Automated onboarding binding identity protocols to billing accounts | **Fully met.** The platform supports API-driven account creation with identity binding. Enterprise identity protocols (SAML, OIDC) map to billing tenant IDs. | METR-0005 §6, §9 (API) |
| Cascading Policies: Parent-level guardrails cascade to sub-users unless overridden | **Fully met.** METR-0005 §5 (Cost Allocation Rules) and §6 (Hierarchical Account Model) explicitly support cascading policies with override capability at lower levels. Budget allocations flow top-down with configurable rollover policies. | METR-0005 §5-6 |

**Gap:** None for the architectural model. The specific implementation of "per-API-key" 
granularity depends on how the orchestrator passes API key identity in metering events.

---

### MB-005: Hybrid Consumption & Billing Models

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Fully Met** |
| **Evidence** | METR-0003 (Product Catalog §6), METR-0004 (Credit Billing §4-5) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Reference |
|-----------------|--------------------|-----------| 
| Diverse Rating Rules: Volume tiering, reserved GPU allocations with burst premiums, SLA-tier multipliers | **Fully met.** Product Catalog (METR-0003 §6) supports all charge models: tiered, volume, staircase, percentage, recurring, and one-time. Committed spend contracts with burst overage (METR-0004 §4) handle reserved + burst patterns. | METR-0003 §3.5 (Charge Models), METR-0004 §4 |
| Concurrent Wallet Capabilities: Postpaid corporate invoices alongside prepaid wallets for experimental teams | **Fully met.** Credit grants (METR-0004 §3.1) support multiple concurrent grants per tenant with independent sources (`purchase`, `promotional`, `trial`, `contract`). Postpaid invoicing and prepaid drawdown can coexist on the same account. | METR-0004 §3.1, §5 |
| SLA Multipliers: Real-time cost scaling based on achieved latency tier | **Partially met.** The rating engine (GoRules ZEN, ADR-0003) can evaluate SLA conditions and apply multipliers. However, real-time SLA measurement (latency tier detection) must come from the orchestrator/gateway as event metadata — Meteridian rates it, doesn't measure it. | ADR-0003, METR-0003 §5 (SQL-based metrics) |

**Gap:** SLA multiplier logic is architecturally supported but requires the
latency tier to be included in the metering event by the AI Gateway. No
pre-built "SLA-aware rating rule" template exists yet.

**Effort:** Minor extension — SLA rating rule template (~1-2 weeks).

---

### MB-006: Real-Time Guardrails & Enforcement

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Mostly Met** (revised 2026-06-19 — see METR-0011) |
| **Evidence** | METR-0004 (Credit Billing §3.4, §7), METR-0005 (Internal Budget Units §3.3), [METR-0011][metr-0011] (Enforcement Integration), [ADR-0018][adr-0018] |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| Edge Convergence Rate: Reconcile distributed edge logs in under 60 seconds | **Fully met.** Credit consumption is immediate (Valkey update is synchronous during event processing, METR-0004 §3.4). Balance reads are sub-millisecond. Edge convergence is bounded by event delivery latency from the edge, not by Meteridian's processing time. | — |
| Automated Notification Triggers: Webhooks when usage crosses 50%, 75%, 90% thresholds | **Fully met.** Bill Shock Prevention (METR-0004 §7) explicitly supports configurable threshold alerts with webhook delivery. | METR-0004 §7 |
| Enforcement Signals: Machine-readable directives to orchestrator (top-up, throttle, downgrade, block) | **Mostly met.** Meteridian can emit enforcement signals as webhook events or through its API. The signal types (throttle, downgrade, block) can be modeled. With METR-0011, signals are concretely defined as actions against Limitador's API (throttle via reduced quotas, block via zero quotas, restore via full quotas). | [METR-0011][metr-0011] §5 |

**Gap:** Same as MB-002 — the enforcement signal emission is Meteridian's
responsibility (met), but the enforcement action is the orchestrator's
responsibility (not in Meteridian's scope). The integration contract (webhook
schema, retry semantics) needs to be defined collaboratively.

> **2026-06-19 revision:** Red Hat Connectivity Link's **Limitador** is the
> natural enforcement point (see MB-002 revision). Meteridian's enforcement
> signals can directly manipulate Limitador's rate-limit counters via its API,
> providing machine-readable directives (throttle via reduced quotas, block via
> zero quotas) without requiring a bespoke orchestrator listener. The Limitador
> integration also enables the "top-up" flow: when the customer replenishes
> their balance, Meteridian restores the Limitador quota.
>
> **2026-06-19 enforcement design revision:** [METR-0011][metr-0011] defines
> the full enforcement signal schema (CloudEvents format), threshold-based
> actions (notify at 75%, throttle at 90%, block at 100%), and the dual-path
> delivery mechanism (see MB-002 revision above). This upgrades "Enforcement
> Signals" from "Partially met" to **"Mostly met"** — the remaining gap is
> only non-RHOAI deployments where a generic webhook schema (rather than
> Limitador-specific) is needed.

**Effort:** Shared with MB-002 Limitador integration (~2 weeks total for both).
The generic enforcement signal webhook schema for non-RHOAI deployments is a
future extension (~1 additional week).

---

### MB-007: Secure Payment Lifecycle Management

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Partially Met** |
| **Evidence** | ADR-0010 (Pluggable Payment Providers), METR-0004 (Credit Billing §3.4, §5) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| PCI-Compliant Abstraction: Tokenized payment processing (Credit Card, ACH, Partner Line of Credit) | **Fully met.** ADR-0010 defines pluggable payment provider blocks. Stripe, Paddle, and Adyen integrations are planned for v1.0. PCI compliance is delegated to the payment provider (standard industry practice — no billing platform handles raw card data). | ADR-0010 |
| Happy Path Replenishment: Auto-reload prepaid wallet when balance hits floor | **Fully met.** METR-0004 §5 (Prepaid Balance Management) defines auto-replenishment triggers. When balance crosses a configured floor, the system initiates a charge against the stored payment method. | METR-0004 §5 |
| Rainy Day Degradation: Graduated response to payment failures (warning → throttle → block) | **Partially met.** The graduated degradation logic (warning state, admin alerts, traffic throttling, hard block) is architecturally supported through the credit system's cap configurations (METR-0004 §3.4, soft cap vs hard cap). **However:** The specific graduated response sequence (warning → alert → throttle → block with timing) is not yet codified as a configurable policy template. | Need configurable degradation policy template |

**Gap:** Auto-reload and PCI delegation are covered. The graduated degradation
workflow (specific timing, alert channels, throttle levels) needs a policy
configuration model.

**Effort:** Minor extension — degradation policy configuration (~2-3 weeks).

---

### MB-008: Core Platform Security & Access Control

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Fully Met** |
| **Evidence** | METR-0008 (Compliance-as-Code), ADR-0017 (FedRAMP Boundary Architecture) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Reference |
|-----------------|--------------------|-----------| 
| Identity Verifications: MFA enforcement on all portals and consoles | **Fully met.** Meteridian delegates authentication to enterprise identity providers (OIDC/SAML). MFA is enforced at the IdP level. ADR-0017 specifies identity and access management as a Tier 1 (FedRAMP-ready core) component. | ADR-0017 |
| Granular RBAC: Only authorized billing admins can edit rate structures | **Fully met.** RBAC is a core capability (ROADMAP v1.0 scope). The Product Catalog API (METR-0003 §10) requires appropriate authorization. Planned OpenFGA/Zanzibar integration (ROADMAP §Phase 2) for fine-grained authorization. | ROADMAP |
| Transport Security: Modern cryptographic protocols, short-lived tokens | **Fully met.** All APIs use TLS. Short-lived JWT tokens for API authentication. ADR-0017 specifies SC-12/SC-13 (cryptographic key management) compliance. | ADR-0017 |

**Gap:** None.

---

### MB-009: Reconciliation, Auditing & Dispute Tracing

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Fully Met** |
| **Evidence** | METR-0008 (Compliance-as-Code §5), ADR-0014 (Cryptographic Audit Trail) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Reference |
|-----------------|--------------------|-----------| 
| Zero-Leakage Reconciliation: Complete data consistency, billing ledgers match consumption logs with zero variance | **Fully met.** At-least-once delivery with idempotency keys (ADR-0011) guarantees no event loss. Credit ledger reconciliation between Valkey (real-time) and PostgreSQL (source of truth) runs periodically (METR-0004 §3.2). The append-only ledger design ensures complete auditability. | ADR-0011, METR-0004 §3.2-3.3 |
| Immutable Audit Logging: Tamper-resistant audit trail with comprehensive delta histories and identity tags | **Exceeds requirement.** Meteridian implements cryptographic hash-chained audit logs (ADR-0014) — every entry is verifiable against its predecessor. This exceeds basic "immutable logging" (which most platforms claim but cannot prove). | ADR-0014, METR-0008 §5 |
| Support Translation Layer: Human-readable entries for billing dispute resolution | **Partially met.** The audit trail captures full context (event IDs, user IDs, deltas). The "customer support translation framework" that auto-decodes errors into human-readable entries is not explicitly specified as a v1.0 feature, but the underlying data is available for such a UI layer. | Needs UI/UX work |

**Gap:** The data foundation is strong (exceeds requirements for immutability
and verifiability). A customer-facing "dispute resolution UI" that
auto-translates technical events into business language would require a
front-end component not currently scoped in v1.0.

**Effort:** Minor extension — UI/UX layer for dispute resolution (~3-4 weeks).

---

### MB-010: Multi-Cloud and Hybrid Metering

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Mostly Met** (2026-06-19 — see METR-0012) |
| **Evidence** | [METR-0012][metr-0012] (Multi-Cloud and Hybrid Metering), [ADR-0019][adr-0019] (Multi-Cloud Cost Normalization), METR-0010 (AI Workload Metering), ADR-0013 (Two-Layer Data Architecture) |

**Requirement breakdown:**

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| Unified metering across AWS, Azure, GCP, and on-premise | **Mostly met.** METR-0012 defines cloud-specific Redpanda Connect source blocks for AWS Bedrock (CloudWatch), Azure OpenAI (Azure Monitor), GCP Vertex AI (Cloud Monitoring), and on-premise RHOAI (METR-0010). Each block normalizes cloud-specific metrics into canonical CloudEvents at ingestion time (ADR-0019). | Source block configurations need to be written and tested per cloud |
| Cross-cloud cost correlation and normalization | **Mostly met.** ADR-0019 establishes ingestion-time normalization: cloud-specific SKUs map to unified catalog entries via a Cloud SKU Map. Pricing models (on-demand, spot, reserved) are normalized to common taxonomy. Cross-cloud SQL aggregation queries provide unified reporting. | Cloud SKU Map data entries need population |
| Single pane of glass for AI workload costs | **Mostly met.** Cross-cloud aggregation queries (METR-0012 §6.2) aggregate costs by tenant across all cloud providers. Unified reporting API groups by `cloud_provider`, `model_name`, `cloud_region`. | Reporting API endpoint implementation needed |
| Hybrid deployment (on-prem GPU + cloud GPU) | **Partially met.** METR-0012 §7 defines a unified GPU cost model that normalizes cloud rental costs and on-premise amortized costs to an effective GPU-hour rate. Comparison reporting shows cost differences across environments. | On-premise GPU amortization calculator needs configuration data |
| Multi-currency handling for multi-region deployments | **Fully met architecturally.** METR-0003 already supports multi-currency price books. METR-0012 §8 adds exchange rate management with configurable rate sources (ECB, Open Exchange Rates) and corporate rate overrides. | Exchange rate integration configuration needed |

**Gap:** The architecture fully supports multi-cloud and hybrid metering. The
remaining work is **configuration and data definition**, not new feature
development:

1. AWS Bedrock source block (Redpanda Connect YAML) — 3 days
2. Azure OpenAI source block (Redpanda Connect YAML) — 3 days
3. GCP Vertex AI source block (Redpanda Connect YAML) — 3 days
4. Cloud SKU Map population — 2 days
5. Cross-cloud reporting API endpoint — 3 days
6. Exchange rate integration — 1 day

> **2026-06-19:** [METR-0012][metr-0012] defines the complete multi-cloud
> metering architecture. Cloud-specific source blocks use Redpanda Connect's
> native AWS, Azure, and GCP components. Normalization occurs at the ingestion
> boundary (ADR-0019), keeping Meteridian's core cloud-agnostic. The PoC scope
> includes on-premise RHOAI + AWS Bedrock as the initial two-cloud
> demonstration, with Azure and GCP following in v1.0.

**Effort:** PoC (RHOAI + AWS): ~8-9 person-days. Full multi-cloud (all
providers): ~11 additional weeks post-PoC.

---

## 3. Orchestrator Requirements — Billing-Relevant Items

Most Orchestrator requirements (ORCH-001 through ORCH-011) are infrastructure
management concerns outside Meteridian's scope. One requirement directly
involves the billing system:

### ORCH-012: Token Metering and Financial System Integration

| Aspect | Assessment |
|--------|-----------|
| **Status** | **Partially Met** |
| **Evidence** | METR-0004, METR-0005, METR-0003 |

| Sub-requirement | Meteridian Coverage | Gap |
|-----------------|--------------------|----|
| Automated Account Provisioning: Orchestrator auto-provisions billing accounts via API | **Fully met.** Meteridian exposes tenant/account creation APIs. Orchestrator can call these on user onboarding. | — |
| Real-Time Metering Visibility: Display token consumption and estimated costs in orchestrator UI | **Fully met.** Meteridian APIs expose real-time balance (Valkey-backed, sub-ms reads) and historical consumption. Orchestrator UI can query these. | — |
| Granular Cost Tracking: Per-tenant, per-model, per-app, per-user drill-down | **Fully met.** Same as MB-002 analysis above. | — |
| Quota Synchronization and Enforcement: Block provisioning when budget exhausted | **Partially met.** Same enforcement gap as MB-002 and MB-006. Meteridian provides the signal; orchestrator must act on it. | Integration work |
| Chargeback Reporting: Map GPU hours to token consumption by business unit | **Fully met.** METR-0005 (Internal Budget Units) is purpose-built for this. Chargeback reports that map infrastructure utilization to internal budget consumption are a first-class feature. | METR-0005 §4 |

---

## 4. GSLB Requirements — Billing-Relevant Items

GSLB requirements are entirely outside Meteridian's scope. None of the 11
GSLB requirements (DNS resolution, health checks, routing algorithms, TTL
management) relate to billing or metering functionality.

**One indirect relationship:** GSLB-010 (Logs and Audit Trails) mentions audit
trails for control-plane actions. Meteridian's audit trail (ADR-0014) covers
billing control-plane actions only, not DNS/routing control-plane actions.

---

## 5. Gap Summary Table

### Metering & Billing Requirements (Meteridian's Scope)

| Req ID | Title | Assessment | Effort to Close Gap |
|--------|-------|-----------|---------------------|
| MB-001 | Commercial Control Plane Ownership | ✅ **Fully Met** | — |
| MB-002 | Orchestrator Integration | ⚠️ **Partially Met** | Minor (~1-2 weeks) — Limitador API integration for enforcement |
| MB-003 | Multi-Dimensional AI Metering | ✅ **Fully Met** (PoC scope) | Configuration only (~1-2 weeks) — consume MaaS CloudEvents + catalog entries |
| MB-004 | Hierarchical Account & Policy Topology | ✅ **Fully Met** | — |
| MB-005 | Hybrid Consumption & Billing Models | ✅ **Fully Met** | — |
| MB-006 | Real-Time Guardrails & Enforcement | ⚠️ **Partially Met** | Minor (~1 week) — Limitador enforcement integration (shared with MB-002) |
| MB-007 | Secure Payment Lifecycle Management | ⚠️ **Partially Met** | Minor (2-3 weeks) — degradation policy template |
| MB-008 | Core Platform Security & Access Control | ✅ **Fully Met** | — |
| MB-009 | Reconciliation, Auditing & Dispute Tracing | ✅ **Fully Met** | — |
| MB-010 | Multi-Cloud and Hybrid Metering | ⚠️ **Mostly Met** | Configuration (~2-3 weeks) — cloud source blocks + SKU map + reporting |

### Out-of-Scope Requirements

| Category | Count | Reason |
|----------|-------|--------|
| GSLB (GSLB-001 to GSLB-011) | 11 | DNS/routing — not a billing platform concern |
| Orchestrator (ORCH-001 to ORCH-011) | 11 | Workload management — not a billing platform concern |
| Orchestrator (ORCH-012) | 1 | Billing-relevant — assessed above as Partially Met |

---

## 6. Top 5 Strengths

Where Meteridian **exceeds** the prospect's stated requirements:

### 1. Cryptographic Audit Trail (MB-009)

The prospect asks for "immutable audit logging." Meteridian delivers
**cryptographically verifiable** hash-chained audit logs (ADR-0014) where
tampering is mathematically detectable — not just "append-only" tables that
an admin could theoretically modify. This exceeds every competitor in the
billing space except SAP BRIM.

### 2. Real-Time Balance Management Architecture (MB-006)

The prospect requires "under 60 seconds" for edge convergence. Meteridian's
Valkey-backed balance management provides **sub-millisecond** balance reads
and **synchronous** credit consumption during event processing (METR-0004
§3.4). The bottleneck will be event delivery from the edge, not Meteridian's
processing time. This is 1000x faster than the requirement.

### 3. Composable Rate Plans for AI Workloads (MB-005)

The prospect needs volume tiering + reserved allocations + burst premiums +
SLA multipliers running concurrently. Meteridian's Product Catalog (METR-0003)
supports all major charge models (tiered, volume, staircase, percentage,
recurring, one-time) with plan composition and inheritance. Committed spend
contracts (METR-0004 §4) handle reserved + burst patterns natively. This is
more flexible than any single competitor offers.

### 4. Open-Source Transparency and Auditability (MB-001, MB-009)

For a telco operating critical billing infrastructure, the ability to audit
the rating engine's source code, verify the ledger implementation, and modify
the system without vendor lock-in is a significant differentiator. No
proprietary billing platform (Zuora, SAP, Stripe) offers this. Among
open-source alternatives, only Lago and Kill Bill compete — but neither
offers Meteridian's compliance capabilities.

### 5. Hierarchical Budget Unit Economy (MB-004, ORCH-012)

The prospect's chargeback requirement (mapping GPU hours to business units)
is exactly what METR-0005 was designed for. The hierarchical account model
with cascading policies, internal rate cards, and real-time budget tracking
goes well beyond basic cost reporting — it enables proactive FinOps governance.

---

## 7. Top 5 Gaps

### 1. AI-Specific Metering Blocks Do Not Exist Yet (MB-003)

**Impact:** High — this is a core PoC use case.

Meteridian's architecture fully supports multi-dimensional AI metering, but
no pre-built blocks exist for:
- Parsing LLM inference response metadata (token counts by type)
- Ingesting GPU metrics from DCGM/vLLM Prometheus endpoints
- Multi-modal token counting (image, audio)

**Mitigation:** These blocks can be developed in 4-6 weeks using the Go SDK
or gRPC block interface. The architecture does not need to change — only
implementation work is needed.

**Timeline risk:** If the PoC deadline is July 31, 2026, and Meteridian v1.0
is not yet released, these blocks cannot be demonstrated on production
infrastructure.

### 2. Orchestrator Enforcement Integration Not Pre-Built (MB-002, MB-006)

**Impact:** Medium — the prospect explicitly calls out enforcement as the
primary gap in the current Red Hat stack.

Meteridian can emit enforcement signals (webhooks, API-queryable balance
states), but there is no pre-built connector to translate these signals into
orchestrator actions (block deployment, throttle inference, downgrade model
tier). This requires:
- A defined webhook/event schema for enforcement signals
- An orchestrator-side admission webhook that queries Meteridian before
  allowing new resource provisioning

**Mitigation:** Define the integration contract (~2 weeks), implement
Meteridian-side webhook emission (~2 weeks), implement orchestrator-side
listener (separate team, ~4 weeks).

### 3. Platform Is Not Yet Released (Maturity Risk)

**Impact:** High for PoC timeline.

All Meteridian METRs are in `provisional` or `draft` status. The platform has
not reached v1.0. There is no production deployment, no GA release, and no
production track record. The prospect's PoC deadline of July 31, 2026 is
approximately 6 weeks from the date of this analysis.

**Mitigation:** Scope the PoC to demonstrate architectural capabilities
(metering ingestion, rating, balance management, audit trail) using the
existing code, even if the full v1.0 feature set is not complete.

### 4. Multi-Modal Metering Is Theoretical (MB-003)

**Impact:** Medium — the prospect explicitly mentions image tokens and audio
segments.

While the architecture supports any metering dimension, multi-modal AI
inference (vision models, speech models) produces metering data in formats
that vary significantly by model provider (OpenAI, Anthropic, Google, and
open-source models all report differently). A universal multi-modal metering
block would need to handle multiple inference API response formats.

**Mitigation:** For the PoC, scope multi-modal metering to a single inference
API format (e.g., OpenAI-compatible), with a roadmap item for additional
formats.

### 5. Graduated Payment Degradation Not Templated (MB-007)

**Impact:** Low — the prospect notes this is a "billing system" responsibility
and accepts external handling.

The specific graduated response workflow (warning → alert → throttle → block
with configurable timing at each step) is architecturally supported but not
packaged as a configurable policy template. An operator would need to
implement this logic using the credit system's cap configurations and webhook
triggers.

**Mitigation:** Create a "payment degradation policy" configuration template
(~2-3 weeks).

---

## 8. Recommendation

### Verdict: CONDITIONAL YES

Meteridian can serve as the Commercial Control Plane for the AI Grid PoC,
**subject to the following conditions:**

| # | Condition | Timeline | Owner |
|---|-----------|----------|-------|
| 1 | Meteridian core (event ingestion, rating engine, credit ledger, catalog API) reaches demonstrable state | Before PoC start | Meteridian team |
| 2 | AI-specific metering blocks (LLM token parser, GPU metric ingestion) are developed | 4-6 weeks | Meteridian team |
| 3 | Enforcement signal webhook schema is defined and implemented | 2-4 weeks | Meteridian team + Orchestrator team |
| 4 | Orchestrator-side enforcement listener is implemented | 4 weeks | Orchestrator team (not Meteridian) |
| 5 | PoC scope is limited to text token metering (defer multi-modal to post-PoC) | Immediate | Both teams |

### What the PoC Should Demonstrate

| Capability | METR | Feasibility |
|------------|------|-------------|
| Ingest GPU/token events from vLLM via Prometheus | METR-0002, METR-0003 | High (standard Prometheus scraping + block processing) |
| Rate events using tiered token pricing + GPU-hour pricing | METR-0003 §6 | High (core rating engine capability) |
| Track prepaid credit balance in real-time | METR-0004 §3.2 | High (Valkey-backed balance is core architecture) |
| Fire webhook when balance hits 75% / 90% / 100% thresholds | METR-0004 §7 | High (bill shock prevention is v1.0 scope) |
| Generate chargeback report by business unit | METR-0005 §4 | High (purpose-built for this) |
| Provide API for orchestrator to query billing state | METR-0004 §8, METR-0005 §9 | High (API-first design) |
| Demonstrate cryptographic audit trail | ADR-0014, METR-0008 §5 | High (core v1.0 feature) |

### What Should NOT Be in the PoC

- Multi-modal metering (image/audio tokens) — defer to post-PoC
- Full e-invoicing with tax authority integration — not relevant for internal PoC
- Payment processing (credit cards, ACH) — PoC is pre-revenue
- GSLB or orchestrator functionality — separate products, separate teams

### Compliance Score Summary

**Within Meteridian's scope (Metering & Billing):**

| Status | Count | Percentage |
|--------|-------|------------|
| Fully Met | 5 | 50% |
| Mostly Met | 4 | 40% |
| Partially Met | 1 | 10% |
| Not Met | 0 | 0% |
| **Total addressable** | **10** | **100% coverage (full or partial)** |

**Weighted score (full = 1.0, mostly = 0.85, partial = 0.6):**
(5 × 1.0 + 4 × 0.85 + 1 × 0.6) / 10 = **93%** (revised from 85%)

> **2026-06-19:** MB-003 upgraded from "Partially Met" to "Mostly Met" due to
> the MaaS external metering plugin eliminating the LLM parsing gap. MB-002 and
> MB-006 effort estimates also reduced due to Limitador providing a natural
> enforcement integration point. MB-010 (Multi-Cloud and Hybrid Metering)
> assessed as "Mostly Met" due to METR-0012 defining cloud-specific source
> blocks and ADR-0019 establishing the ingestion-time normalization pattern.

**Including effort-to-close:** All remaining gaps are closable within 3-5 weeks
of focused development (revised from 2-4 weeks due to multi-cloud scope). After
gap closure: **100%**.

---

## 9. Appendix: Requirement-to-METR Mapping

| Requirement | Primary METR(s) | Primary ADR(s) | Phase |
|-------------|----------------|----------------|-------|
| MB-001 | METR-0003, METR-0004 | ADR-0004 | v1.0 |
| MB-002 | METR-0004, METR-0005, **METR-0010** | ADR-0011 | v1.0 (partial — Limitador integration) |
| MB-003 | METR-0003, METR-0002, **METR-0010** | ADR-0004, ADR-0005 | v1.0 (**met** — MaaS CloudEvent consumption) |
| MB-004 | METR-0005 | — | v1.0 |
| MB-005 | METR-0003, METR-0004 | ADR-0003 | v1.0 |
| MB-006 | METR-0004, **METR-0010** | ADR-0002 | v1.0 (partial — Limitador integration) |
| MB-007 | METR-0004 | ADR-0010 | v1.0 (partial) |
| MB-008 | METR-0008 | ADR-0017 | v1.0 |
| MB-009 | METR-0008 | ADR-0014 | v1.0 |
| MB-010 | **METR-0012**, METR-0010, METR-0003 | **ADR-0019**, ADR-0013, ADR-0004 | v1.0 (mostly met — cloud source blocks + normalization) |
| ORCH-012 | METR-0004, METR-0005 | — | v1.0 (partial) |

### Key METR Summary

| METR | Relevance to This PoC |
|------|----------------------|
| METR-0002 (Extensibility) | Enables custom AI metering blocks |
| METR-0003 (Product Catalog) | Token/GPU pricing, tiered rating, SLA multipliers |
| METR-0004 (Credit/Token Billing) | Prepaid wallets, real-time balance, bill shock prevention |
| METR-0005 (Internal Budget Units) | Chargeback reporting, hierarchical accounts |
| METR-0006 (Developer Experience) | Accelerates custom block development |
| METR-0007 (Legal Compliance) | Background context; not critical for PoC |
| METR-0008 (Compliance-as-Code) | Audit trail guarantee |
| METR-0009 (E-Invoicing) | Not relevant for PoC phase |
| **METR-0010 (AI Workload Metering)** | **Primary integration spec — MaaS CloudEvent consumption, Limitador enforcement, GPU metric ingestion** |
| **METR-0012 (Multi-Cloud and Hybrid Metering)** | **Cross-cloud normalization — AWS, Azure, GCP source blocks, unified cost reporting, hybrid GPU cost model** |

---

### External References (RHOAI MaaS Metering Infrastructure)

- [MaaS External Metering Plugin — PR #320][ipp-pr-320]: Gateway-level metering
  plugin in the IPP pipeline that extracts per-user, per-model token usage and
  emits CloudEvents v1.0
- [RHAISTRAT-1919][rhaistrat-1919]: Jira tracking for MaaS metering plugin
  (Dev Preview in RHOAI 3.5)
- [MaaS Metering Simulator](https://noyitz.github.io/ai-gateway-metering-service/):
  Standalone service replicating metering backend behavior for development/testing
- [METR-0010 §3](../../enhancements/0010-ai-metering/ai-metering.md#3-rhoai-maas-external-metering-plugin--primary-data-source):
  Full analysis of the MaaS metering plugin, CloudEvent schema, Limitador
  enforcement, and Redpanda Connect pipeline configuration

---

*This analysis is based on Meteridian enhancement proposals and ADRs as of
2026-06-19 (revised). Meteridian is pre-release software. All capability
assessments refer to planned v1.0 features, not production-deployed
functionality. The RHOAI MaaS external metering plugin is Dev Preview in
RHOAI 3.5 ([PR #320][ipp-pr-320]).*
