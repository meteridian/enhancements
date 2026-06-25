# METR-0010: AI Workload Metering

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0002 (Platform Extensibility), METR-0003 (Product Catalog)
- **Related:** METR-0004 (Credit and Token Billing), METR-0005 (Internal Budget Units), ADR-0004 (CloudEvents), ADR-0013 (Two-Layer Data Architecture)
- **Motivated by:** [MB-003 Gap Analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [RHOAI MaaS External Metering Plugin — Primary Data Source](#3-rhoai-maas-external-metering-plugin--primary-data-source)
4. [Gap Analysis: What Exists vs What's Needed](#4-gap-analysis-what-exists-vs-whats-needed)
5. [AI Metering Resource Types](#5-ai-metering-resource-types)
6. [Implementation Approach — Alternative Integration Paths](#6-implementation-approach--alternative-integration-paths-for-non-rhoai-deployments)
7. [Integration Patterns](#7-integration-patterns)
8. [Product Catalog Templates](#8-product-catalog-templates)
9. [PoC Scope — AI Grid (July 31, 2026)](#9-poc-scope--ai-grid-july-31-2026)
10. [Timeline and Effort Estimate](#10-timeline-and-effort-estimate)
11. [Hardware-Aware Metering (Moat Deepening)](#11-hardware-aware-metering-moat-deepening)
12. [Open Questions](#12-open-questions)
13. [Related Documents](#13-related-documents)

---

## 1. Summary

This enhancement defines **AI workload metering** capabilities for Meteridian:
canonical resource types for LLM tokens, GPU compute, multi-modal inference,
and queue/latency dimensions. It specifies how each metering dimension is
collected, processed, and rated.

**The central finding is that the gap is dramatically smaller than initially
estimated — and has been further reduced by RHOAI's own metering
infrastructure.** Red Hat OpenShift AI's MaaS (Models-as-a-Service) layer now
includes an **external metering plugin** ([PR #320][ipp-pr-320],
[RHAISTRAT-1919][rhaistrat-1919]) that runs in the AI Inference Gateway's
IPP (Intelligent Payload Processing) pipeline. This plugin:

- **Extracts per-user, per-model token usage at the gateway level** — no
  LLM response parsing needed on Meteridian's side
- **Emits CloudEvents v1.0** — the same event format Meteridian already
  uses as its canonical event protocol (ADR-0004)
- **Includes per-user and per-model attribution** — via Authorino
  (AuthN/AuthZ) and Limitador (rate limiting), both part of Red Hat
  Connectivity Link

This means the primary RHOAI integration path for LLM token metering is
**event consumption and configuration, not new feature development**:

1. **Consume CloudEvents from the MaaS metering plugin** (~40% of work) —
   Meteridian's Redpanda Connect pipeline receives pre-structured CloudEvents
   with token counts already extracted. A simple `http_server` input or Kafka
   consumer with minimal Bloblang transformation maps the MaaS event schema
   to Meteridian's canonical resource types.

2. **Product Catalog resource type definitions** (~30% of the work) —
   Canonical resource type entries for AI metering dimensions (tokens, GPU
   hours, VRAM, latency). These are data definitions in the catalog, not code.

3. **Product Catalog rate plan templates** (~20% of the work) — Pre-built
   pricing templates for common AI billing models (per-token, GPU reservation,
   committed spend). These compose existing charge model primitives from
   METR-0003.

4. **GPU metric ingestion from Prometheus** (~10% of the work) — Redpanda
   Connect configuration to query the OpenShift Thanos Querier for DCGM
   GPU utilization metrics (unchanged from original approach).

No new Meteridian processing blocks are required for the PoC scope. The
`llm-token-metering` block described in the MB-003 gap analysis is now
**entirely replaced** by consuming CloudEvents from the MaaS external metering
plugin. The `gpu-hardware-metering` block remains a Redpanda Connect pipeline
configuration template using existing processors.

[ipp-pr-320]: https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320
[rhaistrat-1919]: https://redhat.atlassian.net/browse/RHAISTRAT-1919

---

## 2. Motivation

### 2.1 The AI Grid PoC

The AI Grid PoC (see [gap analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md))
requires multi-dimensional AI metering for a telco AI inference platform. The
prospect's orchestrator layer is **Red Hat OpenShift AI (RHOAI)** — the
Kubernetes-native AI/ML platform that provides model serving, GPU scheduling,
and multi-tenant governance. RHOAI is the primary integration target for
Meteridian's AI metering capabilities.

The prospect needs to meter:

- **LLM token consumption** — prompt, completion, cached, and reasoning tokens
  from OpenAI-compatible inference APIs served by RHOAI's vLLM ServingRuntime
- **GPU hardware utilization** — H100/B200 GPU-hours, VRAM GB-seconds, queue
  wait times from NVIDIA DCGM Exporter (deployed via the GPU Operator on
  OpenShift)
- **Multi-modal dimensions** — image tokens, audio segments from multi-modal
  inference APIs
- **Per-tenant attribution** — usage mapped to RHOAI MaaS subscriptions,
  namespaces, and API keys for chargeback

### 2.2 Market Context

Every major cloud and AI platform now charges for AI workloads with
dimension-specific pricing. **Red Hat OpenShift AI is the primary platform
for this PoC** — its MaaS (Models-as-a-Service) governance layer already
tracks token quotas, making it the natural metering data source. The table
below shows how RHOAI's metering dimensions compare to public cloud providers:

| Provider | Pricing Dimensions |
|----------|-------------------|
| **Red Hat OpenShift AI (RHOAI)** | **Token quotas per MaaS subscription, GPU-hours per namespace, vLLM Prometheus counters (prompt/generation tokens), DCGM GPU utilization** |
| OpenAI | Input tokens, output tokens, cached tokens (50% discount), reasoning tokens |
| Anthropic | Input tokens, output tokens, prompt caching (90% discount) |
| Google Vertex AI | Input characters, output characters, image processing, video processing |
| AWS Bedrock | Input tokens, output tokens, model-specific multipliers |
| Azure OpenAI | Per-1000 tokens, prompt/completion split, provisioned throughput units (PTUs) |
| CoreWeave | GPU-hours (per SKU), VRAM allocation, network egress |
| Lambda Labs | GPU-hours (per SKU), storage GB-months |

Meteridian must support these pricing patterns as first-class capabilities to
serve AI infrastructure customers. For the AI Grid PoC, RHOAI integration is
the primary deliverable; cloud provider integrations are secondary.

### 2.3 Why This Is Mostly Configuration, Not Code

Meteridian's two-layer architecture (ADR-0013) was designed for exactly this
kind of extensibility:

- **Layer 1 (Redpanda Connect)** handles data collection — scraping Prometheus
  endpoints, intercepting HTTP responses, parsing JSON payloads, and emitting
  CloudEvents. All of these are existing Redpanda Connect capabilities.

- **Layer 2 (Block Runtime)** handles billing logic — rating, aggregation,
  credit deduction, invoicing. These are existing Meteridian blocks (METR-0002)
  configured via the Product Catalog (METR-0003).

The AI metering gap is not an architecture gap. It is a **content gap**: no one
has yet written the specific Redpanda Connect configurations and Product Catalog
entries for AI workloads. This METR closes that gap.

---

## 3. RHOAI MaaS External Metering Plugin — Primary Data Source

> **This section describes the most significant finding for this enhancement.**
> RHOAI's MaaS layer already has a gateway-level metering plugin that extracts
> per-user, per-model token usage and emits CloudEvents. This eliminates the
> need for Meteridian to build LLM response parsers and dramatically reduces
> the integration scope from "new feature development" to "event consumption
> and configuration."

### 3.1 Background: Red Hat Connectivity Link and the IPP Pipeline

RHOAI's MaaS (Models-as-a-Service) governance layer is built on **Red Hat
Connectivity Link** — a Kubernetes-native framework that integrates gateway,
auth, and rate limiting into a unified stack. The key components:

| Component | Role | Relevance to Meteridian |
|-----------|------|------------------------|
| **Envoy Gateway** | Data plane proxy for inference traffic | Routes LLM requests; the metering plugin runs in this request/response pipeline |
| **Authorino** | AuthN/AuthZ policy engine (Kuadrant) | Provides per-user identity (API key, OIDC, mTLS) that the metering plugin uses for `user` and `group` attribution in CloudEvents |
| **Limitador** | Rate limiting service (Rust, Kuadrant) | Enforces token-per-minute/hour quotas. **Natural integration point for Meteridian's enforcement signals** — Meteridian can update Limitador quotas based on billing state (MB-002/MB-006) |
| **IPP (Intelligent Payload Processing)** | Plugin chain in the AI Inference Gateway | The extensible pipeline where the external metering plugin runs. Processes requests before inference and responses after inference |

**Authorino** is a lightweight Envoy external authorization server managed
entirely via Kubernetes CRDs (`AuthPolicy`). It supports JWT/OIDC, API keys,
mTLS, OPA/Rego policies, and Kubernetes RBAC — all enforced at the gateway
level without application code changes. In the MaaS context, Authorino
validates API keys and maps them to groups/subscriptions, providing the
identity context that appears in metering CloudEvents.

**Limitador** is a generic rate limiter written in Rust, also managed via
Kubernetes CRDs (`RateLimitPolicy`). It tracks counters in memory, disk, or
Redis and integrates with Envoy via a Wasm filter. Limitador already enforces
`TokenRateLimitPolicy` (token-per-minute limits extracted from LLM responses).
Meteridian can leverage Limitador as an **enforcement actuator**: when
Meteridian detects budget exhaustion, it can update Limitador's rate limit
counters or configuration to throttle or block the tenant's inference
requests.

### 3.2 The External Metering Plugin (PR #320)

The MaaS external metering plugin ([PR #320][ipp-pr-320], Dev Preview in
RHOAI 3.5) runs in the IPP pipeline and captures per-request token usage
at the gateway level:

**Request phase:**
1. Extracts user identity from `x-maas-username` / `x-maas-subscription`
   headers (set by Authorino)
2. Checks user balance against the metering backend (configurable fail-open)
3. Injects `stream_options.include_usage=true` for streaming responses
   (ensures token counts appear in the final SSE chunk)

**Response phase:**
1. Extracts token usage from the inference response (supports OpenAI and
   Anthropic response formats)
2. Normalizes token counts across provider schemas (see §3.3)
3. Builds a CloudEvents v1.0 envelope with full token breakdown
4. POSTs the event to the configured metering backend

**Plugin configuration** (in RHOAI Helm values):

```yaml
upstreamBbr:
  bbr:
    plugins:
      - type: external-metering
        name: external-metering
        json:
          meteringURL: "http://metering-service.metering.svc:8080"
          failOpen: true
          timeoutSeconds: 5
          featureKey: "tokens"
          source: "maas-gateway"
```

### 3.3 CloudEvent Schema Emitted by the Plugin

The metering plugin emits CloudEvents v1.0 with the following schema. This
is the **actual event format** from the PR and the
[metering simulator][metering-simulator]:

```json
{
  "specversion": "1.0",
  "id": "evt-a1b2c3d4",
  "source": "maas-gateway",
  "type": "inference.tokens.used",
  "subject": "alice",
  "time": "2026-06-10T12:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "user": "alice",
    "group": "engineering",
    "provider": "anthropic",
    "model": "claude-opus-4-8",
    "prompt_tokens": 150,
    "completion_tokens": 80,
    "total_tokens": 230,
    "cached_input_tokens": 12000,
    "cache_creation_tokens": 100,
    "reasoning_tokens": 0,
    "duration_ms": 1200
  }
}
```

**Token type mapping across providers:**

| Token Type | CloudEvent Field | OpenAI Source | Anthropic Source | Pricing Impact |
|-----------|-----------------|--------------|-----------------|---------------|
| Input (new) | `prompt_tokens` | `usage.prompt_tokens` | `usage.input_tokens` | Full input rate |
| Output | `completion_tokens` | `usage.completion_tokens` | `usage.output_tokens` | Full output rate |
| Cached read | `cached_input_tokens` | `usage.prompt_tokens_details.cached_tokens` | `usage.cache_read_input_tokens` | ~10x cheaper |
| Cache write | `cache_creation_tokens` | — | `usage.cache_creation_input_tokens` | ~25% more |
| Reasoning | `reasoning_tokens` | `usage.completion_tokens_details.reasoning_tokens` | — | Output rate |
| Total | `total_tokens` | `usage.total_tokens` | Computed sum | — |

**Key attributes for Meteridian billing:**
- `user` / `subject` — per-user attribution (set by Authorino identity)
- `group` — organizational unit for chargeback (METR-0005)
- `model` — per-model pricing (METR-0003 rate plan metadata)
- `provider` — provider-specific rate differentiation
- `duration_ms` — request latency for SLA-tier multipliers

### 3.4 Why This Dramatically Reduces the Integration Gap

The original METR-0010 and MB-003 gap analysis assumed Meteridian would need
to build:

| Original Assumption | Reality with MaaS Metering Plugin |
|--------------------|----------------------------------|
| LLM response parser block — extract `usage` from OpenAI/Anthropic responses | **Eliminated.** The gateway plugin already does this. Meteridian receives pre-extracted token counts. |
| Sidecar proxy for per-request interception | **Eliminated.** The gateway intercepts all requests; no per-pod sidecar needed for token metering. |
| Streaming SSE response handling | **Eliminated.** The plugin handles `stream_options.include_usage` injection and final-chunk parsing. |
| Per-user identity correlation | **Eliminated.** Authorino provides user/group identity, embedded in the CloudEvent by the plugin. |
| Multi-provider token format normalization | **Eliminated.** The plugin normalizes OpenAI and Anthropic formats into a single schema. |

**What Meteridian still needs to build:**

| Component | Type | Effort |
|-----------|------|--------|
| Redpanda Connect pipeline to consume MaaS CloudEvents | Configuration (YAML) | 1-2 days |
| Bloblang mapping from MaaS event schema to Meteridian resource types | Configuration (YAML) | 1 day |
| Product Catalog resource type definitions for AI dimensions | Data entry | 2 days |
| Product Catalog rate plan templates | Data entry | 2 days |
| GPU metric ingestion from Prometheus/Thanos | Configuration (YAML) | 3-4 days |

**Total estimated PoC effort: ~18 person-days (~4 weeks, 1 engineer)** — down
from the original 29 person-days estimate. See §10.1 for the detailed
week-by-week breakdown.

### 3.5 Redpanda Connect Pipeline for MaaS CloudEvent Consumption

Consuming CloudEvents from the MaaS metering plugin requires a trivial
Redpanda Connect pipeline. The plugin can POST events directly to
Meteridian's HTTP endpoint, or events can flow through Kafka/Redpanda:

**Option A: Direct HTTP consumption (simplest for PoC)**

```yaml
# maas-token-metering.yaml
# Receives CloudEvents from the MaaS external metering plugin

input:
  http_server:
    path: /api/v1/maas-events
    allowed_verbs:
      - POST

pipeline:
  processors:
    # Map MaaS metering plugin CloudEvent fields to Meteridian resource types
    - mapping: |
        let data = this.data

        root = {
          "specversion": "1.0",
          "id": this.id,
          "type": "ai.llm.tokens.consumed",
          "source": this.source,
          "time": this.time,
          "subject": $data.user,
          "datacontenttype": "application/json",
          "data": {
            "prompt_tokens": $data.prompt_tokens,
            "completion_tokens": $data.completion_tokens,
            "total_tokens": $data.total_tokens,
            "cached_tokens": $data.cached_input_tokens.or(0),
            "cache_creation_tokens": $data.cache_creation_tokens.or(0),
            "reasoning_tokens": $data.reasoning_tokens.or(0),
            "model_name": $data.model,
            "provider": $data.provider,
            "tenant_id": $data.group,
            "user_id": $data.user,
            "duration_ms": $data.duration_ms.or(0),
          },
        }

output:
  http_client:
    url: "${METERIDIAN_EVENTS_URL}/api/v1/events"
    verb: POST
    headers:
      Content-Type: "application/cloudevents+json"
      Authorization: "Bearer ${METERIDIAN_TOKEN}"
```

**Option B: Kafka/Redpanda consumer (production path)**

```yaml
# maas-token-metering-kafka.yaml
# Consumes MaaS metering events from a Kafka/Redpanda topic

input:
  kafka:
    addresses:
      - "${KAFKA_BROKERS}"
    topics:
      - "maas.metering.tokens"
    consumer_group: "meteridian-token-billing"

pipeline:
  processors:
    # Same mapping as Option A
    - mapping: |
        # ... (identical Bloblang mapping)
        root = this
        root.type = "ai.llm.tokens.consumed"

output:
  http_client:
    url: "${METERIDIAN_EVENTS_URL}/api/v1/events"
    verb: POST
    headers:
      Content-Type: "application/cloudevents+json"
```

The mapping is minimal because the MaaS plugin already emits well-structured
CloudEvents. The Bloblang transformation renames fields to match Meteridian's
canonical resource type naming and adds any billing-specific metadata.

### 3.6 Enforcement via Limitador (MB-002, MB-006)

Limitador provides a natural **enforcement actuator** for Meteridian's
billing signals. The integration flow:

```
  Meteridian                    Limitador                   MaaS Gateway
  ┌──────────┐                 ┌──────────┐                ┌──────────┐
  │ Balance  │                 │ Rate     │                │ Inference│
  │ Monitor  │                 │ Limiter  │                │ Requests │
  │          │                 │ (Rust)   │                │          │
  │ Detects  │── webhook/API──►│ Update   │── RLS deny ──►│ Request  │
  │ budget   │  (patch quota)  │ counters │                │ blocked  │
  │ exhaust  │                 │ or       │                │          │
  │          │                 │ config   │                │          │
  └──────────┘                 └──────────┘                └──────────┘
```

**Enforcement options via Limitador:**

| Enforcement Action | Limitador Mechanism | Meteridian Trigger |
|-------------------|--------------------|--------------------|
| Throttle requests | Reduce `RateLimitPolicy` max tokens/minute for the group | Balance at 75% threshold |
| Block new requests | Set rate limit to 0 or update Limitador counters to exceed max | Balance at 100% (exhausted) |
| Model tier downgrade | Apply different `RateLimitPolicy` per model class | Exceeded premium-tier budget |
| Burst penalty | Apply multiplied rate limits during burst period | Overage detected above committed spend |

This is more elegant than the original approach (patching `MaaSSubscription`
CRs via a Kubernetes operator) because Limitador operates at the data-plane
level with sub-millisecond enforcement latency. The metering plugin's
request-phase balance check already queries a metering backend before
allowing inference — Meteridian can serve as that metering backend or
synchronize state with it.

### 3.7 Testing with the MaaS Metering Simulator

The RHOAI team provides a [metering simulator][metering-simulator] for
development and testing. This standalone service replicates metering backend
behavior and provides:

- **Token usage ingestion** — `POST /api/v1/events` accepts CloudEvents v1.0
- **Balance checks + circuit breaker** — `GET /api/v1/customers/{id}/entitlements/{key}/value`
  with configurable fail-open behavior
- **Built-in usage dashboard** — per-user, per-model token breakdown with
  cache-aware cost calculation

The simulator is compatible with OpenMeter, Lago, Azure Event Grid, Amazon
EventBridge, and any CloudEvents-compatible backend — including Meteridian.

For PoC development, Meteridian can be tested against this simulator before
connecting to a live RHOAI cluster:

```bash
# Start the metering simulator
git clone https://github.com/noyitz/ai-gateway-metering-service
cd ai-gateway-metering-service
docker compose up

# Configure the MaaS metering plugin to point at the simulator
# (or configure Meteridian's Redpanda Connect to consume from it)
```

[metering-simulator]: https://noyitz.github.io/ai-gateway-metering-service/

---

## 4. Gap Analysis: What Exists vs What's Needed

### 4.1 Redpanda Connect Capabilities (Layer 1)

The following Redpanda Connect processors and inputs are directly applicable to
AI metering. All are existing, production-ready components:

| Capability | Redpanda Connect Component | AI Metering Use |
|-----------|---------------------------|----------------|
| Scheduled HTTP scraping | `generate` input (cron/interval) + `http` processor | Poll Prometheus `/metrics` endpoints on vLLM and DCGM Exporter |
| JSON field extraction | `mapping` processor (Bloblang) | Extract `usage.prompt_tokens`, `usage.completion_tokens`, etc. from OpenAI-compatible API responses |
| HTTP request/response interception | `http_server` input + `http_client` output | Act as reverse proxy to capture LLM API response metadata |
| CloudEvents formatting | `mapping` processor (Bloblang) | Transform extracted metrics into CloudEvents envelope |
| Prometheus text format parsing | `mapping` processor + Bloblang string functions | Parse Prometheus exposition format into structured fields |
| Webhook receiver | `http_server` input | Receive usage callbacks from LLM providers |
| Rate limiting | `rate_limit` resource | Control scrape frequency |
| Schema validation | `json_schema` processor | Validate CloudEvents output against spec |
| Batching | `batching` policy | Aggregate individual events into batches for efficient downstream processing |

### 4.2 What Does NOT Exist in Redpanda Connect

| Capability | Status | Workaround |
|-----------|--------|-----------|
| Native Prometheus scrape input | Does not exist as a dedicated input | `generate` + `http` processor achieves the same result; parse the text exposition format with Bloblang |
| Prometheus exposition format parser | No built-in parser for `metric_name{labels} value` format | Bloblang `re_find_all_object` can parse this, or use a `branch` processor calling a small utility script. Alternatively, query a Prometheus server's HTTP API (`/api/v1/query`) which returns JSON |
| OpenAI response interceptor | No purpose-built LLM proxy | Use `http_server` as a reverse proxy, or deploy as a sidecar that receives forwarded response headers/bodies |

**Key insight:** The Prometheus scraping gap is elegantly solved by querying the
**Prometheus HTTP API** (`/api/v1/query` or `/api/v1/query_range`) instead of
scraping the raw exposition format. The Prometheus API returns structured JSON
that Bloblang can parse trivially. In any production Kubernetes deployment,
Prometheus is already scraping vLLM and DCGM Exporter; Redpanda Connect just
needs to query the aggregated data.

### 4.3 What's Needed from Meteridian (Layer 2)

| Component | Type | Status |
|----------|------|--------|
| AI resource type definitions | Product Catalog data | New — defined in this METR (§5) |
| AI rate plan templates | Product Catalog data | New — defined in this METR (§8) |
| AI metering pipeline templates | Redpanda Connect YAML | New — defined in this METR (§6) |
| Tiered rater block | Meteridian processing block | Exists — METR-0003 charge models |
| Credit consumption block | Meteridian processing block | Exists — METR-0004 |
| TimescaleDB writer | Meteridian sink block | Exists — METR-0002 |

---

## 5. AI Metering Resource Types

The following resource types are defined for the Product Catalog (METR-0003 §4).
Each resource type has a canonical name, unit, and description.

### 5.1 LLM Token Dimensions

| Resource Type | Unit | Description | Source |
|--------------|------|-------------|--------|
| `ai.llm.tokens.prompt` | `tokens` | Input/prompt tokens consumed | LLM API `usage.prompt_tokens` |
| `ai.llm.tokens.completion` | `tokens` | Output/completion tokens generated | LLM API `usage.completion_tokens` |
| `ai.llm.tokens.cached` | `tokens` | Prompt tokens served from cache (eligible for discount) | LLM API `usage.prompt_tokens_details.cached_tokens` |
| `ai.llm.tokens.reasoning` | `tokens` | Internal reasoning tokens (chain-of-thought models) | LLM API `usage.completion_tokens_details.reasoning_tokens` |
| `ai.llm.tokens.total` | `tokens` | Total tokens (prompt + completion) | LLM API `usage.total_tokens` |

### 5.2 GPU Compute Dimensions

| Resource Type | Unit | Description | Source |
|--------------|------|-------------|--------|
| `ai.gpu.hours` | `gpu-hours` | GPU time consumed, per SKU (H100, B200, A100, etc.) | Derived from DCGM `DCGM_FI_DEV_GPU_UTIL` × wall-clock time |
| `ai.gpu.vram.gb_seconds` | `GB-seconds` | VRAM consumed over time (framebuffer used × duration) | Derived from DCGM `DCGM_FI_DEV_FB_USED` × scrape interval |
| `ai.gpu.utilization` | `percent` | GPU utilization percentage (for SLA-tier multipliers) | DCGM `DCGM_FI_DEV_GPU_UTIL` |
| `ai.gpu.memory.utilization` | `percent` | GPU memory copy utilization | DCGM `DCGM_FI_DEV_MEM_COPY_UTIL` |

### 5.3 Queue and Latency Dimensions

| Resource Type | Unit | Description | Source |
|--------------|------|-------------|--------|
| `ai.inference.queue.wait_seconds` | `seconds` | Time request waited in queue before processing | vLLM `vllm:num_requests_waiting` (gauge) + request timestamps |
| `ai.inference.latency.ttft_seconds` | `seconds` | Time to first token (TTFT) | vLLM `vllm:time_to_first_token_seconds` |
| `ai.inference.latency.e2e_seconds` | `seconds` | End-to-end request latency | vLLM `vllm:e2e_request_latency_seconds` |
| `ai.inference.latency.itl_seconds` | `seconds` | Inter-token latency (decode phase) | vLLM `vllm:inter_token_latency_seconds` |

### 5.4 Multi-Modal Dimensions

| Resource Type | Unit | Description | Source |
|--------------|------|-------------|--------|
| `ai.multimodal.image.tokens` | `tokens` | Image processing tokens | LLM API `usage.prompt_tokens_details.audio_tokens` or model-specific fields |
| `ai.multimodal.audio.seconds` | `seconds` | Audio input/output duration | LLM API `usage.completion_tokens_details.audio_tokens` (converted from tokens to seconds at model-specific rate) |
| `ai.multimodal.audio.tokens` | `tokens` | Audio tokens (input or output) | LLM API audio token fields |

### 5.5 Resource Type Metadata

Each resource type carries metadata for rating context:

```yaml
resource_type: ai.llm.tokens.prompt
unit: tokens
metadata:
  model_name: "meta-llama/Llama-3.1-70B-Instruct"
  model_provider: "vllm"
  gpu_sku: "H100-SXM"
  cluster_id: "us-east-gpu-01"
  tenant_id: "org-acme-corp"
  api_key_id: "key-abc123"
```

This metadata enables per-model, per-GPU-SKU, per-tenant, and per-API-key
pricing granularity as required by MB-003 and MB-004.

---

## 6. Implementation Approach — Alternative Integration Paths for Non-RHOAI Deployments

> **Note:** For RHOAI-based deployments, §3 (MaaS External Metering Plugin)
> is the **primary** data source. The patterns below apply to non-RHOAI
> platforms (self-hosted vLLM, standalone Prometheus, third-party LLM
> providers) or as supplementary data sources alongside the MaaS plugin
> (e.g., GPU metrics from Prometheus).

### 6.1 Pattern 1: LLM Token Metering from API Responses

**Data source:** OpenAI-compatible chat completion API response (`/v1/chat/completions`)

**Approach:** Redpanda Connect acts as an HTTP reverse proxy (sidecar pattern)
or receives forwarded response bodies from an API gateway. The `mapping`
processor extracts token counts and emits CloudEvents.

**Redpanda Connect configuration:**

```yaml
# llm-token-metering.yaml
# Receives forwarded LLM API responses and emits token usage CloudEvents

input:
  http_server:
    path: /v1/llm-usage
    allowed_verbs:
      - POST

pipeline:
  processors:
    # Extract token counts from OpenAI-compatible response format
    - mapping: |
        let usage = this.usage
        let model = this.model
        let request_id = this.id

        # Build CloudEvents envelope for prompt tokens
        root = {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.llm.tokens.consumed",
          "source": "vllm/" + $model,
          "time": now(),
          "subject": this.metadata.tenant_id,
          "datacontenttype": "application/json",
          "data": {
            "request_id": $request_id,
            "model_name": $model,
            "prompt_tokens": $usage.prompt_tokens,
            "completion_tokens": $usage.completion_tokens,
            "total_tokens": $usage.total_tokens,
            "cached_tokens": $usage.prompt_tokens_details.cached_tokens.or(0),
            "reasoning_tokens": $usage.completion_tokens_details.reasoning_tokens.or(0),
            "audio_prompt_tokens": $usage.prompt_tokens_details.audio_tokens.or(0),
            "audio_completion_tokens": $usage.completion_tokens_details.audio_tokens.or(0),
            "tenant_id": this.metadata.tenant_id,
            "api_key_id": this.metadata.api_key_id,
          },
        }

    # Validate CloudEvents format
    - json_schema:
        schema_path: "file://./cloudevents.spec.json"

    - catch:
        - log:
            level: ERROR
            message: "CloudEvents validation failed: ${!error()}"
        - mapping: "root = deleted()"

output:
  http_client:
    url: "${METERIDIAN_EVENTS_URL}/api/v1/events"
    verb: POST
    headers:
      Content-Type: "application/cloudevents+json"
      Authorization: "Bearer ${METERIDIAN_TOKEN}"
```

**What's happening here:**

- `http_server` input receives POST requests containing OpenAI-compatible response
  bodies (forwarded by the API gateway or LLM proxy).
- The `mapping` processor uses Bloblang to extract all token dimensions from the
  nested `usage` object, including the newer `prompt_tokens_details` and
  `completion_tokens_details` sub-objects.
- `.or(0)` provides safe defaults for optional fields (`cached_tokens`,
  `reasoning_tokens`, `audio_tokens`).
- Output sends the CloudEvents-formatted event to Meteridian's event ingestion
  endpoint.

**No new code required.** This is pure Redpanda Connect configuration using
existing processors.

### 6.2 Pattern 2: GPU Metric Ingestion from Prometheus

**Data source:** Prometheus HTTP API, querying metrics from DCGM Exporter and
vLLM.

**Approach:** Redpanda Connect's `generate` input triggers periodic HTTP requests
to the Prometheus query API. The JSON response is parsed with Bloblang and
emitted as CloudEvents.

**Redpanda Connect configuration:**

```yaml
# gpu-metrics-collector.yaml
# Polls Prometheus for GPU metrics and emits usage CloudEvents

input:
  generate:
    interval: "60s"
    mapping: "root = {}"

pipeline:
  processors:
    # Query DCGM GPU utilization from Prometheus
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                ${PROMETHEUS_URL}/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL
              verb: GET
              headers:
                Accept: "application/json"
        result_map: "root.gpu_util = this"

    # Query DCGM framebuffer used from Prometheus
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                ${PROMETHEUS_URL}/api/v1/query?query=DCGM_FI_DEV_FB_USED
              verb: GET
              headers:
                Accept: "application/json"
        result_map: "root.fb_used = this"

    # Query vLLM request queue depth from Prometheus
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                ${PROMETHEUS_URL}/api/v1/query?query=vllm:num_requests_waiting
              verb: GET
              headers:
                Accept: "application/json"
        result_map: "root.queue_depth = this"

    # Transform Prometheus JSON responses into CloudEvents
    - mapping: |
        root = this.gpu_util.data.result.map_each(r -> {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.gpu.utilization.sampled",
          "source": "dcgm-exporter/" + r.metric.gpu,
          "time": now(),
          "subject": r.metric.namespace.or("default"),
          "data": {
            "gpu_id": r.metric.gpu,
            "gpu_uuid": r.metric.UUID,
            "gpu_utilization_pct": r.value.index(1).number(),
            "hostname": r.metric.Hostname.or("unknown"),
            "namespace": r.metric.namespace.or("default"),
            "pod": r.metric.pod.or("unknown"),
            "sample_interval_seconds": 60,
          },
        })

    - unarchive:
        format: json_array

output:
  http_client:
    url: "${METERIDIAN_EVENTS_URL}/api/v1/events"
    verb: POST
    headers:
      Content-Type: "application/cloudevents+json"
      Authorization: "Bearer ${METERIDIAN_TOKEN}"
```

**What's happening here:**

- `generate` input fires every 60 seconds (matching a typical Prometheus scrape
  interval).
- Three `branch` processors query the Prometheus HTTP API for GPU utilization,
  VRAM usage, and queue depth. Each query returns JSON.
- The `mapping` processor transforms the Prometheus JSON response format
  (`{"data": {"result": [{"metric": {...}, "value": [timestamp, value]}]}}`)
  into CloudEvents.
- `unarchive` with `json_array` splits the array of CloudEvents into individual
  messages for downstream processing.

**No new code required.** This leverages the Prometheus HTTP API (which returns
JSON) instead of scraping the raw Prometheus text exposition format.

### 6.3 Pattern 3: vLLM Token Counters from Prometheus

**Data source:** vLLM's Prometheus metrics for aggregate token counters.

This pattern complements Pattern 1 (per-request token counts) with aggregate
counters for reconciliation and high-level monitoring.

```yaml
# vllm-token-counters.yaml
# Polls vLLM Prometheus counters for aggregate token metrics

input:
  generate:
    interval: "30s"
    mapping: "root = {}"

pipeline:
  processors:
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                ${PROMETHEUS_URL}/api/v1/query?query=vllm:prompt_tokens_total
              verb: GET
        result_map: "root.prompt_tokens = this"

    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                ${PROMETHEUS_URL}/api/v1/query?query=vllm:generation_tokens_total
              verb: GET
        result_map: "root.generation_tokens = this"

    - mapping: |
        root = this.prompt_tokens.data.result.map_each(r -> {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.llm.tokens.counter",
          "source": "vllm/" + r.metric.model_name,
          "time": now(),
          "subject": "aggregate",
          "data": {
            "model_name": r.metric.model_name,
            "prompt_tokens_total": r.value.index(1).number(),
            "metric_type": "counter",
          },
        })

    - unarchive:
        format: json_array

output:
  http_client:
    url: "${METERIDIAN_EVENTS_URL}/api/v1/events"
    verb: POST
    headers:
      Content-Type: "application/cloudevents+json"
      Authorization: "Bearer ${METERIDIAN_TOKEN}"
```

### 6.4 Derived Metrics (SQL-Based, Product Catalog)

Some metering dimensions are **derived** from raw events via SQL aggregation
(METR-0003 §5, SQL-based metric definitions). These run in TimescaleDB, not in
Redpanda Connect:

| Derived Metric | SQL Logic | Inputs |
|---------------|----------|--------|
| `ai.gpu.hours` | `SUM(gpu_utilization_pct / 100.0 * sample_interval_seconds) / 3600` | `ai.gpu.utilization.sampled` events |
| `ai.gpu.vram.gb_seconds` | `SUM(fb_used_mib / 1024.0 * sample_interval_seconds)` | `ai.gpu.vram.sampled` events |
| `ai.llm.tokens.uncached` | `prompt_tokens - cached_tokens` | `ai.llm.tokens.consumed` events |
| `ai.llm.tokens.billable_output` | `completion_tokens - reasoning_tokens` (if reasoning is included in price) | `ai.llm.tokens.consumed` events |

These are defined as SQL-based metrics in the Product Catalog, not as
processing blocks.

---

## 7. Integration Patterns

Five integration patterns cover the full range of AI metering deployments.
Each pattern uses Redpanda Connect (Layer 1) for data collection and the
Meteridian block runtime (Layer 2) for billing processing. **Pattern 7.5
(RHOAI) is the primary pattern for the AI Grid PoC** and composes
patterns 7.1 and 7.2 with RHOAI-specific infrastructure.

### 7.1 Sidecar / Reverse Proxy Pattern

Best for: **RHOAI vLLM ServingRuntime, self-hosted LLM inference (vLLM, TGI, Ollama)**

```
  Client                  Redpanda Connect          vLLM
  ┌──────┐               ┌──────────────┐          ┌──────┐
  │      │── request ────►│  Proxy       │── req ──►│      │
  │      │               │  (http_server │          │      │
  │      │◄── response ──│   + forward)  │◄── res ──│      │
  │      │               │              │          │      │
  └──────┘               │  Extract     │          └──────┘
                          │  usage from  │
                          │  response    │
                          │      │       │
                          │      ▼       │
                          │  CloudEvent  │
                          │  to          │
                          │  Meteridian  │
                          └──────────────┘
```

Redpanda Connect sits as a sidecar or reverse proxy in front of the LLM
inference service. It forwards requests transparently, captures the response
`usage` object, and emits CloudEvents to Meteridian. The LLM service is
unmodified.

### 7.2 Prometheus Scraper Pattern

Best for: **GPU metrics (DCGM Exporter on OpenShift/RHOAI), aggregate inference metrics (vLLM)**

```
  DCGM Exporter          Prometheus          Redpanda Connect
  ┌──────────┐          ┌──────────┐        ┌──────────────┐
  │ /metrics │◄─ scrape─│          │◄─ query│              │
  │ :9400    │          │  tsdb    │  JSON  │  generate    │
  └──────────┘          │          │────────►│  + http      │
                         │          │        │  + mapping   │
  vLLM                  │          │        │      │       │
  ┌──────────┐          │          │        │      ▼       │
  │ /metrics │◄─ scrape─│          │        │  CloudEvent  │
  │ :8000    │          └──────────┘        │  to          │
  └──────────┘                              │  Meteridian  │
                                            └──────────────┘
```

This pattern queries an existing Prometheus server (which is already scraping
DCGM Exporter and vLLM) via its HTTP API. This is the recommended approach for
GPU metrics because:

1. Prometheus is already deployed in any production Kubernetes cluster.
2. The Prometheus API returns structured JSON, which Bloblang parses trivially.
3. Prometheus handles the complexity of scraping, retries, and staleness.
4. PromQL queries can pre-aggregate metrics (e.g., `rate()`, `avg_over_time()`)
   before Redpanda Connect sees them.

### 7.3 Direct Instrumentation Pattern (SDK)

Best for: **Custom inference services, fine-grained per-request metering**

```
  Inference Service
  ┌────────────────────────────────────┐
  │                                    │
  │  def infer(prompt):                │
  │      result = model.generate(...)  │
  │      meteridian.emit({             │
  │        type: "ai.llm.tokens",      │
  │        data: {                     │
  │          prompt_tokens: ...,       │
  │          completion_tokens: ...,   │
  │        }                           │
  │      })                            │
  │      return result                 │
  │                                    │
  └────────────────────────────────────┘
```

The inference service uses a Meteridian SDK (METR-0006) to emit CloudEvents
directly. This provides the most accurate per-request metering but requires
code changes in the inference service.

### 7.4 Webhook / Callback Pattern

Best for: **Third-party LLM providers (OpenAI, Anthropic, Bedrock)**

```
  LLM Provider           Webhook Endpoint          Meteridian
  ┌──────────┐          ┌──────────────┐          ┌──────────┐
  │ OpenAI   │── POST ─►│ Redpanda     │── event──►│          │
  │ usage    │          │ Connect      │          │  Rating  │
  │ webhook  │          │ http_server  │          │  Engine  │
  └──────────┘          │ + mapping    │          └──────────┘
                         └──────────────┘
```

For providers that offer usage webhooks, Redpanda Connect receives the
callback, normalizes the provider-specific format into the canonical Meteridian
CloudEvents format, and forwards to the billing pipeline.

### 7.5 Red Hat OpenShift AI (RHOAI) Integration Pattern

**Best for: The AI Grid PoC and any RHOAI-based inference platform.**

RHOAI is the **primary integration target** for this enhancement. It provides
a full model serving stack on OpenShift with built-in multi-tenancy, GPU
scheduling, and Prometheus-based observability — all of which Meteridian can
consume for billing-grade metering.

#### 7.5.1 RHOAI Model Serving Stack

RHOAI provides model serving through KServe with multiple serving runtimes:

| Runtime | Use Case | Prometheus Metrics Prefix | API Format |
|---------|----------|--------------------------|------------|
| **vLLM NVIDIA GPU ServingRuntime** | LLM inference (primary) | `vllm:` | OpenAI-compatible (`/v1/chat/completions`) |
| **Caikit-TGIS ServingRuntime** | Caikit-wrapped models | `caikit_`, `tgi_` | Caikit REST/gRPC |
| **llm-d (distributed inference)** | Multi-node LLM serving | `vllm:` (same engine) | OpenAI-compatible |
| **OpenVINO Model Server** | CPU-based inference | `ovms_` | REST/gRPC |

For LLM inference, vLLM on RHOAI exposes the **same Prometheus metrics and
OpenAI-compatible API** as standalone vLLM. This means:

- **Pattern 1 (§6.1)** applies directly — the `llm-token-metering.yaml`
  Redpanda Connect configuration works unmodified against RHOAI's vLLM
  endpoint.
- **Pattern 2 (§6.2)** applies directly — DCGM Exporter runs on OpenShift
  via the NVIDIA GPU Operator, and metrics are scraped into OpenShift's
  built-in Prometheus (User Workload Monitoring).
- **Pattern 3 (§6.3)** applies directly — vLLM's Prometheus token counters
  (`vllm:prompt_tokens_total`, `vllm:generation_tokens_total`) are available
  via the OpenShift Thanos Querier.

#### 7.5.2 RHOAI-Specific Data Sources

RHOAI adds several data sources beyond standalone vLLM:

| Data Source | What It Provides | How to Access |
|-------------|-----------------|---------------|
| **Thanos Querier** | Aggregated Prometheus metrics from all namespaces (vLLM, DCGM, Istio) | `https://thanos-querier.openshift-monitoring.svc.cluster.local:9092/api/v1/query` |
| **MaaS Gateway** (RHOAI 3.4+) | Per-subscription token consumption, API key identity, rate limit state | Gateway response headers, `TokenRateLimitPolicy` status |
| **MaaSSubscription CR** | Token quota definitions per group, priority levels, cost allocation metadata | Kubernetes API (`kubectl get maassubscriptions`) |
| **KServe InferenceService** | Model deployment metadata (model name, runtime, namespace, GPU allocation) | Kubernetes API (`kubectl get inferenceservices`) |
| **ServiceMonitor** | Auto-discovery of vLLM `/metrics` endpoints per InferenceService | OpenShift User Workload Monitoring |

**Key insight:** RHOAI's Thanos Querier provides a single, authenticated
Prometheus API endpoint that aggregates metrics across all namespaces. Redpanda
Connect queries this endpoint instead of scraping individual pod `/metrics`
endpoints. This is simpler, handles pod scaling automatically, and respects
OpenShift RBAC.

#### 7.5.3 RHOAI Integration Architecture

```
  RHOAI Cluster
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  Namespace: ai-team-a                    Namespace: ai-team-b      │
  │  ┌──────────────────────┐               ┌──────────────────────┐   │
  │  │ InferenceService     │               │ InferenceService     │   │
  │  │ (vLLM + H100 GPUs)   │               │ (vLLM + B200 GPUs)   │   │
  │  │ /v1/chat/completions │               │ /v1/chat/completions │   │
  │  │ /metrics             │               │ /metrics             │   │
  │  └──────────┬───────────┘               └──────────┬───────────┘   │
  │             │                                      │               │
  │  ┌──────────┴──────────────────────────────────────┴───────────┐   │
  │  │                    MaaS Gateway (Kuadrant + Envoy)           │   │
  │  │  - API key auth (MaaSAuthPolicy)                            │   │
  │  │  - Token rate limits (TokenRateLimitPolicy)                 │   │
  │  │  - Per-subscription metering                                │   │
  │  └──────────────────────────┬───────────────────────────────────┘   │
  │                             │                                      │
  │  ┌──────────────────────────┴───────────────────────────────────┐   │
  │  │                 OpenShift Monitoring Stack                    │   │
  │  │  Prometheus (User Workload Monitoring)                       │   │
  │  │  ├── vllm:prompt_tokens_total{namespace,model_name}         │   │
  │  │  ├── vllm:generation_tokens_total{namespace,model_name}     │   │
  │  │  ├── DCGM_FI_DEV_GPU_UTIL{namespace,pod,gpu}               │   │
  │  │  └── DCGM_FI_DEV_FB_USED{namespace,pod,gpu}                │   │
  │  │                                                              │   │
  │  │  Thanos Querier ──── single query endpoint ──►               │   │
  │  └──────────────────────────┬───────────────────────────────────┘   │
  │                             │                                      │
  └─────────────────────────────┼──────────────────────────────────────┘
                                │
                     Prometheus API (JSON)
                                │
                                ▼
  ┌────────────────────────────────────────────────────┐
  │              Redpanda Connect (Layer 1)             │
  │                                                    │
  │  ┌──────────────────┐   ┌────────────────────────┐ │
  │  │ LLM Token        │   │ GPU Metrics            │ │
  │  │ Pipeline (§6.1)  │   │ Pipeline (§6.2)        │ │
  │  └────────┬─────────┘   └───────────┬────────────┘ │
  │           └──────────┬──────────────┘              │
  │                CloudEvents                         │
  └──────────────────────┼─────────────────────────────┘
                         │
                         ▼
  ┌────────────────────────────────────────────────────┐
  │              Meteridian (Layer 2)                   │
  └────────────────────────────────────────────────────┘
```

#### 7.5.4 Deployment Model: Sidecar on ServingRuntime vs Prometheus Federation

Two deployment options exist for RHOAI integration:

**Option A: Prometheus Federation (Recommended for PoC)**

Redpanda Connect queries the Thanos Querier for aggregated vLLM and DCGM
metrics. This requires no changes to RHOAI's serving infrastructure:

- **Pros:** Zero-touch on RHOAI side, leverages existing monitoring stack,
  handles pod scaling automatically, single query endpoint for all namespaces.
- **Cons:** Prometheus scrape interval (typically 15-60s) introduces latency;
  per-request granularity requires additional integration (see Option B).
- **Best for:** GPU-hour metering, aggregate token counters, SLA monitoring.

Configuration: Use the GPU metrics pipeline (§6.2) with
`PROMETHEUS_URL=https://thanos-querier.openshift-monitoring.svc.cluster.local:9092`
and add bearer token authentication for OpenShift service account access.

**Option B: Sidecar on ServingRuntime (For Per-Request Token Metering)**

Add a Redpanda Connect sidecar container to the vLLM ServingRuntime definition.
The sidecar intercepts or receives forwarded API responses for per-request
token extraction:

```yaml
# Patch to RHOAI ServingRuntime to add Meteridian sidecar
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-runtime-metered
spec:
  containers:
    - name: kserve-container
      image: quay.io/modh/vllm:latest
      # ... standard vLLM configuration ...
    - name: meteridian-metering
      image: ghcr.io/meteridian/redpanda-connect:latest
      args: ["run", "/etc/meteridian/llm-token-metering.yaml"]
      ports:
        - containerPort: 4195
          name: metering
      volumeMounts:
        - name: metering-config
          mountPath: /etc/meteridian
  volumes:
    - name: metering-config
      configMap:
        name: meteridian-llm-metering
```

- **Pros:** Per-request token granularity, captures full `usage` object
  including `cached_tokens` and `reasoning_tokens`, sub-second metering.
- **Cons:** Requires ServingRuntime modification, adds a container to each
  inference pod, needs network routing configuration.
- **Best for:** Per-request billing, per-API-key metering, per-tenant attribution.

**Recommended approach for PoC:** Use **both** — Prometheus federation (Option A)
for GPU metrics and aggregate token counters, plus the sidecar (Option B) on a
single InferenceService for per-request token metering demonstration.

#### 7.5.5 RHOAI MaaS Integration for Multi-Tenant Metering

RHOAI 3.4+ provides Models-as-a-Service (MaaS) with native multi-tenant
governance. MaaS introduces three CRDs relevant to metering:

| CRD | Metering Relevance |
|-----|-------------------|
| `MaaSSubscription` | Defines per-group token quotas with `metadata` for cost allocation. Meteridian can read subscription metadata to correlate usage with billing accounts. |
| `MaaSAuthPolicy` | Maps API keys to groups. Meteridian can use the API key identity passed through the gateway to attribute usage to specific tenants. |
| `TokenRateLimitPolicy` | Kuadrant-based token rate limiting. The gateway extracts `usage.total_tokens` from responses and enforces limits. Meteridian can consume the same token counts for billing. |

**MaaS gateway metering flow:**

1. Client sends request with API key → MaaS gateway authenticates via
   `MaaSAuthPolicy`, identifies group/subscription.
2. Gateway forwards request to vLLM InferenceService.
3. vLLM returns response with `usage` object (OpenAI-compatible format).
4. Gateway's `TokenRateLimitPolicy` extracts `total_tokens` and enforces
   the subscription's rate limit.
5. **Meteridian integration point:** The same response body (or a forwarded
   copy) is sent to the Redpanda Connect metering pipeline, which extracts
   the full token breakdown and emits CloudEvents with the subscription/group
   identity as metadata.

This means Meteridian provides **billing-grade financial metering** alongside
RHOAI's operational quota enforcement. MaaS handles "can this group make
another request?" (rate limiting); Meteridian handles "how much does this
group owe?" (billing).

#### 7.5.6 RHOAI Enforcement Signal Integration (MB-002, MB-006)

The gap analysis identifies enforcement signal integration as a key gap:
Meteridian emits signals (balance exhausted, threshold breached), and the
orchestrator (RHOAI) must act on them.

RHOAI's MaaS architecture provides a natural enforcement integration point:

| Enforcement Action | RHOAI Mechanism | Meteridian Signal |
|-------------------|-----------------|-------------------|
| **Throttle inference** | Update `MaaSSubscription` token rate limits or `TokenRateLimitPolicy` to reduce allowed tokens/minute | Webhook on 75%/90% threshold breach |
| **Block new requests** | Set `MaaSSubscription` token limit to 0, or remove `MaaSAuthPolicy` for the group | Webhook on 100% balance exhaustion |
| **Downgrade model tier** | Modify `MaaSSubscription` to remove access to premium models (e.g., only allow smaller models) | Webhook with enforcement action `downgrade` |
| **Block GPU provisioning** | Update `ResourceQuota` in the tenant's namespace to set `nvidia.com/gpu: 0` | Webhook on GPU-hour budget exhaustion |

**Implementation approach:** A custom Meteridian enforcement block (or a
Kubernetes operator) listens for Meteridian's webhook events and translates
them into RHOAI API operations (patching `MaaSSubscription`, updating
`ResourceQuota`, etc.). This bridges Meteridian's financial state with
RHOAI's operational enforcement.

---

## 8. Product Catalog Templates

Pre-built rate plans for common AI billing models, using the charge model
primitives defined in METR-0003.

### 8.1 Pay-Per-Token (Input/Output Priced Differently)

The most common LLM pricing model. Input tokens and output tokens have different
rates, with optional discounts for cached tokens.

```yaml
plan:
  name: "LLM Token Billing — Standard"
  charges:
    - name: "Prompt Tokens"
      resource_type: "ai.llm.tokens.prompt"
      model: tiered
      tiers:
        - up_to: 1000000
          unit_price: 0.000003    # $3.00 per 1M tokens
        - up_to: 10000000
          unit_price: 0.0000025  # $2.50 per 1M tokens (volume discount)
        - up_to: null
          unit_price: 0.000002   # $2.00 per 1M tokens
      currency: USD

    - name: "Completion Tokens"
      resource_type: "ai.llm.tokens.completion"
      model: tiered
      tiers:
        - up_to: 1000000
          unit_price: 0.000015   # $15.00 per 1M tokens
        - up_to: null
          unit_price: 0.000012   # $12.00 per 1M tokens
      currency: USD

    - name: "Cached Token Discount"
      resource_type: "ai.llm.tokens.cached"
      model: volume
      unit_price: -0.0000015   # 50% discount on prompt token price
      currency: USD
```

### 8.2 GPU-Hour Reservation with Burst Overage

Reserved GPU capacity at a flat monthly rate, with per-hour overage billing for
burst usage above the reservation.

```yaml
plan:
  name: "GPU Compute — Reserved + Burst"
  charges:
    - name: "H100 Reservation (monthly)"
      resource_type: "ai.gpu.hours"
      model: recurring
      amount: 5000.00           # $5,000/month for reserved block
      included_quantity: 720    # 720 GPU-hours included (1 GPU × 30 days)
      currency: USD
      metadata:
        gpu_sku: "H100-SXM"

    - name: "H100 Burst Overage"
      resource_type: "ai.gpu.hours"
      model: volume
      unit_price: 8.50          # $8.50/GPU-hour above reservation
      applies_after: 720        # Only charges above included quantity
      currency: USD
      metadata:
        gpu_sku: "H100-SXM"

    - name: "VRAM Overage"
      resource_type: "ai.gpu.vram.gb_seconds"
      model: volume
      unit_price: 0.000002      # $0.002 per 1000 GB-seconds
      currency: USD
```

### 8.3 Committed Spend with Drawdown

A committed spend contract (METR-0004 §4) where the customer prepays a credit
balance and draws down against it with AI usage.

```yaml
plan:
  name: "AI Committed Spend — Annual"
  contract:
    type: committed_spend
    commitment: 120000.00       # $120,000 annual commitment
    term_months: 12
    overage_rate: 1.15          # 15% premium on usage above commitment
    credit_grant:
      amount: 120000.00
      source: contract
      rollover: false           # Unused credits expire at term end

  charges:
    - name: "Token Consumption"
      resource_type: "ai.llm.tokens.total"
      model: volume
      unit_price: 0.000008      # Blended rate per token
      deduct_from: credit_balance
      currency: USD

    - name: "GPU Compute"
      resource_type: "ai.gpu.hours"
      model: volume
      unit_price: 7.00          # Committed rate per GPU-hour
      deduct_from: credit_balance
      currency: USD
```

### 8.4 Tiered Pricing by Volume

Progressive volume discounts that reward higher usage.

```yaml
plan:
  name: "LLM Token Billing — Enterprise Volume"
  charges:
    - name: "Total Tokens (Volume Tiered)"
      resource_type: "ai.llm.tokens.total"
      model: staircase           # Each tier applies only to tokens in that range
      tiers:
        - up_to: 10000000       # First 10M tokens
          unit_price: 0.000010
        - up_to: 100000000      # 10M - 100M tokens
          unit_price: 0.000008
        - up_to: 1000000000     # 100M - 1B tokens
          unit_price: 0.000005
        - up_to: null           # Above 1B tokens
          unit_price: 0.000003
      currency: USD
```

---

## 9. PoC Scope — AI Grid (July 31, 2026)

### 9.1 Minimum Viable Set

The AI Grid PoC requires demonstrating metering on a **Red Hat OpenShift AI**
cluster running vLLM with NVIDIA GPUs. The three core capabilities:

| # | Capability | Implementation | RHOAI Component | Effort |
|---|-----------|---------------|-----------------|--------|
| 1 | Text token metering via MaaS CloudEvent consumption | Redpanda Connect config (§3.5) — `http_server` input receives pre-structured CloudEvents from the MaaS external metering plugin with token counts already extracted. Bloblang mapping transforms MaaS fields to Meteridian resource types. No LLM response parsing needed. | MaaS External Metering Plugin ([PR #320][ipp-pr-320]) + MaaS Gateway + Authorino | 2-3 days |
| 2 | GPU metric ingestion from DCGM Exporter (via OpenShift Monitoring) | Redpanda Connect config (§6.2) — `generate` + `http` queries Thanos Querier for DCGM metrics, `mapping` emits CloudEvents | NVIDIA GPU Operator + OpenShift User Workload Monitoring | 2-3 days |
| 3 | Basic tiered token pricing | Product Catalog entries (§8.1) — resource type definitions + tiered charge model | N/A (Meteridian-only) | 2-3 days |
| 4 | Billing-state enforcement via Limitador | Limitador API integration (§3.6) — Meteridian updates rate limits on budget exhaustion to throttle/block inference requests | Limitador (Red Hat Connectivity Link) | 2 days |

### 9.2 PoC Architecture

The PoC runs on an RHOAI cluster with the MaaS governance layer. Meteridian
consumes metering data from two RHOAI sources: the MaaS gateway (per-request
tokens) and the OpenShift Monitoring stack (GPU metrics + aggregate counters).

```
  RHOAI Cluster (OpenShift AI)
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  InferenceService (vLLM)       OpenShift Monitoring                │
  │  ┌───────────────┐            ┌──────────────────────┐             │
  │  │ /v1/chat/     │            │ Thanos Querier       │             │
  │  │ completions   │            │ ├── DCGM metrics     │             │
  │  │ /metrics      │──scraped──►│ ├── vLLM counters    │             │
  │  └───────┬───────┘            │ └── Istio metrics    │             │
  │          │                    └──────────┬───────────┘             │
  │          │                               │                         │
  │  ┌───────┴───────────────┐               │                         │
  │  │ MaaS Gateway          │               │                         │
  │  │ (Kuadrant + Envoy)    │               │                         │
  │  │ - API key auth        │               │                         │
  │  │ - TokenRateLimitPolicy│               │                         │
  │  │ - Subscription ID     │               │                         │
  │  └───────┬───────────────┘               │                         │
  │          │                               │                         │
  └──────────┼───────────────────────────────┼─────────────────────────┘
             │                               │
             │ (response forwarded           │ (Prometheus API
             │  with subscription metadata)  │  query via Thanos)
             ▼                               ▼
  ┌────────────────────────────────────────────────────┐
  │              Redpanda Connect (Layer 1)             │
  │                                                    │
  │  ┌─────────────────┐     ┌───────────────────────┐ │
  │  │ LLM Token       │     │ GPU Metrics           │ │
  │  │ Pipeline (§6.1) │     │ Pipeline (§6.2)       │ │
  │  └────────┬────────┘     └───────────┬───────────┘ │
  │           └───────────┬──────────────┘             │
  │                CloudEvents                         │
  └────────────────────────┼───────────────────────────┘
                           │
                           ▼
  ┌────────────────────────────────────────────────────┐
  │              Meteridian (Layer 2)                   │
  │                                                    │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │
  │  │ Rating   │─►│ Credit   │─►│ TimescaleDB  │     │
  │  │ Engine   │  │ Deduction│  │ Writer       │     │
  │  └──────────┘  └──────────┘  └──────────────┘     │
  │                                                    │
  │  ┌──────────────────────────────────────────────┐  │
  │  │ Product Catalog                              │  │
  │  │ - ai.llm.tokens.prompt (tiered)              │  │
  │  │ - ai.llm.tokens.completion (tiered)          │  │
  │  │ - ai.gpu.hours (volume)                      │  │
  │  └──────────────────────────────────────────────┘  │
  │                                                    │
  │  Enforcement Webhooks ──► RHOAI MaaS (§7.5.6)     │
  └────────────────────────────────────────────────────┘
```

### 9.3 What's Deferred to Post-PoC

| Item | Reason for Deferral |
|------|-------------------|
| Multi-modal metering (image/audio tokens) | Requires multi-modal model deployment; MaaS metering plugin does not yet extract multi-modal tokens |
| SLA-tier multiplier rating rules | Requires latency tier definition agreement with RHOAI/orchestrator team |
| Full RHOAI enforcement integration (§7.5.6) | PoC includes basic Limitador enforcement (§3.6 — throttle/block via rate limit updates). Full enforcement (patching `MaaSSubscription` CRs, `ResourceQuota` updates, model tier downgrades) is deferred to post-PoC |
| Per-API-key billing isolation | Requires identity protocol agreement with RHOAI MaaS gateway; `MaaSAuthPolicy` API key identity must be forwarded to metering pipeline |
| Prometheus text format direct scraping | Querying OpenShift Thanos Querier (Prometheus HTTP API) is simpler and sufficient |
| Caikit/TGIS runtime metering | Caikit exposes different metrics (`caikit_*`); vLLM is the primary runtime for the PoC |

---

## 10. Timeline and Effort Estimate

### 10.1 PoC Delivery (Target: July 31, 2026)

| Week | Deliverable | Effort | Dependencies |
|------|------------|--------|-------------|
| 1 (Jun 23-27) | MaaS CloudEvent consumer pipeline (§3.5) + resource type definitions (§5.1, §5.2) | 3 days | MaaS metering simulator (§3.7), METR-0003 Product Catalog API |
| 1 (Jun 23-27) | GPU metrics Redpanda Connect pipeline (§6.2) via Thanos Querier | 2 days | RHOAI with NVIDIA GPU Operator + DCGM Exporter |
| 2 (Jun 30-Jul 4) | Tiered token pricing rate plan (§8.1) + Limitador enforcement PoC (§3.6) | 3 days | Product Catalog charge model support |
| 2 (Jun 30-Jul 4) | Integration testing on RHOAI cluster with MaaS gateway | 2 days | Full RHOAI stack with MaaS external metering plugin |
| 3 (Jul 7-11) | Credit balance integration (METR-0004) + bill shock thresholds | 3 days | Credit system implementation |
| 3 (Jul 7-11) | End-to-end PoC demonstration on RHOAI + documentation | 2 days | All above |
| 4 (Jul 14-18) | Buffer / bug fixes / demo preparation | 3 days | — |

**Total estimated effort: ~18 person-days (~4 weeks, 1 engineer)**

> **Revised from 29 person-days.** The MaaS external metering plugin (§3)
> eliminates the need to build LLM response parsers, reducing the token
> metering pipeline from ~6 days to ~3 days of CloudEvent consumer
> configuration. Integration testing is also faster because the metering
> simulator (§3.7) enables local development without a full RHOAI cluster.

### 10.2 Post-PoC Roadmap

| Phase | Deliverable | Effort |
|-------|------------|--------|
| v1.0 | RHOAI MaaS enforcement integration — Kubernetes operator that patches `MaaSSubscription` based on Meteridian signals (§7.5.6) | 3-4 weeks |
| v1.0 | RHOAI MaaS token consumption correlation — consume `TokenRateLimitPolicy` usage data for billing reconciliation | 2-3 weeks |
| v1.0 | Multi-modal metering pipeline templates | 1-2 weeks |
| v1.0 | GPU reservation + burst rate plan template (§8.2) | 1 week |
| v1.0 | Committed spend template with AI workloads (§8.3) | 1 week |
| v1.0 | SLA-tier multiplier rating rules | 1-2 weeks |
| v1.0 | Caikit-TGIS runtime metering templates (`caikit_*` / `tgi_*` metrics) | 1-2 weeks |
| v1.x | Direct Prometheus scraper input for Redpanda Connect (optional) | 2-3 weeks (Go plugin) |
| v1.x | RHOAI AI Gateway enforcement block (bi-directional enforcement signals) | 2-4 weeks |

---

## 11. Hardware-Aware Metering (Moat Deepening)

### 11.1 Motivation

No billing platform offers native, protocol-level metering for heterogeneous accelerators. All competitors rely on external sidecars or customer-reported metrics. Meteridian's infrastructure collector architecture (METR-0005) uniquely positions it to meter at the hardware level.

### 11.2 GPU MIG (Multi-Instance GPU) Metering

NVIDIA A100/H100 GPUs can be partitioned into MIG instances. Each MIG slice has:
- Dedicated compute (SMs), memory, and memory bandwidth
- Independent error isolation
- Separate UUID reported by DCGM

Meteridian collectors query NVIDIA DCGM (Data Center GPU Manager) for per-MIG-instance metrics:

```go
type MIGInstanceMetric struct {
    NodeID          string          `json:"node_id"`
    GPUUUID         string          `json:"gpu_uuid"`
    MIGInstanceID   int             `json:"mig_instance_id"`
    Profile         string          `json:"profile"`
    SMUtilization   float64         `json:"sm_utilization_percent"`
    MemoryUsedMB    int64           `json:"memory_used_mb"`
    MemoryTotalMB   int64           `json:"memory_total_mb"`
    EnergyWh        decimal.Decimal `json:"energy_wh"`
    Timestamp       time.Time       `json:"timestamp"`
}
```

Rating by MIG profile (different prices for different slice sizes):
- 1g.5gb (1/7 of A100): $0.30/hr
- 2g.10gb (2/7): $0.60/hr
- 3g.20gb (3/7): $0.90/hr
- 7g.40gb (full): $2.10/hr

### 11.3 FPGA Metering

For FPGA-as-a-Service deployments (Xilinx Alveo, Intel Stratix):

```go
type FPGAMetric struct {
    NodeID          string  `json:"node_id"`
    DeviceID        string  `json:"device_id"`
    BitstreamLoaded string  `json:"bitstream_id"`
    SlicesUsed      int     `json:"slices_used"`
    SlicesTotal     int     `json:"slices_total"`
    PowerWatts      float64 `json:"power_watts"`
    Temperature     float64 `json:"temperature_c"`
}
```

Billing models: per-slot-hour, per-bitstream-load, or hybrid.

### 11.4 SmartNIC / DPU Metering

For NVIDIA BlueField, AMD Pensando, Intel IPU:

Metrics: packets processed, bandwidth offloaded, crypto operations, storage IOPS offloaded.

Enables billing for network acceleration services separately from compute.

### 11.5 Distributed Training Job BOM (Bill of Materials)

Multi-node training jobs span multiple GPUs across multiple nodes. The BOM tracks:

```go
type TrainingJobBOM struct {
    JobID           string              `json:"job_id"`
    Framework       string              `json:"framework"`
    StartTime       time.Time           `json:"start_time"`
    EndTime         *time.Time          `json:"end_time,omitempty"`
    Nodes           []TrainingNode      `json:"nodes"`
    TotalGPUHours   decimal.Decimal     `json:"total_gpu_hours"`
    TotalEnergy     decimal.Decimal     `json:"total_energy_kwh"`
    NetworkTransfer decimal.Decimal     `json:"network_transfer_gb"`
    StorageIO       decimal.Decimal     `json:"storage_io_gb"`
    TotalCost       decimal.Decimal     `json:"total_cost"`
}

type TrainingNode struct {
    NodeID      string `json:"node_id"`
    GPUCount    int    `json:"gpu_count"`
    GPUModel    string `json:"gpu_model"`
    Role        string `json:"role"`
}
```

Integration with K8s: Watch for PyTorchJob, MPIJob, TFJob CRDs to automatically detect distributed training jobs and correlate multi-node GPU usage into a single billable unit.

### 11.6 Protocol-Level Integration

- **NVIDIA DCGM:** Native Go client, poll every 10s (configurable)
- **AMD ROCm SMI:** CLI-based collector for MI300X
- **Intel Level Zero:** For Gaudi/Ponte Vecchio accelerators
- **Kubernetes Device Plugin:** GPU allocation events from kubelet
- **NCCL/RCCL:** Network bandwidth between GPUs (for training BOM)

---

## 12. Open Questions

1. **Prometheus vs direct scrape** — Should we build a native Prometheus
   scraper input for Redpanda Connect, or is querying the Prometheus HTTP API
   sufficient? The HTTP API approach is simpler and leverages existing
   infrastructure, but adds a Prometheus server dependency. For environments
   without Prometheus, a direct scraper would be needed.

2. **Token counting accuracy for streaming responses** — When vLLM returns
   streaming responses (SSE), the `usage` object is only present in the final
   chunk. The sidecar pattern must buffer the response stream or only capture
   the final chunk. This is a Redpanda Connect configuration concern, not a
   code concern.

3. **GPU-hour attribution to tenants** — DCGM Exporter reports GPU metrics
   per-GPU (identified by UUID), not per-tenant. Mapping GPU utilization to
   tenants requires pod-level attribution: correlating DCGM GPU UUIDs with
   Kubernetes pod resource allocations. This may require an additional query
   to the Kubernetes API or kube-state-metrics.

4. **Multi-modal token pricing** — Different providers report multi-modal usage
   in incompatible formats: OpenAI uses `audio_tokens`, Anthropic uses
   character counts, Google uses "billable characters" with model-specific
   multipliers. Should we normalize to a single unit or support
   provider-specific units?

5. **Reconciliation between per-request and aggregate counters** — Pattern 1
   (per-request) and Pattern 3 (aggregate counters from Prometheus) may drift
   due to timing. Should we implement automated reconciliation, or is the
   per-request source sufficient as the source of truth?

6. **RHOAI MaaS subscription integration depth** — RHOAI's MaaS layer
   (3.4+) already tracks per-subscription token consumption via
   `TokenRateLimitPolicy`. Should Meteridian consume MaaS's token count
   data directly (single source of truth for token counts) or maintain an
   independent count via the sidecar/proxy pattern? The former avoids
   double-counting but creates a dependency on MaaS internals; the latter
   is independent but requires reconciliation with MaaS's own accounting.

7. **RHOAI enforcement signal latency requirements** — The gap analysis
   (MB-002, MB-006) calls for enforcement signals from Meteridian to
   RHOAI. What is the acceptable latency for enforcement actions? If
   Meteridian detects budget exhaustion, how quickly must the
   `MaaSSubscription` be updated to block further requests? Sub-second
   enforcement may require a Kubernetes operator watching Meteridian's
   Valkey balance state; webhook-based enforcement adds HTTP round-trip
   latency.

8. **DCGM GPU-to-tenant attribution on RHOAI** — DCGM Exporter reports
   GPU metrics per-GPU UUID. With the NVIDIA GPU Operator's pod metadata
   enrichment API (setting `DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS`),
   metrics include `namespace` and `pod` labels, which map to RHOAI's
   per-namespace tenant model. Is namespace-level attribution sufficient,
   or does the PoC require per-InferenceService or per-API-key GPU
   attribution? The latter would require correlating DCGM pod-level
   metrics with KServe's InferenceService-to-pod mapping.

---

## 13. Related Documents

### Enhancement Proposals

- [METR-0002](../0002-extensibility/extensibility.md) — Platform Extensibility
  and Block Marketplace (block model, pipeline definition)
- [METR-0003](../0003-product-catalog/product-catalog.md) — Product and Service
  Catalog (resource types, charge models, SQL-based metrics)
- [METR-0004](../0004-credit-token-billing/credit-token-billing.md) — Credit,
  Prepaid, and Token Billing (credit grants, balance management)
- [METR-0005](../0005-internal-token-economy/internal-token-economy.md) —
  Internal Budget Units and Chargeback (chargeback reporting)

### Architecture Decision Records

- [ADR-0004](../../docs/adr/0004-cloudevents-event-format.md) — CloudEvents as
  canonical event format
- [ADR-0013](../../docs/adr/0013-two-layer-data-architecture.md) — Two-layer
  data architecture (Redpanda Connect + block runtime)

### Prospect Analysis

- [AI Grid PoC Gap Analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md)
  — MB-003 gap and overall prospect assessment

### External References

#### Red Hat OpenShift AI and Connectivity Link (Primary Integration Target)

- [MaaS External Metering Plugin — PR #320](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320) —
  IPP plugin that captures per-user, per-model token usage at the gateway level
  and emits CloudEvents v1.0 (the primary data source for Meteridian AI metering)
- [MaaS Metering Simulator](https://noyitz.github.io/ai-gateway-metering-service/) —
  Development/testing tool that simulates MaaS metering CloudEvents for backend
  integration testing without requiring a full RHOAI cluster
- [Red Hat Connectivity Link](https://www.redhat.com/en/technologies/cloud-computing/connectivity-link) —
  Unified Kubernetes-native framework for gateway, AuthN/AuthZ (Authorino), and
  rate limiting (Limitador) in RHOAI
- [Red Hat Connectivity Link Developer Hub](https://developers.redhat.com/products/red-hat-connectivity-link) —
  Developer documentation, quick starts, and solution patterns
- [Connectivity Link Solution Pattern](https://www.solutionpatterns.io/soln-pattern-connectivity-link/solution-pattern/index.html) —
  End-to-end solution pattern demonstrating Envoy Gateway + Authorino + Limitador
- [Red Hat Connectivity Link Blog](https://www.redhat.com/en/blog/red-hat-transforms-application-connectivity-hybrid-cloud-red-hat-connectivity-link) —
  Architecture overview and product positioning
- [RHOAI Model Serving Configuration (3.4)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html-single/configuring_your_model-serving_platform/index) —
  ServingRuntime definitions, vLLM configuration, Prometheus metric annotations
- [RHOAI Models-as-a-Service Governance (3.4)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html-single/govern_llm_access_with_models-as-a-service/index) —
  MaaSSubscription, MaaSAuthPolicy, TokenRateLimitPolicy CRDs for multi-tenant
  token quota management
- [RHOAI Model Monitoring (3.2)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.2/html-single/managing_and_monitoring_models/index) —
  User Workload Monitoring, ServiceMonitor configuration, KEDA autoscaling
- [TokenRateLimitPolicy for AI Resource Management](https://developers.redhat.com/articles/2026/02/18/manage-ai-resource-use-tokenratelimitpolicy) —
  Kuadrant-based token rate limiting on OpenShift, extracting `usage.total_tokens`
  from inference responses
- [Autoscaling vLLM on OpenShift AI](https://developers.redhat.com/articles/2025/11/26/autoscaling-vllm-openshift-ai-model-serving) —
  vLLM Prometheus metrics on RHOAI, KEDA ScaledObject configuration
- [NVIDIA GPU Monitoring on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/enable-gpu-monitoring-dashboard.html) —
  DCGM Exporter dashboard, GPU telemetry on OpenShift

#### Inference Engines and GPU Telemetry

- [vLLM Prometheus Metrics](https://docs.vllm.ai/en/latest/usage/metrics/) —
  Full list of vLLM metrics
- [vLLM Metrics Design](https://github.com/vllm-project/vllm/blob/main/docs/design/metrics.md) —
  Metrics architecture, `vllm:prompt_tokens_total`, `vllm:generation_tokens_total`
  counters, histogram definitions
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) — GPU
  telemetry for Prometheus
- [DCGM Exporter Pod Metadata Enrichment](https://github.com/NVIDIA/gpu-operator/pull/2406) —
  GPU Operator API for adding namespace/pod labels to DCGM metrics

#### API Standards and Metering Infrastructure

- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat) —
  Usage object format with `prompt_tokens_details` and `completion_tokens_details`
- [Redpanda Connect Mapping Processor](https://docs.redpanda.com/connect/components/processors/mapping/) —
  Bloblang mapping documentation
- [Redpanda Connect Generate Input](https://docs.redpanda.com/connect/components/inputs/generate/) —
  Scheduled input generation
- [OpenMeter CloudEvents Collector](https://openmeter.io/docs/collectors/how-it-works) —
  Reference implementation of Redpanda Connect for metering (validates that
  this approach is production-proven)
