# METR-0012: Multi-Cloud and Hybrid Metering

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0002 (Platform Extensibility), METR-0003 (Product Catalog), METR-0010 (AI Workload Metering)
- **Related:** METR-0004 (Credit and Token Billing), METR-0005 (Internal Budget Units), ADR-0004 (CloudEvents), ADR-0013 (Two-Layer Data Architecture), ADR-0019 (Multi-Cloud Cost Normalization)
- **Motivated by:** [MB-007 Gap Analysis](../../docs/prospect-analyses/ai-grid-poc-gap-analysis.md)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Multi-Cloud Architecture](#3-multi-cloud-architecture)
4. [Cloud-Specific Source Blocks](#4-cloud-specific-source-blocks)
5. [Unified Cost Normalization Layer](#5-unified-cost-normalization-layer)
6. [Cross-Cloud Aggregation](#6-cross-cloud-aggregation)
7. [Hybrid Deployment Metering](#7-hybrid-deployment-metering)
8. [Currency and Exchange Rate Handling](#8-currency-and-exchange-rate-handling)
9. [Product Catalog Integration](#9-product-catalog-integration)
10. [Implementation Approach](#10-implementation-approach)
11. [PoC Scope — AI Grid](#11-poc-scope--ai-grid)
12. [Timeline and Effort Estimate](#12-timeline-and-effort-estimate)
13. [Open Questions](#13-open-questions)
14. [Related Documents](#14-related-documents)

---

## 1. Summary

This enhancement defines **multi-cloud and hybrid metering** capabilities for
Meteridian: the ability to collect, normalize, and aggregate AI workload costs
across multiple cloud providers (AWS, Azure, GCP) and on-premise infrastructure
into a single unified view. It addresses the prospect requirement for a "single
pane of glass" for AI workload costs regardless of where the model runs.

The design leverages Meteridian's existing architecture:

- **Redpanda Connect source blocks** (METR-0002, ADR-0013) provide the
  integration mechanism — each cloud provider gets a dedicated source block
  configuration that emits CloudEvents in Meteridian's canonical format
- **Product Catalog** (METR-0003) provides the normalization layer — cloud-
  specific SKUs map to unified catalog entries with a common cost taxonomy
- **Credit and Balance Management** (METR-0004) provides the aggregation
  layer — all costs from all clouds roll into a single tenant balance

The key architectural decision ([ADR-0019][adr-0019]) is to normalize at the
**event ingestion boundary** rather than at query time. Each cloud source block
transforms provider-specific cost data into Meteridian's canonical CloudEvents
format with standardized resource types, enabling downstream rating, billing,
and reporting to be cloud-agnostic.

[adr-0019]: ../../docs/adr/0019-multi-cloud-cost-normalization.md

---

## 2. Motivation

### 2.1 The Multi-Cloud Reality

Enterprise AI workloads increasingly span multiple environments:

| Environment | Typical Use | Example Services |
|-------------|-------------|------------------|
| AWS | Managed AI services, training at scale | Bedrock, SageMaker, EC2 P5 instances |
| Azure | Enterprise AI with Microsoft ecosystem | Azure OpenAI, Azure ML, ND-series VMs |
| GCP | Research and specialized models | Vertex AI, TPU v5, Cloud GPUs |
| On-premise (RHOAI) | Data sovereignty, latency-sensitive inference | OpenShift AI, vLLM on bare-metal GPUs |

A single customer may run the same model family across all four environments for
geographic proximity, cost optimization, regulatory compliance, or disaster
recovery. They need:

1. **Unified cost visibility** — Total AI spend across all environments in one report
2. **Cross-cloud cost comparison** — Same model, same workload, different costs per cloud
3. **Normalized metrics** — Token counts from AWS Bedrock and Azure OpenAI must aggregate
4. **Single billing relationship** — One invoice covering all environments

### 2.2 The AI Grid PoC Requirement

The AI Grid prospect operates a global AI inference platform spanning multiple
regions and cloud providers. Their requirement (MB-007) states:

> "Unified metering across multiple cloud providers (AWS, Azure, GCP) and
> on-premise infrastructure — a single pane of glass for AI workload costs
> regardless of where the model runs."

This requires Meteridian to:

- Ingest cost and usage data from AWS CloudWatch and Cost Explorer, Azure Monitor
  and Cost Management, GCP Cloud Monitoring and Billing, and on-premise RHOAI
- Normalize cloud-specific pricing (on-demand, spot, reserved instances) into a
  common cost model
- Aggregate costs across all environments for unified tenant billing
- Handle multi-currency scenarios (USD, EUR, GBP) with exchange rate management

### 2.3 Why This Is an Extension, Not a Redesign

Meteridian's architecture already supports multi-cloud metering in principle:

- **CloudEvents format** (ADR-0004) is cloud-agnostic by design
- **Redpanda Connect** (ADR-0013) has native connectors for AWS, Azure, and GCP
- **Product Catalog** (METR-0003) supports multi-currency price books
- **Block architecture** (METR-0002) allows cloud-specific source blocks

What is missing is the **specific configurations and normalization mappings** for
each cloud provider's AI services. This METR closes that gap.

---

## 3. Multi-Cloud Architecture

### 3.1 Design Principles

1. **Normalize at ingestion, not at query** — Cloud-specific formats are
   transformed into canonical CloudEvents at the source block boundary. All
   downstream processing (rating, billing, reporting) operates on a single
   unified schema.

2. **Cloud-specific source blocks, cloud-agnostic core** — Each cloud provider
   has its own Redpanda Connect pipeline configuration. The Meteridian core
   (rating engine, credit system, reporting) never sees cloud-specific formats.

3. **Additive, not replacing** — On-premise RHOAI metering (METR-0010) remains
   the primary data source. Cloud sources are additional inputs into the same
   billing pipeline.

4. **Eventual cost accuracy** — Cloud cost data may arrive with delay (AWS CUR
   is up to 24h delayed). Meteridian handles this through cost reconciliation
   cycles, not by blocking on cloud data availability.

### 3.2 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Cloud-Specific Source Blocks                         │
│                     (Redpanda Connect Pipelines)                         │
│                                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │ AWS      │   │ Azure    │   │ GCP      │   │ On-Prem (RHOAI)  │    │
│  │ Source   │   │ Source   │   │ Source   │   │ Source (§3.5     │    │
│  │ Block    │   │ Block    │   │ Block    │   │ of METR-0010)    │    │
│  │          │   │          │   │          │   │                  │    │
│  │ Bedrock  │   │ Azure    │   │ Vertex   │   │ MaaS Gateway     │    │
│  │ SageMaker│   │ OpenAI   │   │ AI       │   │ DCGM Exporter    │    │
│  │ CUR      │   │ Cost Mgmt│   │ BigQuery │   │ Prometheus       │    │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────────┬─────────┘    │
│       │              │              │                    │              │
│       └──────────────┴──────────────┴────────────────────┘              │
│                              │                                          │
│                    Canonical CloudEvents                                 │
│                    (Normalized resource types)                           │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Meteridian Core (Cloud-Agnostic)                      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                  Cost Normalization Layer                         │   │
│  │  - Cloud pricing model mapping (on-demand, spot, reserved)       │   │
│  │  - Exchange rate application                                     │   │
│  │  - SKU-to-catalog-entry resolution                               │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │                                       │
│  ┌──────────────┐  ┌───────────┴───┐  ┌──────────────┐                 │
│  │ Rating       │  │ Credit        │  │ Reporting    │                 │
│  │ Engine       │──│ and Balance   │──│ (Unified     │                 │
│  │ (METR-0003)  │  │ (METR-0004)   │  │  cross-cloud)│                 │
│  └──────────────┘  └───────────────┘  └──────────────┘                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 CloudEvent Envelope for Multi-Cloud

All cloud source blocks emit CloudEvents with a consistent envelope that
includes cloud-specific provenance metadata:

```json
{
  "specversion": "1.0",
  "id": "evt-mc-a1b2c3d4",
  "type": "ai.llm.tokens.consumed",
  "source": "aws-bedrock/us-east-1/anthropic.claude-v2",
  "time": "2026-06-19T14:30:00Z",
  "subject": "tenant-acme-corp",
  "datacontenttype": "application/json",
  "meteridiancloud": "aws",
  "meteridianregion": "us-east-1",
  "data": {
    "prompt_tokens": 1500,
    "completion_tokens": 800,
    "total_tokens": 2300,
    "model_name": "anthropic.claude-v2",
    "tenant_id": "acme-corp",
    "cloud_provider": "aws",
    "cloud_region": "us-east-1",
    "cloud_service": "bedrock",
    "pricing_model": "on-demand",
    "cloud_cost_usd": 0.0345,
    "cloud_currency": "USD",
    "cloud_account_id": "123456789012",
    "cloud_resource_id": "arn:aws:bedrock:us-east-1:123456789012:inference-profile/default"
  }
}
```

**Standard extension attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `meteridiancloud` | string | Cloud provider identifier: `aws`, `azure`, `gcp`, `onprem` |
| `meteridianregion` | string | Cloud region or on-premise cluster identifier |

**Standard data fields for multi-cloud events:**

| Field | Type | Description |
|-------|------|-------------|
| `cloud_provider` | string | Provider: `aws`, `azure`, `gcp`, `onprem` |
| `cloud_region` | string | Region identifier (provider-specific format) |
| `cloud_service` | string | Service name: `bedrock`, `azure-openai`, `vertex-ai`, `rhoai` |
| `pricing_model` | string | `on-demand`, `spot`, `reserved`, `committed-use`, `savings-plan` |
| `cloud_cost_usd` | decimal | Provider-reported cost in USD (for reconciliation) |
| `cloud_currency` | string | Original billing currency from the provider |
| `cloud_account_id` | string | Cloud account or subscription ID |
| `cloud_resource_id` | string | Cloud-specific resource ARN, URI, or ID |

---

## 4. Cloud-Specific Source Blocks

Each cloud provider has a dedicated Redpanda Connect pipeline configuration
that collects AI workload metrics and costs, normalizes them into Meteridian's
canonical CloudEvents format, and forwards them to the core billing pipeline.

### 4.1 AWS Source Block

**Data sources:**

| AWS Service | Data Type | Collection Method |
|-------------|-----------|-------------------|
| AWS Bedrock | Token usage per model invocation | CloudWatch Metrics (`Invocations`, `InputTokenCount`, `OutputTokenCount`) |
| AWS Bedrock | Per-invocation cost | CloudWatch Logs (model invocation logs with usage details) |
| AWS SageMaker | GPU instance hours | CloudWatch Metrics (`GPUUtilization`, `GPUMemoryUtilization`) |
| AWS Cost Explorer | Aggregated daily costs by service and tag | Cost Explorer API (`GetCostAndUsage`) |
| AWS CUR (Cost and Usage Report) | Line-item billing data | S3 export (Parquet or CSV) |

**Redpanda Connect pipeline:**

```yaml
# aws-ai-metering.yaml
# Collects AI workload metrics from AWS and emits normalized CloudEvents

input:
  generate:
    interval: "300s"
    mapping: "root = {}"

pipeline:
  processors:
    # Query AWS Bedrock invocation metrics from CloudWatch
    - branch:
        request_map: |
          root = {
            "MetricDataQueries": [{
              "Id": "bedrock_input_tokens",
              "MetricStat": {
                "Metric": {
                  "Namespace": "AWS/Bedrock",
                  "MetricName": "InputTokenCount",
                  "Dimensions": [{"Name": "ModelId", "Value": "${AWS_BEDROCK_MODEL_ID}"}]
                },
                "Period": 300,
                "Stat": "Sum"
              }
            }, {
              "Id": "bedrock_output_tokens",
              "MetricStat": {
                "Metric": {
                  "Namespace": "AWS/Bedrock",
                  "MetricName": "OutputTokenCount",
                  "Dimensions": [{"Name": "ModelId", "Value": "${AWS_BEDROCK_MODEL_ID}"}]
                },
                "Period": 300,
                "Stat": "Sum"
              }
            }],
            "StartTime": (now().ts_unix() - 300).ts_format("2006-01-02T15:04:05Z"),
            "EndTime": now().ts_format("2006-01-02T15:04:05Z")
          }
        processors:
          - aws_cloudwatch:
              region: "${AWS_REGION}"
              operation: GetMetricData
        result_map: "root.metrics = this"

    # Transform to canonical CloudEvents
    - mapping: |
        let input_tokens = this.metrics.MetricDataResults.
            filter(r -> r.Id == "bedrock_input_tokens").
            index(0).Values.index(0).or(0)
        let output_tokens = this.metrics.MetricDataResults.
            filter(r -> r.Id == "bedrock_output_tokens").
            index(0).Values.index(0).or(0)

        root = {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.llm.tokens.consumed",
          "source": "aws-bedrock/" + "${AWS_REGION}" + "/" + "${AWS_BEDROCK_MODEL_ID}",
          "time": now(),
          "subject": "${TENANT_ID}",
          "meteridiancloud": "aws",
          "meteridianregion": "${AWS_REGION}",
          "datacontenttype": "application/json",
          "data": {
            "prompt_tokens": $input_tokens,
            "completion_tokens": $output_tokens,
            "total_tokens": $input_tokens + $output_tokens,
            "model_name": "${AWS_BEDROCK_MODEL_ID}",
            "cloud_provider": "aws",
            "cloud_region": "${AWS_REGION}",
            "cloud_service": "bedrock",
            "pricing_model": "on-demand",
            "tenant_id": "${TENANT_ID}",
            "cloud_account_id": "${AWS_ACCOUNT_ID}",
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

### 4.2 Azure Source Block

**Data sources:**

| Azure Service | Data Type | Collection Method |
|---------------|-----------|-------------------|
| Azure OpenAI | Token usage per deployment | Azure Monitor Metrics (`TokenTransaction`, `ProcessedPromptTokens`, `GeneratedTokens`) |
| Azure OpenAI | Per-request token breakdown | Azure Monitor Diagnostic Logs |
| Azure ML | GPU compute hours | Azure Monitor Metrics (`GpuUtilization`, `GpuMemoryUtilization`) |
| Azure Cost Management | Daily costs by resource and tag | Cost Management API (`/query`) |

**Redpanda Connect pipeline:**

```yaml
# azure-ai-metering.yaml
# Collects AI workload metrics from Azure and emits normalized CloudEvents

input:
  generate:
    interval: "300s"
    mapping: "root = {}"

pipeline:
  processors:
    # Query Azure OpenAI token metrics from Azure Monitor
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                https://management.azure.com/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${AZURE_RG}/providers/Microsoft.CognitiveServices/accounts/${AZURE_OPENAI_ACCOUNT}/providers/microsoft.insights/metrics?api-version=2024-02-01&metricnames=ProcessedPromptTokens,GeneratedTokens&timespan=PT5M&interval=PT5M&aggregation=Total
              verb: GET
              headers:
                Authorization: "Bearer ${AZURE_TOKEN}"
        result_map: "root.metrics = this"

    # Transform to canonical CloudEvents
    - mapping: |
        let prompt_tokens = this.metrics.value.
            filter(m -> m.name.value == "ProcessedPromptTokens").
            index(0).timeseries.index(0).data.index(0).total.or(0)
        let completion_tokens = this.metrics.value.
            filter(m -> m.name.value == "GeneratedTokens").
            index(0).timeseries.index(0).data.index(0).total.or(0)

        root = {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.llm.tokens.consumed",
          "source": "azure-openai/" + "${AZURE_REGION}" + "/" + "${AZURE_DEPLOYMENT}",
          "time": now(),
          "subject": "${TENANT_ID}",
          "meteridiancloud": "azure",
          "meteridianregion": "${AZURE_REGION}",
          "datacontenttype": "application/json",
          "data": {
            "prompt_tokens": $prompt_tokens,
            "completion_tokens": $completion_tokens,
            "total_tokens": $prompt_tokens + $completion_tokens,
            "model_name": "${AZURE_DEPLOYMENT}",
            "cloud_provider": "azure",
            "cloud_region": "${AZURE_REGION}",
            "cloud_service": "azure-openai",
            "pricing_model": "on-demand",
            "tenant_id": "${TENANT_ID}",
            "cloud_account_id": "${AZURE_SUBSCRIPTION_ID}",
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

### 4.3 GCP Source Block

**Data sources:**

| GCP Service | Data Type | Collection Method |
|-------------|-----------|-------------------|
| Vertex AI | Token usage per prediction | Cloud Monitoring Metrics (`aiplatform.googleapis.com/prediction/online/token_count`) |
| Vertex AI | GPU utilization | Cloud Monitoring Metrics (`compute.googleapis.com/instance/gpu/utilization`) |
| GCP Billing | Line-item costs | BigQuery Billing Export |
| Cloud Monitoring | Custom metrics | Monitoring API v3 (`timeSeries.list`) |

**Redpanda Connect pipeline:**

```yaml
# gcp-ai-metering.yaml
# Collects AI workload metrics from GCP and emits normalized CloudEvents

input:
  generate:
    interval: "300s"
    mapping: "root = {}"

pipeline:
  processors:
    # Query Vertex AI token metrics from Cloud Monitoring
    - branch:
        request_map: "root = \"\""
        processors:
          - http:
              url: >-
                https://monitoring.googleapis.com/v3/projects/${GCP_PROJECT_ID}/timeSeries?filter=metric.type%3D%22aiplatform.googleapis.com%2Fprediction%2Fonline%2Ftoken_count%22&interval.startTime=${START_TIME}&interval.endTime=${END_TIME}&aggregation.alignmentPeriod=300s&aggregation.perSeriesAligner=ALIGN_SUM
              verb: GET
              headers:
                Authorization: "Bearer ${GCP_TOKEN}"
        result_map: "root.metrics = this"

    # Transform to canonical CloudEvents
    - mapping: |
        root = this.metrics.timeSeries.map_each(ts -> {
          "specversion": "1.0",
          "id": uuid_v4(),
          "type": "ai.llm.tokens.consumed",
          "source": "gcp-vertex/" + ts.resource.labels.location + "/" + ts.metric.labels.model_id.or("unknown"),
          "time": now(),
          "subject": "${TENANT_ID}",
          "meteridiancloud": "gcp",
          "meteridianregion": ts.resource.labels.location,
          "datacontenttype": "application/json",
          "data": {
            "total_tokens": ts.points.index(0).value.int64Value.or(0),
            "model_name": ts.metric.labels.model_id.or("unknown"),
            "cloud_provider": "gcp",
            "cloud_region": ts.resource.labels.location,
            "cloud_service": "vertex-ai",
            "pricing_model": "on-demand",
            "tenant_id": "${TENANT_ID}",
            "cloud_account_id": "${GCP_PROJECT_ID}",
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

### 4.4 On-Premise Source Block (RHOAI)

The on-premise source block is already defined in METR-0010 (§3.5 and §6.2).
It consumes CloudEvents from the MaaS external metering plugin and GPU metrics
from the OpenShift Thanos Querier. The multi-cloud extension adds the
`cloud_provider: "onprem"` metadata field for consistent cross-cloud reporting.

### 4.5 Operator Setup Checklist (Per Provider)

Use these checklists when onboarding a tenant's cloud billing data into
Meteridian. Split responsibilities between what the **customer configures** in
their cloud account and what the **Meteridian operator** configures in source
blocks (Redpanda Connect pipelines or FOCUS ingress).

#### AWS

**Customer configures (cloud-side):**

- [ ] Enable Cost and Usage Report (CUR) delivery to S3 (Parquet or CSV), or
      configure Cost Explorer API access for aggregated daily costs
- [ ] Create IAM role or user with least-privilege access:
      `cloudwatch:GetMetricData`, `ce:GetCostAndUsage`, `s3:GetObject` on the
      CUR bucket and prefix
- [ ] If Meteridian runs in a different account, configure cross-account
      `sts:AssumeRole` trust on the billing/metrics account
- [ ] Tag AI workloads (Bedrock, SageMaker) with tenant or cost-allocation keys
      for downstream attribution

**Meteridian source config:**

- [ ] AWS region and account ID
- [ ] CUR S3 bucket and object prefix (or Cost Explorer-only mode if no CUR)
- [ ] Credentials: IRSA, workload identity, or assumed-role ARN + external ID
- [ ] `tenant_id` mapping (static or from resource tags)
- [ ] Poll interval: ~5 minutes for CloudWatch metrics; daily batch for CUR
      reconciliation (see [§13 Open Questions](#13-open-questions) on two-speed
      metering)

#### Azure

**Customer configures (cloud-side):**

- [ ] Enable Cost Management exports and/or Cost Management Query API access
- [ ] Register a service principal or managed identity with **Monitoring Reader**
      and **Cost Management Reader** on the subscription or resource group
- [ ] Grant Azure OpenAI / Azure ML metric read access on Cognitive Services
      and compute resources
- [ ] Apply cost allocation tags on deployments and resource groups

**Meteridian source config:**

- [ ] Subscription ID and resource group scope
- [ ] Azure AD tenant ID, client ID, and client secret (or federated workload
      identity)
- [ ] Azure region and OpenAI deployment names
- [ ] `tenant_id` mapping
- [ ] Poll interval (typically 5–15 minutes for Monitor metrics)

#### GCP

**Customer configures (cloud-side):**

- [ ] Enable Cloud Monitoring and BigQuery billing export to a dedicated dataset
- [ ] Create a service account with `roles/monitoring.viewer` and
      `roles/bigquery.dataViewer` on the billing export dataset
- [ ] Enable Vertex AI monitoring metrics on prediction endpoints
- [ ] Label projects and resources for tenant or team attribution

**Meteridian source config:**

- [ ] GCP project ID and billing export BigQuery dataset name
- [ ] Service account key or Workload Identity binding
- [ ] Regions and Vertex AI model identifiers to poll
- [ ] `tenant_id` mapping
- [ ] Poll interval: ~5 minutes for Monitoring; 4–8 hour lag acceptable for
      BigQuery billing reconciliation

#### FOCUS (universal — Oracle, IBM, Alibaba, Huawei, any vendor)

FOCUS v1.4 provides a vendor-neutral cost export. Use when native cloud API
blocks are unavailable or for air-gapped deployments.

**Customer configures (cloud-side):**

- [ ] Export FOCUS v1.4 CSV or JSON from the vendor billing portal, or land
      files in a customer-controlled S3 bucket or blob container
- [ ] No cloud API credentials required for push-only delivery

**Meteridian source config:**

- [ ] **Push path:** POST files to `POST /api/v1/ingress/focus` (scheduled or
      ad hoc upload)
- [ ] **Pull path:** S3 or blob polling source block with bucket, prefix, and
      credentials (if Meteridian pulls from customer storage)
- [ ] `tenant_id` and file naming convention
- [ ] No live API polling when using push-only FOCUS ingress

#### General notes

| Topic | Guidance |
|-------|----------|
| **Pull vs push** | Native blocks (§4.1–4.3) **pull** on a schedule via cloud APIs. FOCUS supports **push** (ingress POST) or **pull** (object storage polling). |
| **Two-speed metering** | Use real-time metric pipelines for estimates, alerts, and balance enforcement; reconcile daily against authoritative billing (CUR, Azure Cost Management, BigQuery export, FOCUS files). |
| **Air-gapped / sovereign** | Prefer FOCUS file upload or object-storage polling; no outbound cloud API calls from the Meteridian cluster. |
| **Kubernetes-only fidelity** | On-prem and RHOAI workloads follow METR-0010 ingestion fidelity; see METR-0001 §1.1 fidelity tiers and ADR-0020 (when published) for cluster-local metering accuracy. |

---

## 5. Unified Cost Normalization Layer

### 5.1 The Normalization Problem

Cloud providers report costs differently:

| Aspect | AWS | Azure | GCP | On-Prem |
|--------|-----|-------|-----|---------|
| Token pricing unit | Per 1K tokens | Per 1K tokens | Per 1K characters (some models) | Self-hosted (no per-token charge from provider) |
| GPU pricing | Per-second (p5.48xlarge) | Per-hour (ND-series) | Per-second (a2-ultragpu) | Capital cost amortized per GPU-hour |
| Pricing models | On-demand, Spot, Reserved, Savings Plans | Pay-as-you-go, Reserved, Spot VMs | On-demand, Preemptible, CUDs (Committed Use Discounts) | Fixed (hardware depreciation) |
| Billing currency | USD (or local via AWS Marketplace) | USD, EUR, GBP, etc. (per EA agreement) | USD | N/A (internal transfer pricing) |
| Cost report delivery | CUR (S3, Parquet, 24h delay) | Cost Management API (near real-time) | BigQuery export (4-8h delay) | Immediate (local Prometheus) |
| Discount programs | EDP, Private Pricing | EA Discount, MACC | SUDs, CUDs, Flex CUDs | Volume purchase agreements |

### 5.2 Normalization Strategy

Meteridian normalizes at two levels:

**Level 1: Metric Normalization (at ingestion)**

All cloud-specific metrics are mapped to Meteridian's canonical resource types
(METR-0010 §5):

| Cloud Metric | Normalized Resource Type | Transformation |
|--------------|--------------------------|----------------|
| AWS `InputTokenCount` | `ai.llm.tokens.prompt` | Direct mapping |
| Azure `ProcessedPromptTokens` | `ai.llm.tokens.prompt` | Direct mapping |
| GCP `token_count` (input) | `ai.llm.tokens.prompt` | Filter by label `type=input` |
| AWS `OutputTokenCount` | `ai.llm.tokens.completion` | Direct mapping |
| Azure `GeneratedTokens` | `ai.llm.tokens.completion` | Direct mapping |
| GCP `token_count` (output) | `ai.llm.tokens.completion` | Filter by label `type=output` |
| AWS GPU utilization | `ai.gpu.hours` | Utilization × time × instance GPU count |
| Azure GPU utilization | `ai.gpu.hours` | Utilization × time × VM GPU count |
| GCP GPU utilization | `ai.gpu.hours` | Utilization × time × accelerator count |

**Level 2: Cost Normalization (at rating)**

Cloud-reported costs are mapped to Meteridian's unified cost model in the
Product Catalog:

| Cloud Pricing Concept | Meteridian Catalog Concept | Mapping |
|-----------------------|----------------------------|---------|
| On-demand per-token price | Tiered charge (METR-0003 §3.5) | Direct rate per resource type |
| Reserved Instance or CUD | Committed spend contract (METR-0004 §4) | Reservation charge + burst overage |
| Spot or Preemptible | Percentage discount modifier | SLA-tier multiplier (lower SLA = lower cost) |
| Savings Plan | Committed spend with flexible allocation | Credit grant with multi-resource drawdown |
| Enterprise Discount | Percentage modifier on base price | Markup and discount in price book |

### 5.3 Cloud SKU Resolution

Each cloud provider uses different identifiers for equivalent services. The
Product Catalog maintains a **Cloud SKU Map** that resolves cloud-specific
identifiers to unified catalog entries:

```yaml
cloud_sku_map:
  - cloud_provider: aws
    cloud_sku: "anthropic.claude-3-5-sonnet-20241022-v2:0"
    catalog_entry: "ai.llm.claude-3.5-sonnet"
    input_token_price: 0.003      # per 1K tokens
    output_token_price: 0.015     # per 1K tokens

  - cloud_provider: azure
    cloud_sku: "claude-3-5-sonnet"
    catalog_entry: "ai.llm.claude-3.5-sonnet"
    input_token_price: 0.003
    output_token_price: 0.015

  - cloud_provider: gcp
    cloud_sku: "claude-3-5-sonnet@20241022"
    catalog_entry: "ai.llm.claude-3.5-sonnet"
    input_token_price: 0.003
    output_token_price: 0.015

  - cloud_provider: onprem
    cloud_sku: "vllm/anthropic/claude-3.5-sonnet"
    catalog_entry: "ai.llm.claude-3.5-sonnet"
    input_token_price: 0.0        # Self-hosted: cost is GPU-hours, not tokens
    output_token_price: 0.0
    gpu_hour_cost: 3.50           # Internal transfer price per GPU-hour
```

This map enables:
- **Cross-cloud cost comparison**: Same model across different providers
- **Unified reporting**: "Total Claude 3.5 Sonnet spend" regardless of provider
- **Rate plan flexibility**: Different customers can have provider-specific or
  unified pricing

---

## 6. Cross-Cloud Aggregation

### 6.1 Aggregation Dimensions

Meteridian aggregates multi-cloud costs along these dimensions:

| Dimension | Description | Example Values |
|-----------|-------------|----------------|
| `tenant_id` | Customer or organization | `acme-corp`, `engineering-team-a` |
| `cloud_provider` | Cloud environment | `aws`, `azure`, `gcp`, `onprem` |
| `cloud_region` | Geographic region | `us-east-1`, `westeurope`, `us-central1` |
| `cloud_service` | AI service used | `bedrock`, `azure-openai`, `vertex-ai`, `rhoai` |
| `model_name` | AI model identifier | `claude-3.5-sonnet`, `gpt-4o`, `gemini-1.5-pro` |
| `pricing_model` | Cost structure | `on-demand`, `reserved`, `spot`, `committed-use` |
| `catalog_entry` | Unified SKU | `ai.llm.claude-3.5-sonnet` |

### 6.2 Aggregation Queries (SQL-Based Metrics)

Cross-cloud aggregation is implemented as SQL-based metrics in the Product
Catalog (METR-0003 §5), running in TimescaleDB:

```sql
-- Total AI spend by tenant across all clouds (monthly)
SELECT
    tenant_id,
    time_bucket('1 month', event_time) AS month,
    SUM(rated_amount_usd) AS total_spend_usd,
    cloud_provider,
    model_name
FROM rated_events
WHERE event_type = 'ai.llm.tokens.consumed'
GROUP BY tenant_id, month, cloud_provider, model_name
ORDER BY total_spend_usd DESC;

-- Cross-cloud cost comparison for same model
SELECT
    cloud_provider,
    cloud_region,
    model_name,
    SUM(prompt_tokens + completion_tokens) AS total_tokens,
    SUM(rated_amount_usd) AS total_cost_usd,
    SUM(rated_amount_usd) / NULLIF(SUM(prompt_tokens + completion_tokens), 0) * 1000
        AS effective_cost_per_1k_tokens
FROM rated_events
WHERE catalog_entry = 'ai.llm.claude-3.5-sonnet'
  AND event_time >= NOW() - INTERVAL '30 days'
GROUP BY cloud_provider, cloud_region, model_name;
```

### 6.3 Unified Reporting API

The reporting API exposes cross-cloud aggregations:

```
GET /api/v1/reports/ai-costs?
    tenant_id=acme-corp&
    group_by=cloud_provider,model_name&
    time_range=last_30d&
    currency=USD
```

Response:

```json
{
  "tenant_id": "acme-corp",
  "time_range": {"start": "2026-05-20", "end": "2026-06-19"},
  "total_cost_usd": 47250.00,
  "breakdown": [
    {
      "cloud_provider": "aws",
      "model_name": "anthropic.claude-3-5-sonnet",
      "tokens_consumed": 15000000,
      "cost_usd": 18500.00,
      "percentage": 39.2
    },
    {
      "cloud_provider": "azure",
      "model_name": "gpt-4o",
      "tokens_consumed": 8000000,
      "cost_usd": 12000.00,
      "percentage": 25.4
    },
    {
      "cloud_provider": "onprem",
      "model_name": "meta-llama/Llama-3.1-70B",
      "tokens_consumed": 50000000,
      "gpu_hours": 2400,
      "cost_usd": 8400.00,
      "percentage": 17.8
    },
    {
      "cloud_provider": "gcp",
      "model_name": "gemini-1.5-pro",
      "tokens_consumed": 12000000,
      "cost_usd": 8350.00,
      "percentage": 17.6
    }
  ]
}
```

---

## 7. Hybrid Deployment Metering

### 7.1 The Hybrid Challenge

Hybrid deployments combine on-premise GPU clusters with cloud GPU instances.
The metering challenge is that these environments report costs in fundamentally
different ways:

| Aspect | Cloud GPUs | On-Premise GPUs |
|--------|-----------|-----------------|
| Cost basis | Per-second or per-hour rental | Capital expenditure amortized over lifetime |
| Utilization reporting | Cloud monitoring APIs | DCGM Exporter and Prometheus |
| Pricing model | Pay-as-you-go or reserved | Internal transfer pricing |
| Cost visibility | Cloud billing APIs (delayed) | Real-time (local metrics) |
| Scaling | Elastic (add and remove instances) | Fixed capacity (hardware procurement) |

### 7.2 Unified GPU Cost Model

Meteridian normalizes hybrid GPU costs using an **effective GPU-hour rate**
that accounts for both cloud rental costs and on-premise amortized costs:

```yaml
gpu_cost_model:
  # Cloud GPU instances: direct cost from provider
  cloud_gpus:
    - provider: aws
      instance_type: p5.48xlarge
      gpu_count: 8
      gpu_sku: H100-SXM
      on_demand_rate_per_hour: 98.32    # AWS on-demand price
      reserved_rate_per_hour: 62.50     # 1-year reserved

    - provider: azure
      vm_size: Standard_ND96isr_H100_v5
      gpu_count: 8
      gpu_sku: H100-SXM
      on_demand_rate_per_hour: 98.00
      reserved_rate_per_hour: 61.80

    - provider: gcp
      machine_type: a3-highgpu-8g
      gpu_count: 8
      gpu_sku: H100-80GB
      on_demand_rate_per_hour: 98.68
      cud_rate_per_hour: 65.00          # 1-year CUD

  # On-premise GPUs: amortized capital cost
  onprem_gpus:
    - cluster_id: us-east-gpu-01
      gpu_sku: H100-SXM
      gpu_count: 64
      acquisition_cost: 1920000         # $30K per GPU × 64
      useful_life_months: 36
      monthly_depreciation: 53333       # acquisition / useful_life
      monthly_opex: 12800               # power, cooling, maintenance
      effective_rate_per_gpu_hour: 1.26  # (depreciation + opex) / (64 × 730 hours)
```

This enables cost comparisons like: "Running Llama 3.1 70B on-prem costs
$1.26 per GPU-hour vs $12.29 per GPU-hour on AWS Bedrock (on-demand) for
equivalent capacity."

### 7.3 Hybrid Metering Pipeline

The hybrid metering pipeline combines cloud and on-premise GPU metrics into
unified reports:

```
On-Premise (RHOAI)                     Cloud (AWS, Azure, GCP)
┌───────────────────┐                  ┌───────────────────┐
│ DCGM Exporter     │                  │ CloudWatch        │
│ Prometheus        │                  │ Azure Monitor     │
│ Thanos Querier    │                  │ Cloud Monitoring  │
└────────┬──────────┘                  └────────┬──────────┘
         │                                      │
         │ GPU utilization events               │ GPU utilization events
         │ (ai.gpu.hours, ai.gpu.utilization)   │ (ai.gpu.hours)
         │                                      │
         └──────────────────┬───────────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │    Meteridian Core           │
              │                             │
              │  Unified GPU cost report:   │
              │  - On-prem: 2400 GPU-hours  │
              │    @ $1.26/hr = $3,024      │
              │  - AWS: 800 GPU-hours       │
              │    @ $12.29/hr = $9,832     │
              │  - Azure: 400 GPU-hours     │
              │    @ $12.25/hr = $4,900     │
              │                             │
              │  Total: 3600 GPU-hours      │
              │  Total cost: $17,756        │
              │  Avg effective rate: $4.93  │
              └─────────────────────────────┘
```

---

## 8. Currency and Exchange Rate Handling

### 8.1 Multi-Currency Architecture

Meteridian's Product Catalog (METR-0003) already supports multi-currency
price books. Multi-cloud metering extends this with:

1. **Provider billing currencies** — Each cloud provider may bill in a different
   currency (Azure EA agreements often use EUR or GBP)
2. **Normalization currency** — All costs are normalized to a single reporting
   currency (configurable per tenant, defaults to USD)
3. **Exchange rate sources** — Configurable exchange rate providers (ECB, Open
   Exchange Rates, custom corporate rates)

### 8.2 Exchange Rate Management

```yaml
exchange_rate_config:
  normalization_currency: USD
  rate_source: ecb                    # European Central Bank (free, daily)
  fallback_source: openexchangerates  # Paid API for real-time rates
  refresh_interval: 86400             # Daily refresh (seconds)
  historical_rates: true              # Maintain rate history for reconciliation

  # Override rates for internal transfer pricing
  overrides:
    - from: EUR
      to: USD
      rate: 1.08                      # Fixed corporate rate
      effective_from: "2026-01-01"
      effective_to: "2026-12-31"
```

### 8.3 Cost Normalization Flow

```
Cloud Provider Invoice        Exchange Rate Service        Meteridian
┌──────────────────┐         ┌──────────────────┐        ┌──────────────┐
│ Azure: €1,234.56 │         │ EUR→USD: 1.0812  │        │ Normalized:  │
│ AWS:   $5,678.90 │─────────│ GBP→USD: 1.2645  │───────►│ $7,013.46    │
│ GCP:   $3,456.78 │         │ (daily ECB rate)  │        │ (all USD)    │
└──────────────────┘         └──────────────────┘        └──────────────┘
```

For the Azure invoice in EUR:
- Raw cost: €1,234.56
- Exchange rate (EUR→USD, date of invoice): 1.0812
- Normalized cost: $1,334.56

All downstream processing (credit deduction, reporting, budgets) operates
in the normalized currency.

---

## 9. Product Catalog Integration

### 9.1 Multi-Cloud Rate Plans

The Product Catalog supports rate plans that span multiple cloud providers:

```yaml
plan:
  name: "Enterprise AI — Multi-Cloud"
  description: "Unified AI billing across AWS, Azure, GCP, and on-premise"

  charges:
    # Per-token charges (same rate regardless of cloud)
    - name: "LLM Tokens (Unified Rate)"
      resource_type: "ai.llm.tokens.total"
      model: tiered
      tiers:
        - up_to: 100000000        # First 100M tokens
          unit_price: 0.000008
        - up_to: null
          unit_price: 0.000005
      currency: USD
      filter:
        catalog_entry: "ai.llm.*"  # Applies to all LLM models

    # Per-GPU-hour charges (different rate by environment)
    - name: "Cloud GPU Hours"
      resource_type: "ai.gpu.hours"
      model: volume
      unit_price: 12.00           # Pass-through cloud cost + margin
      currency: USD
      filter:
        cloud_provider: ["aws", "azure", "gcp"]

    - name: "On-Premise GPU Hours"
      resource_type: "ai.gpu.hours"
      model: volume
      unit_price: 3.50            # Internal transfer price
      currency: USD
      filter:
        cloud_provider: "onprem"
```

### 9.2 Cloud Cost Pass-Through Model

For customers who want actual cloud costs passed through (with markup):

```yaml
plan:
  name: "AI Infrastructure — Cost Plus"
  description: "Actual cloud costs with 15% management fee"

  charges:
    - name: "Cloud Cost Pass-Through"
      resource_type: "ai.cloud.cost"
      model: percentage
      base_field: "cloud_cost_usd"    # Actual cost reported by cloud provider
      percentage: 115                  # 100% cost + 15% markup
      currency: USD

    - name: "On-Premise GPU (Fixed Rate)"
      resource_type: "ai.gpu.hours"
      model: volume
      unit_price: 3.50
      currency: USD
      filter:
        cloud_provider: "onprem"
```

---

## 10. Implementation Approach

### 10.1 Phased Delivery

| Phase | Scope | Effort |
|-------|-------|--------|
| Phase 1 (PoC) | AWS Bedrock source block + on-premise RHOAI + unified reporting | 2-3 weeks |
| Phase 2 | Azure OpenAI source block + cost normalization + currency handling | 2 weeks |
| Phase 3 | GCP Vertex AI source block + cross-cloud aggregation queries | 2 weeks |
| Phase 4 | Cloud cost reconciliation (CUR, Azure Cost Management, BigQuery export) | 3-4 weeks |

### 10.2 Component Breakdown

| Component | Type | Effort | Dependencies |
|-----------|------|--------|--------------|
| AWS Bedrock source block (Redpanda Connect YAML) | Configuration | 3-4 days | AWS credentials, CloudWatch access |
| Azure OpenAI source block (Redpanda Connect YAML) | Configuration | 3-4 days | Azure credentials, Monitor access |
| GCP Vertex AI source block (Redpanda Connect YAML) | Configuration | 3-4 days | GCP credentials, Monitoring API |
| Cloud SKU Map (Product Catalog entries) | Data definition | 2-3 days | Model pricing research |
| Cost normalization processor (Bloblang) | Configuration | 2-3 days | Exchange rate API access |
| Cross-cloud aggregation SQL metrics | SQL definitions | 2-3 days | TimescaleDB schema |
| Unified reporting API endpoint | Code | 3-5 days | Rating engine, TimescaleDB |
| Exchange rate integration block | Configuration | 1-2 days | ECB or OER API |
| On-premise GPU amortization calculator | Configuration | 1-2 days | Hardware cost data |

### 10.3 Redpanda Connect as the Integration Mechanism

All cloud source blocks are implemented as **Redpanda Connect pipeline
configurations** (YAML files), not as custom code. This is consistent with
the two-layer architecture (ADR-0013):

- **Layer 1 (Redpanda Connect):** Handles cloud API authentication, metric
  collection, format transformation, and CloudEvent emission
- **Layer 2 (Meteridian Blocks):** Handles rating, credit deduction, and
  reporting (cloud-agnostic)

Redpanda Connect provides native components for all three cloud providers:

| Cloud | Redpanda Connect Components |
|-------|----------------------------|
| AWS | `aws_cloudwatch`, `aws_s3`, `aws_sqs`, `aws_kinesis` inputs; IAM auth |
| Azure | `http` processor with Azure AD OAuth; Azure Blob, Event Hubs inputs |
| GCP | `http` processor with GCP service account auth; GCS, Pub/Sub inputs |

---

## 11. PoC Scope — AI Grid

### 11.1 PoC Minimum Viable Set

For the AI Grid PoC (target: July 31, 2026), the multi-cloud demonstration
includes:

| # | Capability | Implementation | Effort |
|---|-----------|---------------|--------|
| 1 | On-premise RHOAI metering | Already defined in METR-0010 §3.5 and §6.2 | (included in METR-0010) |
| 2 | AWS Bedrock token metering | CloudWatch source block (§4.1) | 3 days |
| 3 | Unified cross-cloud report | SQL aggregation query + API endpoint (§6.2) | 2-3 days |
| 4 | Single-tenant multi-cloud view | Reporting API with `group_by=cloud_provider` | 1-2 days |

### 11.2 PoC Architecture

```
                    AI Grid PoC — Multi-Cloud View
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  On-Premise (Primary)         Cloud (Supplementary)             │
│  ┌───────────────────┐       ┌───────────────────┐             │
│  │ RHOAI + vLLM      │       │ AWS Bedrock       │             │
│  │ (METR-0010)       │       │ (Claude, Llama)   │             │
│  │                   │       │                   │             │
│  │ MaaS Gateway      │       │ CloudWatch        │             │
│  │ DCGM Exporter     │       │ Metrics           │             │
│  └────────┬──────────┘       └────────┬──────────┘             │
│           │                           │                        │
│           └───────────┬───────────────┘                        │
│                       │                                        │
│                       ▼                                        │
│           ┌───────────────────────┐                            │
│           │ Meteridian            │                            │
│           │ Unified AI Cost View  │                            │
│           │                       │                            │
│           │ Total: $47,250        │                            │
│           │ On-prem: $8,400 (18%) │                            │
│           │ AWS: $38,850 (82%)    │                            │
│           └───────────────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.3 Post-PoC Extensions

| Item | Phase | Effort |
|------|-------|--------|
| Azure OpenAI source block | v1.0 | 2 weeks |
| GCP Vertex AI source block | v1.0 | 2 weeks |
| Full CUR and BigQuery cost reconciliation | v1.0 | 3-4 weeks |
| Multi-currency with exchange rates | v1.0 | 1-2 weeks |
| Hybrid GPU cost comparison reports | v1.0 | 1-2 weeks |
| Cloud cost optimization recommendations | v1.x | 4-6 weeks |

---

## 12. Timeline and Effort Estimate

### 12.1 PoC Delivery (Integrated with METR-0010 Timeline)

The multi-cloud metering PoC runs in parallel with the METR-0010 AI metering
PoC (both target July 31, 2026):

| Week | Deliverable | Effort |
|------|------------|--------|
| 2 (Jun 30-Jul 4) | AWS Bedrock source block configuration | 3 days |
| 3 (Jul 7-11) | Cross-cloud aggregation queries and reporting endpoint | 2-3 days |
| 3 (Jul 7-11) | Cloud SKU map for Bedrock models | 1 day |
| 4 (Jul 14-18) | Integration testing (RHOAI + AWS) and demo preparation | 2 days |

**Additional PoC effort: ~8-9 person-days** (on top of METR-0010's 18 person-days).

### 12.2 Full Multi-Cloud Delivery (Post-PoC)

| Phase | Deliverable | Effort |
|-------|------------|--------|
| v1.0 (Aug) | Azure OpenAI source block + Azure Cost Management integration | 2 weeks |
| v1.0 (Aug) | GCP Vertex AI source block + BigQuery billing export | 2 weeks |
| v1.0 (Sep) | Cost normalization layer with exchange rate management | 2 weeks |
| v1.0 (Sep) | Hybrid GPU cost model and comparison reporting | 2 weeks |
| v1.0 (Oct) | Cloud cost reconciliation (CUR, Azure, BigQuery batch processing) | 3 weeks |

---

## 13. Open Questions

1. **Cloud credential management** — How should Meteridian manage credentials
   for multiple cloud providers? Options: Kubernetes Secrets, HashiCorp Vault,
   cloud-native secret managers (AWS Secrets Manager, Azure Key Vault, GCP
   Secret Manager). This may warrant a separate ADR.

2. **Cost data freshness vs accuracy** — Cloud cost data arrives with delay
   (AWS CUR: up to 24h; Azure: near real-time; GCP: 4-8h). Should Meteridian
   use real-time metric estimates or wait for authoritative cost data? The
   recommendation is: use real-time metrics for balance management and threshold
   enforcement, reconcile with authoritative cost data daily.

3. **Multi-cloud enforcement** — If a tenant exhausts their budget on AWS
   Bedrock, should Meteridian also block their Azure OpenAI and on-prem
   usage? The answer depends on whether the budget is unified or per-cloud.
   Both models should be supported.

4. **Cloud discount attribution** — Enterprise discounts (AWS EDP, Azure MACC,
   GCP CUDs) apply at the account level, not per-workload. How should these
   discounts be attributed to individual tenants? Options: proportional
   allocation, first-come-first-served, or manual assignment.

5. **Provider-native billing integration** — Should Meteridian consume cloud
   providers' native billing data (CUR, Azure Cost Management exports, GCP
   BigQuery) for reconciliation, or is the metric-based approach sufficient?
   For billing-grade accuracy, consuming native billing data is recommended
   but adds significant complexity.

---

## 14. Related Documents

### Enhancement Proposals

- [METR-0002](../0002-extensibility/extensibility.md) — Platform Extensibility
  and Block Marketplace (source block architecture)
- [METR-0003](../0003-product-catalog/product-catalog.md) — Product and Service
  Catalog (multi-currency price books, cloud SKU mapping)
- [METR-0004](../0004-credit-token-billing/credit-token-billing.md) — Credit,
  Prepaid, and Token Billing (unified balance management across clouds)
- [METR-0005](../0005-internal-token-economy/internal-token-economy.md) —
  Internal Budget Units and Chargeback (cross-cloud chargeback reporting)
- [METR-0010](../0010-ai-metering/ai-metering.md) — AI Workload Metering
  (on-premise RHOAI metering, resource type definitions)
- [METR-0011](../0011-enforcement-integration/enforcement-integration.md) —
  Enforcement Integration (cross-cloud enforcement signals)

### Architecture Decision Records

- [ADR-0004](../../docs/adr/0004-cloudevents-event-format.md) — CloudEvents as
  canonical event format (enables cloud-agnostic downstream processing)
- [ADR-0013](../../docs/adr/0013-two-layer-data-architecture.md) — Two-layer
  data architecture (Redpanda Connect source blocks for cloud integration)
- [ADR-0019](../../docs/adr/0019-multi-cloud-cost-normalization.md) — Multi-
  cloud cost normalization (ingestion-time vs query-time normalization)

### External References

- [AWS Bedrock CloudWatch Metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-cw.html) —
  `InputTokenCount`, `OutputTokenCount`, `Invocations` metrics
- [Azure OpenAI Monitor Metrics](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/monitoring) —
  `ProcessedPromptTokens`, `GeneratedTokens` metrics
- [GCP Vertex AI Monitoring](https://cloud.google.com/vertex-ai/docs/predictions/monitor-prediction) —
  `aiplatform.googleapis.com/prediction/online/token_count`
- [Redpanda Connect AWS Components](https://docs.redpanda.com/connect/components/inputs/aws_cloudwatch/) —
  Native AWS input and output connectors
- [ECB Exchange Rates API](https://data.ecb.europa.eu/help/api/data) —
  Free daily exchange rates from the European Central Bank
