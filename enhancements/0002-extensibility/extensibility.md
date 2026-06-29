# METR-0002: Platform Extensibility and Block Marketplace

- **Status:** provisional
- **Authors:** @pgarciaq, @jordigilh
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18
- **Depends on:** METR-0001 (Platform Architecture)
- **Related:** METR-0003 (Product Catalog), METR-0004 (Credit and Token Billing), METR-0005 (Internal Budget Units), METR-0006 (Developer Experience), METR-0007 (Legal and Regulatory Compliance), METR-0008 (Compliance-as-Code), METR-0009 (E-Invoicing Engine)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Block Model](#3-block-model)
4. [Pipeline Definition](#4-pipeline-definition)
5. [Apache Arrow as Unified Serialization](#5-apache-arrow-as-unified-serialization)
6. [Plugin Runtime Model](#6-plugin-runtime-model)
7. [Observability and Performance](#7-observability-and-performance)
8. [AI-First Extensibility](#8-ai-first-extensibility)
22. [Marketplace and Developer Program](#9-marketplace-and-developer-program)
10. [Pluggable Payment Providers](#10-pluggable-payment-providers)
11. [Capability-Based Block Security](#11-capability-based-block-security)
12. [Stateful Blocks](#12-stateful-blocks)
13. [Schema Evolution](#13-schema-evolution)
14. [Error Handling and Delivery Guarantees](#14-error-handling-and-delivery-guarantees)
15. [Hot Reload](#15-hot-reload)
16. [Open Questions](#16-open-questions)
17. [Related Documents](#17-related-documents)
18. [External References](#18-external-references)

---

## 1. Summary

Meteridian extends through a **block-based dataflow architecture**. A processing
block is the fundamental unit of computation: it receives Apache Arrow
RecordBatches on its input ports, performs a well-defined transformation, and
emits Arrow RecordBatches on its output ports. Blocks are connected into
directed acyclic graphs (DAGs) called **pipelines**, forming composable
metering-to-billing workflows.

Blocks can be **official** (maintained by the Meteridian team), **third-party
verified** (published by partners who have undergone code review and signed a
developer program agreement), or **community** (published by anyone, running in
a fully sandboxed environment). A tiered trust model governs the capabilities
each block may exercise.

Extensibility is enabled through complementary mechanisms, phased across two
releases (see ADR-0006):

**v1.0 (launch):**
- **Go SDK** — native, in-process blocks with zero overhead for
  performance-critical paths.
- **gRPC out-of-process (Arrow Flight)** — separate-process blocks for
  marketplace code, heavy compute, or languages requiring full runtimes
  (Python, Java, Node.js, Rust). Arrow Flight is production-proven in Dremio,
  InfluxDB, and Apache Spark (see ADR-0005).

**v2.0 (planned):**
- **WASM sandbox (Extism)** — in-process but fully isolated blocks for higher
  block density, supporting Rust, TinyGo, C, AssemblyScript, and Zig. Deferred
  because Arrow IPC over WASM boundaries is not yet production-proven and WASI
  Preview 2 is still stabilizing.

The primary user interface for building and managing pipelines is **AI-first**:
Meteridian exposes a Model Context Protocol (MCP) server and an Agent-to-Agent
(A2A) endpoint, enabling AI assistants to create blocks, assemble pipelines,
diagnose performance bottlenecks, and optimize configurations through natural
language. A visual editor (Node-RED-inspired) is planned for v2+ as a secondary
interface; v1 provides a read-only pipeline dashboard.

A **block marketplace** with software supply chain security (SLSA + Sigstore)
and a pluggable payment provider model rounds out the extensibility story,
creating a self-sustaining ecosystem around the platform.

---

## 2. Motivation

### 2.1 The Gap in Billing Platform Extensibility

No metering or billing platform today offers a true dataflow plugin
architecture. The state of the art in extensibility across billing
competitors is limited:

| Platform | Extensibility model | Limitations |
|----------|-------------------|-------------|
| Stripe | Webhooks, Stripe Apps (iframe) | Fire-and-forget events, no data transformation pipeline |
| Zuora | Workflow Builder, custom fields | Linear sequences, no branching, no third-party blocks |
| Orb | API-only, event ingestion | No plugin system, all customization via API calls |
| Amberflo | Webhooks, SDK ingestion | Event forwarding only, no in-pipeline processing |
| OpenMeter | CloudEvents, Kafka | Raw event bus, no typed processing blocks |
| Lago | Webhooks, API | No plugin architecture, customization via application code |

Every platform above forces customers into one of two patterns:

1. **Webhook chains** — events are POSTed to external HTTP endpoints, processed
   externally, and results pushed back via API. This adds latency, requires
   customers to host and scale their own infrastructure, and offers no schema
   enforcement or backpressure.

2. **Build-it-yourself** — customers write custom application code that calls
   the billing platform's API, implementing their own metering enrichment,
   rating logic, and payment routing. This is the dominant pattern and it
   results in fragile, tightly-coupled integrations.

Meteridian proposes a third path: **blocks as composable computation units**,
connected into typed, validated pipelines with built-in observability and
lifecycle management.

### 2.2 Precedent in Adjacent Domains

The block-based dataflow model is battle-tested in multiple domains:

**Audio and multimedia processing (DAWs):** Digital Audio Workstations like
Ableton Live, Logic Pro, and Reaper have used a block-based plugin architecture
for decades. Audio plugins (VST, AU, AAX) receive audio buffers, process them
(EQ, compression, reverb), and emit processed buffers. Plugins are connected
into signal chains. The plugin ecosystem has its own marketplace (Plugin
Boutique, Splice), trust tiers (manufacturer-signed vs. community), and
standardized interfaces. This model has proven that a thriving third-party
ecosystem can emerge around a well-defined block interface.

**Data engineering:** Apache NiFi pioneered visual dataflow programming for data
routing and transformation. Redpanda Connect (formerly Benthos) offers a
configuration-driven pipeline of typed processors. Node-RED provides a visual
flow editor for IoT and event processing. Apache Beam defines a unified model
for batch and stream processing with composable transforms. All of these
demonstrate that dataflow architectures scale from simple ETL to complex
event-driven systems.

**Linux kernel extensibility (eBPF):** eBPF provides a sandboxed, verified
execution model inside the Linux kernel. eBPF programs are small blocks of
computation that attach to kernel hooks, process data (packets, syscalls,
tracepoints), and can be composed. The eBPF verifier ensures safety before
execution — analogous to our capability-based security model for WASM blocks.
The success of eBPF demonstrates that even the most performance-sensitive
systems benefit from safe, sandboxed extensibility.

### 2.3 Why Now

Three technology shifts make this architecture viable in 2026:

1. **Apache Arrow maturity** — Arrow Flight RPC, Arrow IPC, and multi-language
   SDKs are production-ready. Arrow provides the universal serialization format
   that makes block interoperability practical without schema translation layers.

2. **WASM runtime maturity** — Extism and the WASI preview2 specification
   provide production-grade sandboxed execution with near-native performance.
   WASM is no longer a browser-only technology.

3. **AI agents as first-class users** — MCP and A2A establish standard protocols
   for AI agents to interact with tools and other agents. Building for AI-first
   interaction is no longer speculative; it's the expected interface for
   developer platforms.

---

## 3. Block Model

### 3.1 Block Definition

A **processing block** is the atomic unit of computation in Meteridian. Every
block implements a uniform interface:

```mermaid
flowchart LR
    Input["Arrow RecordBatch\n(input)"] --> PB["Processing Block\n\n- Config (TOML)\n- State (optional)\n- Metrics (OTel)\n- Health (/healthz)"]
    PB --> Output["Arrow RecordBatch\n(output)"]
```

Blocks are pure data transformations with explicit inputs, outputs, and side
effects. A block's behavior is fully determined by its configuration and input
data — no hidden dependencies, no implicit global state.

### 3.2 Block Types

Five fundamental block types cover all dataflow patterns:

**Source blocks** produce events. They are the entry points of a pipeline.

```mermaid
flowchart LR
    Source["Source\n\n(collector,\nimporter,\ngenerator)"] --> Output["Arrow RecordBatch"]
```

No input ports. One or more output ports.

Examples: Prometheus collector, CloudEvents receiver, CSV importer, Kafka
consumer, S3 bucket watcher.

**Transform blocks** modify, enrich, filter, or aggregate events.

```mermaid
flowchart LR
    Input["Arrow RecordBatch"] --> Transform["Transform\n\n(enrich, filter,\naggregate, rate)"]
    Transform --> Output["Arrow RecordBatch"]
```

One input port. One output port.

Examples: geo-enricher, tiered rater, tag normalizer, unit converter,
deduplicator, windowed aggregator, payment provider (request-response).

**Sink blocks** consume events. They are the terminal nodes of a pipeline.

```mermaid
flowchart LR
    Input["Arrow RecordBatch"] --> Sink["Sink\n\n(storage, export,\nwebhook)"]
```

No output ports. One input port.

Examples: TimescaleDB writer, S3 Parquet exporter, webhook emitter, audit log
writer, data lake archiver.

**Fork blocks** duplicate events to multiple downstream paths.

```mermaid
flowchart LR
    Input["Arrow RecordBatch"] --> Fork["Fork"]
    Fork --> A["Path A"]
    Fork --> B["Path B"]
    Fork --> C["Path C"]
```

Fork blocks perform zero-copy duplication using Arrow's reference-counted
buffers. All downstream paths receive the same RecordBatch without memory
duplication.

**Join blocks** merge events from multiple upstream paths.

```mermaid
flowchart LR
    A["Path A"] --> Join["Join"]
    B["Path B"] --> Join
    C["Path C"] --> Join
    Join --> Output["Arrow RecordBatch"]
```

Join blocks support merge (interleave), union (concatenate schemas), and
temporal join (align by timestamp) strategies.

### 3.3 Block Lifecycle

Every block follows a deterministic lifecycle:

```mermaid
stateDiagram-v2
    [*] --> Init
    Init --> Ready : Load config, validate schemas,\nacquire capabilities
    Ready --> Processing
    Processing --> Draining : Pipeline stop or block update
    Draining --> Stopped : Flush buffers, close connections,\npersist state
    Init --> Error : config invalid,\ncapability denied
    Processing --> Error : processing failure
    Draining --> Error : flush failed, timeout
```

- **Init**: The platform loads the block's configuration, validates input/output
  Arrow schemas against the pipeline, and grants declared capabilities. If any
  validation fails, the block enters an error state and the pipeline does not
  start.

- **Ready**: The block has passed initialization and is waiting for its first
  input batch. Health endpoint returns healthy.

- **Processing**: The block is actively receiving and emitting RecordBatches.
  Metrics are reported continuously via OpenTelemetry.

- **Draining**: Triggered by a pipeline stop or block update. The block stops
  accepting new input, flushes any internal buffers, and persists state if
  stateful. Upstream blocks apply backpressure.

- **Stopped**: The block has released all resources and is no longer active.

### 3.4 Block Manifest

Every block declares its contract through a manifest file:

```toml
[block]
name = "geo-enricher"
version = "1.2.0"
description = "Enriches events with geolocation data from IP addresses"
author = "acme-corp"
license = "Apache-2.0"
runtime = "grpc"  # "go" or "grpc" (v1); "wasm" added in v2

[input]
schema = [
    { name = "source_ip", type = "utf8", required = true },
    { name = "timestamp", type = "timestamp[us, tz=UTC]", required = true },
]

[output]
schema = [
    { name = "source_ip", type = "utf8" },
    { name = "timestamp", type = "timestamp[us, tz=UTC]" },
    { name = "country_code", type = "utf8" },
    { name = "region", type = "utf8" },
    { name = "city", type = "utf8" },
    { name = "latitude", type = "float64" },
    { name = "longitude", type = "float64" },
]

[capabilities]
network = ["api.maxmind.com:443"]
fields.read = ["source_ip", "timestamp"]
fields.write = ["country_code", "region", "city", "latitude", "longitude"]
environment = ["MAXMIND_API_KEY"]

[config]
schema = "config.schema.json"  # JSON Schema for block-specific config

[health]
endpoint = "/healthz"
interval = "10s"
timeout = "5s"

[metrics]
endpoint = "/metrics"
format = "otlp"
```

---

## 4. Pipeline Definition

### 4.1 YAML Pipeline Syntax

Pipelines are defined declaratively in YAML. The pipeline specification
describes the blocks, their configurations, and the connections between them.

```yaml
apiVersion: meteridian.io/v1
kind: Pipeline
metadata:
  name: ocp-cost-enrichment
  labels:
    team: platform-engineering
    environment: production
spec:
  blocks:
    - name: ocp-collector
      type: meteridian/ocp-prometheus-collector
      config:
        prometheus_url: "http://thanos-querier:9090"
        scrape_interval: 60s
        metrics:
          - kube_pod_container_resource_requests
          - kube_pod_container_resource_limits
          - container_cpu_usage_seconds_total
          - container_memory_working_set_bytes

    - name: geo-enricher
      type: acme-corp/geo-enricher
      version: "1.2.0"
      config:
        database: maxmind-city
        cache_ttl: 3600s

    - name: tag-normalizer
      type: meteridian/tag-normalizer
      config:
        rules:
          - match: "app.kubernetes.io/*"
            normalize_to: "k8s_{key}"
          - match: "cost-center"
            normalize_to: "cost_center"

    - name: cost-rater
      type: meteridian/tiered-rater
      config:
        price_list: "default"
        currency: "USD"

    - name: cost-fork
      type: meteridian/fork
      config:
        strategy: reference  # zero-copy

    - name: timescale-sink
      type: meteridian/timescaledb-writer
      config:
        connection: "postgres://meteridian:***@tsdb:5432/metering"
        hypertable: "usage_events"

    - name: invoice-sink
      type: meteridian/invoice-generator
      config:
        template: "standard-v2"
        output_format: "pdf"

  connections:
    - from: ocp-collector
      to: geo-enricher
    - from: geo-enricher
      to: tag-normalizer
    - from: tag-normalizer
      to: cost-rater
    - from: cost-rater
      to: cost-fork
    - from: cost-fork
      to: timescale-sink
      port: 0
    - from: cost-fork
      to: invoice-sink
      port: 1
```

The resulting pipeline DAG:

```mermaid
flowchart TD
    A["ocp-collector\n(source)"] --> B["geo-enricher\n(transform)"]
    B --> C["tag-normalizer\n(transform)"]
    C --> D["cost-rater\n(transform)"]
    D --> E["cost-fork\n(fork)"]
    E --> F["timescale-sink\n(sink)"]
    E --> G["invoice-sink\n(sink)"]
```

### 4.2 CUE for Type Safety

For complex pipelines where YAML's lack of type safety becomes a liability,
Meteridian supports CUE as an alternative pipeline definition language:

```cue
import "meteridian.io/v1"

pipeline: v1.#Pipeline & {
    metadata: {
        name: "ocp-cost-enrichment"
    }
    spec: {
        blocks: [
            {
                name: "ocp-collector"
                type: "meteridian/ocp-prometheus-collector"
                config: {
                    prometheus_url: "http://thanos-querier:9090"
                    scrape_interval: "60s"
                }
            },
            // ... additional blocks
        ]
    }
}
```

CUE provides compile-time validation of block configurations against their
declared schemas, catching errors before deployment.

### 4.3 Pipeline Validation

Before a pipeline is deployed, the platform performs validation:

1. **Schema compatibility** — Every connection's output Arrow schema must be a
   superset of the downstream block's required input schema.
2. **DAG validation** — The graph must be acyclic. Cycles are rejected.
3. **Capability validation** — Every block's declared capabilities must be
   permitted by its trust tier.
4. **Resource validation** — Estimated resource requirements (CPU, memory) must
   fit within the tenant's quotas.
5. **Version compatibility** — Block versions must satisfy declared dependency
   constraints.

---

## 5. Apache Arrow as Unified Serialization

### 5.1 The Decision

Apache Arrow is the **single serialization format** for all data exchange in
Meteridian. This is a deliberate, opinionated choice that eliminates an entire
category of complexity: format conversion.

```mermaid
flowchart TB
    subgraph ArrowEverywhere["Arrow Everywhere"]
        direction LR
        B1["Block"] <-- "Arrow Flight RPC (v1)" --> B2["Block"]
        B3["Block"] <-- "Arrow IPC · shared mem (v2)" --> B4["WASM Block"]
        B5["Block"] <-- "Arrow RecordBatch (v1)" --> S["Storage\n(Parquet)"]
        API["API"] <-- "Arrow Flight SQL (v1)" --> QE["Query Engine"]
    end
```

> One developer-facing format. One schema. Zero conversion.
> (Arrow Flight uses Protobuf internally for control plane — hidden from block developers by the Flight SDK.)

### 5.2 Arrow Flight RPC for Block-to-Block Communication

Arrow Flight is an RPC framework built on gRPC that uses Arrow RecordBatches as
its native payload. Block authors interact only with Arrow types — Arrow Flight
handles the transport.

**Important clarification:** Arrow Flight internally uses Protobuf for control
plane messages (FlightDescriptor, FlightInfo, Ticket, Action, Result). This is
by design and is encapsulated within the Flight SDK. Block developers never
write `.proto` files, never run `protoc`, and never manage Protobuf schemas.
From the developer's perspective, there is one data model: Arrow.

Arrow Flight is production-proven: Dremio uses it as its primary data transfer
protocol, InfluxDB v3 uses it for query results, Apache Spark uses it for
cross-language UDFs, and DuckDB uses it for remote data access. Published
benchmarks show 20-100x throughput improvements over JDBC/ODBC for analytical
workloads.

For block-to-block communication in gRPC out-of-process mode:

```mermaid
flowchart LR
    subgraph Go["Block A (Go)"]
        PA["Process\nbatch"]
    end
    subgraph Python["Block B (Python)"]
        PB["Process\nbatch"]
    end
    PA -->|"Arrow Flight\nRecordBatch as Arrow IPC\nover gRPC stream"| PB
```

> Block authors write Arrow code only. No Protobuf schema. No codegen.

Key advantages over writing raw gRPC + Protobuf services:

- **No codegen step** — Arrow schemas are defined at runtime, not compile time.
  Adding a new field to a RecordBatch does not require regenerating stubs.
- **Native decimal support** — Arrow has `Decimal128` and `Decimal256` types.
  Protobuf has no decimal type; billing platforms using Protobuf resort to
  string-encoded decimals or integer cents, both of which are error-prone.
- **Columnar layout** — Arrow is columnar, which means analytical operations
  (sum a column, filter by value, group-by) operate on contiguous memory. This
  is 10-100x faster than row-oriented formats for aggregation workloads.
- **Zero-copy read** — The receiver can read Arrow data without copying it into
  application-specific structures.

### 5.3 Arrow IPC for WASM Block Boundaries (v2)

> **Phasing note:** WASM blocks are planned for v2.0. This section describes the
> target architecture. In v1.0, blocks that would use WASM run as gRPC/Arrow
> Flight processes instead (section 5.2), which provides stronger isolation at
> the cost of higher per-call overhead (~1-5ms vs. ~50-200μs estimated). See
> ADR-0006 for the phasing rationale.

WASM blocks will run in-process but in a sandboxed memory space. Data crosses
the WASM boundary via Arrow IPC format over shared memory:

```mermaid
flowchart LR
    subgraph Host["Host Process"]
        H_In["Arrow\nRecordBatch"]
        H_Out["Arrow\nRecordBatch"]
    end
    subgraph WASM["WASM Sandbox"]
        W_In["Arrow\nRecordBatch"]
        W_Process["Process..."]
        W_Out["Arrow\nRecordBatch"]
        W_In --> W_Process --> W_Out
    end
    H_In -->|"shared mem (IPC)"| W_In
    W_Out -->|"shared mem (IPC)"| H_Out
```

> Zero-copy: the WASM guest reads directly from shared memory. No serialization/deserialization per call.

The Extism WASM runtime provides shared memory regions that both host and guest
can access. Arrow IPC format is a simple framing of Arrow RecordBatches that can
be written to and read from a flat byte buffer — ideal for shared memory.

**Key risk:** Arrow IPC through WASM shared memory has not been battle-tested at
production scale. The `arrow-rs` crate compiles to WASM but zero-copy IPC
through the boundary is novel. v2 development will begin with a focused
prototype to validate performance and correctness before committing to full
implementation.

### 5.4 Why NOT Other Formats as the Developer-Facing Data Model

Note: Arrow Flight uses Protobuf internally for control plane messages. The
comparison below is about requiring **block developers** to write and maintain
Protobuf schemas as part of their block interface, which we reject.

**Developer-facing Protobuf (raw gRPC):**
- Requires `.proto` files and code generation per language for every block
- No native decimal type (critical for financial data)
- Row-oriented: poor for analytical aggregation
- Two schema systems to maintain if used alongside Arrow for data
- Tight coupling between schema and compiled code

**Cap'n Proto:**
- Limited language support — strong in C++, Rust, Go, Java, but no first-class
  Python binding, which excludes the largest data science ecosystem
- Small ecosystem compared to Arrow
- No columnar layout

**FlatBuffers:**
- Designed for random access to individual records (game assets, mobile apps)
- Not columnar — no benefit for batch analytical processing
- Would need a separate columnar format for storage/analytics anyway

**JSON:**
- No schema enforcement at the wire level
- No binary efficiency (10-100x larger than Arrow for numeric data)
- No zero-copy reads
- No native types for timestamps, decimals, nested structs
- Parsing overhead dominates processing time for high-throughput pipelines

### 5.5 The vgi-rpc Precedent

The vgi-rpc project (discussed on the Arrow dev mailing list) demonstrates a
pure Arrow RPC model that eliminates the gRPC dependency entirely, achieving
**29 GB/s** throughput over shared memory. While Meteridian uses Arrow Flight
(which builds on gRPC) for the v1 implementation, the vgi-rpc approach validates
the architectural direction: Arrow as both data format AND RPC format.

### 5.6 Field-Level Access Control via Arrow

Arrow's columnar layout enables a powerful security feature: **field-level
access control**. Because each column is a separate buffer, the platform can
expose only the columns that a block has declared in its capabilities:

```mermaid
flowchart TD
    Full["Full RecordBatch (platform)\ntenant_id · user_id · source_ip · timestamp · amount"]
    Projected["Block sees (fields.read)\nsource_ip · timestamp"]
    Full -->|"Schema projection"| Projected
```

> The block physically cannot access `tenant_id`, `user_id`, or `amount`.

This is enforced at the Arrow schema level: the platform constructs a projected
RecordBatch containing only the permitted columns before passing it to the
block. For WASM blocks, this means the sensitive columns never enter the
sandbox's memory space.

---

## 6. Plugin Runtime Model

### 6.1 Phased Runtimes, One Interface

All runtimes implement the same logical interface — receive Arrow RecordBatches,
process them, emit Arrow RecordBatches. The choice of runtime determines the
performance/isolation trade-off. See ADR-0006 for the phasing rationale.

```mermaid
flowchart TD
    subgraph MP["Meteridian Process"]
        subgraph V1["v1.0"]
            GoSDK["Go SDK · Block A\nIn-process · No barrier"]
            gRPCStub["gRPC Stub\n(Arrow Flight client)"]
            BlockC["Block C\n(Python, Java, Rust, etc.)"]
            gRPCStub -.->|"Arrow Flight"| BlockC
        end
        subgraph V2["v2.0 (planned)"]
            WASM["WASM (Extism)\nBlock B · Sandboxed Memory\nArrow IPC shared memory"]
        end
    end
```

### 6.2 Go SDK (Native)

For core platform blocks and performance-critical paths. Blocks compiled with
the Go SDK run in the same process as the Meteridian engine with zero
serialization overhead.

```go
package geocollector

import (
    "github.com/meteridian/sdk-go/block"
    "github.com/apache/arrow-go/v18/arrow"
    "github.com/apache/arrow-go/v18/arrow/array"
    "github.com/apache/arrow-go/v18/arrow/memory"
)

type GeoEnricher struct {
    block.Base
    db *maxmind.Reader
}

func (g *GeoEnricher) Init(ctx block.Context) error {
    dbPath := ctx.Config().GetString("database_path")
    var err error
    g.db, err = maxmind.Open(dbPath)
    return err
}

func (g *GeoEnricher) Process(ctx block.Context, batch arrow.Record) (arrow.Record, error) {
    alloc := memory.NewGoAllocator()

    ipCol := batch.Column(batch.Schema().FieldIndices("source_ip")[0])
    ips := ipCol.(*array.String)

    countryBuilder := array.NewStringBuilder(alloc)
    defer countryBuilder.Release()

    for i := 0; i < ips.Len(); i++ {
        record, err := g.db.City(net.ParseIP(ips.Value(i)))
        if err != nil {
            countryBuilder.AppendNull()
            continue
        }
        countryBuilder.Append(record.Country.IsoCode)
    }

    // Build output batch with original columns + new country_code column
    outputFields := append(batch.Schema().Fields(), arrow.Field{
        Name: "country_code", Type: arrow.BinaryTypes.String,
    })
    outputSchema := arrow.NewSchema(outputFields, nil)
    columns := make([]arrow.Array, batch.NumCols()+1)
    for i := 0; i < int(batch.NumCols()); i++ {
        columns[i] = batch.Column(i)
    }
    columns[len(columns)-1] = countryBuilder.NewArray()

    return array.NewRecord(outputSchema, columns, int64(ips.Len())), nil
}

func (g *GeoEnricher) Close() error {
    return g.db.Close()
}
```

**Characteristics:**
- Zero serialization overhead (shared memory space)
- Full access to Meteridian internal APIs
- Used for: official blocks, collectors, core rating engine, storage adapters
- Trust requirement: Tier 2+ (Verified Partner and Official)

### 6.3 WASM Runtime (Extism) — v2.0

> **Phasing note:** WASM support is deferred to v2.0. In v1.0, marketplace
> blocks and community contributions use gRPC/Arrow Flight (section 6.4), which
> provides process-level isolation. WASM is planned for v2 as an optimization
> for higher block density once the WASM+Arrow ecosystem matures.

For marketplace blocks that need in-process sandboxing with lower overhead than
gRPC. The Extism runtime provides capability-based security with near-native
performance.

```rust
// Rust WASM block using Extism PDK
use extism_pdk::*;
use arrow::array::{StringArray, RecordBatch};
use arrow::ipc::reader::StreamReader;
use arrow::ipc::writer::StreamWriter;

#[plugin_fn]
pub fn process(input: Vec<u8>) -> FnResult<Vec<u8>> {
    // Deserialize Arrow IPC from shared memory
    let reader = StreamReader::try_new(&input[..], None)?;
    let batches: Vec<RecordBatch> = reader.collect::<Result<Vec<_>, _>>()?;

    let mut output_batches = Vec::new();
    for batch in batches {
        // Access only declared fields (enforced by host)
        let ip_col = batch.column_by_name("source_ip")
            .unwrap()
            .as_any()
            .downcast_ref::<StringArray>()
            .unwrap();

        // Process and build output...
        let enriched = enrich_batch(&batch, ip_col)?;
        output_batches.push(enriched);
    }

    // Serialize back to Arrow IPC
    let mut buf = Vec::new();
    let mut writer = StreamWriter::try_new(&mut buf, &output_batches[0].schema())?;
    for batch in &output_batches {
        writer.write(batch)?;
    }
    writer.finish()?;

    Ok(buf)
}
```

**Characteristics:**
- In-process execution with full memory isolation
- ~50-200μs overhead per call (Arrow IPC serialize/deserialize) — **estimated,
  not yet benchmarked** (see ADR-0006 risk section)
- Capability-based security: network, filesystem, field access all deny-by-default
- Languages: Rust, TinyGo, C, AssemblyScript, Zig (any language targeting WASM)
- Used for: marketplace blocks, community contributions, tenant-specific logic
- Trust requirement: All tiers (v2+ only)
- **Availability:** v2.0 (in v1.0, marketplace tiers 0-2 use gRPC/Arrow Flight; Tier 3 uses Go SDK or gRPC)

### 6.4 gRPC Out-of-Process (Arrow Flight)

For blocks requiring full language runtimes (Python with NumPy/Pandas, Java with
JVM libraries) or heavy compute (ML inference).

```python
# Python gRPC block using Arrow Flight
import pyarrow as pa
import pyarrow.flight as flight

class GeoEnricherFlightServer(flight.FlightServerBase):
    """Arrow Flight server implementing the block interface."""

    def __init__(self, location, **kwargs):
        super().__init__(location, **kwargs)
        self.db = geoip2.database.Reader('/data/GeoLite2-City.mmdb')

    def do_exchange(self, context, descriptor, reader, writer):
        """Process Arrow RecordBatches via bidirectional streaming."""
        for chunk in reader:
            batch = chunk.data

            # Read source_ip column
            ips = batch.column("source_ip")

            # Enrich with geolocation
            countries = []
            regions = []
            for ip in ips:
                try:
                    record = self.db.city(str(ip))
                    countries.append(record.country.iso_code)
                    regions.append(record.subdivisions.most_specific.name)
                except Exception:
                    countries.append(None)
                    regions.append(None)

            # Build enriched batch
            enriched = batch.append_column("country_code", pa.array(countries))
            enriched = enriched.append_column("region", pa.array(regions))

            writer.write_batch(enriched)

if __name__ == "__main__":
    server = GeoEnricherFlightServer("grpc://0.0.0.0:8815")
    server.serve()
```

**Characteristics:**
- Strongest isolation (separate process or container)
- ~1-5ms overhead per call (network + Arrow Flight serialization)
- Any language with Arrow Flight support (Python, Java, Node.js, Ruby, C++, Rust, Go, C#)
- Can run in separate containers for resource isolation (CPU/memory limits)
- Used for: ML inference blocks, data science workloads, legacy system adapters,
  all marketplace blocks (v1)
- Trust requirement: All tiers (v1 and v2+)

### 6.5 Performance Comparison

| Mechanism | Latency overhead | Memory isolation | Language support | Best for |
|-----------|-----------------|------------------|-----------------|----------|
| Go SDK | ~0 (in-process) | None (shared address space) | Go only | Core platform blocks, collectors, storage |
| WASM (Extism) | ~50-200μs per call * | Full sandbox (linear memory) | Rust, TinyGo, C, AssemblyScript, Zig | Marketplace blocks, tenant-specific transforms |
| gRPC (Arrow Flight) | ~1-5ms per call | Process/container boundary | Python, Java, Node.js, Ruby, C++, any | Heavy compute, ML inference, legacy adapters |

\* WASM latency is estimated, not yet benchmarked. See ADR-0006 risk section.

### 6.6 Runtime Selection Guidance

**v1 (Go SDK + gRPC only):**

```mermaid
flowchart TD
    Q1{"Core Meteridian\nblock?"} -->|Yes| GoSDK1["Go SDK\n(Tier 3)"]
    Q1 -->|No| Q2{"Verified Partner (Tier 2)\nwanting in-process?"}
    Q2 -->|Yes| GoSDK2["Go SDK\n(after code review +\nsecurity audit)"]
    Q2 -->|No| Q3{"Needs GPU?"}
    Q3 -->|Yes| gRPCGPU["gRPC\n(separate container\nwith GPU access)"]
    Q3 -->|No| gRPC["gRPC (Arrow Flight)\nmarketplace tiers 0-2"]
```

**v2+ (Go SDK + gRPC + WASM):**

```mermaid
flowchart TD
    Q1{"Core Meteridian\nblock?"} -->|Yes| GoSDK["Go SDK\n(Tier 3)"]
    Q1 -->|No| Q2{"Needs Python/Java/\nfull runtime or GPU?"}
    Q2 -->|Yes| gRPC["gRPC\n(Arrow Flight)"]
    Q2 -->|No| Q3{"Lightweight marketplace\nblock (Tier 0-1)?"}
    Q3 -->|Yes| WASM["WASM preferred\n(lower overhead,\nbetter density)"]
    Q3 -->|No| Choice["gRPC or WASM\n(developer choice)"]
```

---

## 7. Observability and Performance

### 7.1 OpenTelemetry Instrumentation

Every block is automatically instrumented with OpenTelemetry. The platform
injects tracing, metrics, and logging without requiring block authors to write
instrumentation code.

```mermaid
flowchart LR
    subgraph Pipeline["Pipeline: ocp-cost-enrichment"]
        A["collector\np50: 45ms · p99: 120ms\n1.2k rec/s · err: 0.01%"] -->|"3.2ms"| B["enricher\np50: 2.1ms · p99: 8.5ms\n1.2k rec/s · err: 0.1%"]
        B -->|"0.8ms"| C["rater\np50: 0.3ms · p99: 1.2ms\n1.2k rec/s · err: 0%"]
    end
```

> Pipeline throughput: 1,200 records/sec
> End-to-end latency: p50=48ms, p95=85ms, p99=130ms

### 7.2 Per-Block Metrics

The following metrics are emitted for every block via OpenTelemetry:

**Latency metrics (histogram):**
- `meteridian.block.process_duration` — Time to process a single batch
  - Labels: `block_name`, `block_type`, `pipeline`, `tenant`
  - Buckets: p50, p75, p90, p95, p99

**Throughput metrics (counter):**
- `meteridian.block.records_in` — Total records received
- `meteridian.block.records_out` — Total records emitted
- `meteridian.block.batches_processed` — Total batches processed

**Error metrics (counter):**
- `meteridian.block.errors` — Total errors by type
  - Labels: `error_type` (transient, permanent, panic)

**Resource metrics (gauge):**
- `meteridian.block.memory_bytes` — Current memory usage
- `meteridian.block.cpu_seconds` — CPU time consumed

### 7.3 SpanMetricsConnector

Meteridian uses the OpenTelemetry Collector's SpanMetricsConnector to derive
RED metrics (Rate, Error, Duration) from traces without requiring explicit
metric instrumentation:

```mermaid
flowchart LR
    A["Block traces"] --> B["OTel Collector"] --> C["SpanMetricsConnector"]
    C -->|"calls_total (rate)\nduration_bucket (histogram)\nerrors_total (error count)"| D["Prometheus/Mimir"]
```

This means block authors get full observability for free — the platform extracts
metrics from the traces it already generates.

### 7.4 Performance Tuning Mode

For debugging performance bottlenecks, blocks can be put into a detailed
profiling mode that adds per-field timing:

```yaml
apiVersion: meteridian.io/v1
kind: Pipeline
metadata:
  name: ocp-cost-enrichment
  annotations:
    meteridian.io/performance-tuning: "true"  # Enable detailed profiling
```

In tuning mode, the platform records:
- Per-column read time (which fields is the block spending time on?)
- Serialization/deserialization overhead per batch
- Memory allocation patterns (allocation rate, GC pressure)
- Queue depth between blocks (where is backpressure building?)

### 7.5 Block-Level SLOs

Operators can define per-block SLOs:

```yaml
blocks:
  - name: cost-rater
    type: meteridian/tiered-rater
    slo:
      max_latency_p99: 10ms
      max_error_rate: 0.1%
      min_throughput: 1000  # records/sec
```

When a block violates its SLO, the platform emits alerts and can optionally
auto-scale the block (for gRPC blocks running in containers) or route traffic
to a fallback block.

---

## 8. AI-First Extensibility

### 8.1 Design Philosophy

Instead of investing 6-12 months building a visual editor as the primary
interface, Meteridian treats **AI agents as first-class users**. The platform
exposes structured tool interfaces that AI agents can use to create, configure,
debug, and optimize pipelines through natural language.

This is not "AI as an afterthought" — it is the primary interaction model:

```mermaid
flowchart LR
    Agent["AI Agent\n(Claude, ChatGPT,\nCursor, Kimi)"] <-->|"MCP / A2A\n'Create a pipeline that...'"| Server["Meteridian\nMCP Server + A2A Agent\n\nTools:\nlist_blocks · create_block\ncreate_pipeline · optimize_pipe\ndebug_block · get_metrics"]
```

### 8.2 Model Context Protocol (MCP)

Meteridian exposes an MCP server that any compatible AI agent can connect to.
The MCP server provides tools for pipeline management:

**Available MCP Tools:**

| Tool | Description | Parameters |
|------|------------|------------|
| `list_blocks` | List available blocks in marketplace and installed | `filter`, `category`, `runtime` |
| `create_block` | Scaffold a new block project | `name`, `runtime`, `input_schema`, `output_schema` |
| `get_block_metrics` | Get performance metrics for a running block | `block_name`, `pipeline`, `time_range` |
| `create_pipeline` | Create a new pipeline from block connections | `name`, `blocks`, `connections` |
| `validate_pipeline` | Check pipeline schema compatibility and capabilities | `pipeline_yaml` |
| `deploy_pipeline` | Deploy a pipeline to a target environment | `pipeline`, `environment`, `canary_percent` |
| `optimize_pipeline` | Analyze pipeline and suggest optimizations | `pipeline`, `objective` (latency/throughput/cost) |
| `debug_block` | Diagnose issues with a specific block | `block_name`, `pipeline`, `symptoms` |
| `list_pipelines` | List all pipelines with status | `tenant`, `status_filter` |
| `get_pipeline_topology` | Get the DAG structure of a pipeline | `pipeline` |
| `update_block_config` | Update a running block's configuration | `block_name`, `pipeline`, `config_patch` |

**Example MCP interaction:**

```
User: "My OCP cost pipeline is slow. The enrichment step takes too long."

AI Agent → Meteridian MCP:
  Tool: get_block_metrics
  Params: { block_name: "geo-enricher", pipeline: "ocp-cost-enrichment", time_range: "1h" }

Meteridian → AI Agent:
  Result: {
    "p50_ms": 45.2,
    "p99_ms": 312.5,
    "records_per_sec": 850,
    "error_rate": 0.02,
    "memory_mb": 256,
    "bottleneck": "network_latency_to_maxmind_api"
  }

AI Agent → Meteridian MCP:
  Tool: optimize_pipeline
  Params: { pipeline: "ocp-cost-enrichment", objective: "latency" }

Meteridian → AI Agent:
  Result: {
    "suggestions": [
      {
        "action": "enable_cache",
        "block": "geo-enricher",
        "config_change": { "cache_ttl": "3600s", "cache_max_entries": 100000 },
        "estimated_improvement": "60-80% reduction in p99 latency"
      },
      {
        "action": "replace_block",
        "current": "acme-corp/geo-enricher",
        "suggested": "meteridian/geo-enricher-embedded",
        "reason": "Uses embedded MaxMind database instead of API calls",
        "estimated_improvement": "95% reduction in latency"
      }
    ]
  }
```

### 8.3 Agent-to-Agent (A2A)

Meteridian also acts as an autonomous A2A agent. External agents can request
collaboration for complex tasks that go beyond simple tool calls:

**Agent card** (published at `/.well-known/agent-card.json`):

```json
{
  "name": "Meteridian",
  "description": "Metering and billing platform with block-based extensibility",
  "url": "https://api.meteridian.io",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "pipeline-management",
      "name": "Pipeline Management",
      "description": "Create, configure, and deploy metering pipelines"
    },
    {
      "id": "cost-optimization",
      "name": "Cost Optimization",
      "description": "Analyze and optimize billing pipeline performance and cost"
    },
    {
      "id": "block-development",
      "name": "Block Development",
      "description": "Scaffold, test, and publish processing blocks"
    },
    {
      "id": "billing-analysis",
      "name": "Billing Analysis",
      "description": "Query and analyze metering data, usage patterns, and cost trends"
    }
  ],
  "authentication": {
    "schemes": ["bearer"]
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"]
}
```

### 8.4 Reference Implementation: Kubernaut

The MCP and A2A implementation follows patterns established by the **Kubernaut**
project (github.com/jordigilh/kubernaut), which demonstrates:

- **Centralized tool registry** (`mcpToolRegistry`): All MCP tools are
  registered in a single registry. The registry provides tool discovery, schema
  validation, and execution dispatch.

- **Agent card generation from tool registry** (`DefaultAgentSkills`): A2A agent
  skills are automatically derived from the MCP tool registry, ensuring the MCP
  and A2A interfaces are always in sync.

- **HTTP router with middleware**: Authentication, CORS, and audit logging are
  applied uniformly to both MCP and A2A endpoints via Go HTTP middleware.

- **Go SDK integration**: Kubernaut uses the official Go SDKs:
  - `github.com/modelcontextprotocol/go-sdk/mcp` for MCP
  - `github.com/a2aproject/a2a-go/a2a` for A2A

Meteridian adopts this pattern, extending it with pipeline-specific tools and
billing domain knowledge.

### 8.5 Pipeline Dashboard (v1) and Visual Editor (v2+)

In **v1**, a read-only **pipeline dashboard** provides observability:

- Pipeline topology visualization (auto-generated from pipeline TOML)
- Real-time metrics overlay (per-block latency, throughput, error rate)
- Block health status indicators
- This is part of the observability story, not an editing tool

In **v2+**, a full **visual editor** extends the dashboard with:

- Node-RED-inspired drag-and-drop block placement
- Bidirectional sync with pipeline TOML (the TOML is the source of truth)
- Marketplace block browser integrated into the canvas
- Built with React Flow or similar library

The AI-first interface (MCP + A2A) remains the primary interface for pipeline
creation and modification in both v1 and v2.

---

## 9. Marketplace and Developer Program

### 9.1 Tiered Trust Model

The marketplace uses a four-tier trust model that balances openness with
security:

| Tier | Name | Requirements | Runtime (v1) | Runtime (v2+) | Capabilities |
|------|------|-------------|---|---|-------------|
| 0 | Unverified | None (anyone can publish) | gRPC | gRPC or WASM | Minimal: no network, no filesystem, no secrets, no state, declared fields only |
| 1 | Community Verified | Signed + SLSA provenance | gRPC | gRPC or WASM | Standard: DNS resolution, declared network hosts, state:read/write, metrics:write |
| 2 | Verified Partner | Developer program agreement + code review + security audit | gRPC or Go SDK | All | Extended: broader network, filesystem (scoped), environment variables, secrets, payments (with PCI-DSS review) |
| 3 | Official | Meteridian team | All (Go SDK, gRPC) | All (Go SDK, gRPC, WASM) | Full: all capabilities, internal APIs |

> **Design principle:** The tier system controls *capabilities* (what a block can
> access), not *runtimes* (how it is isolated). In v1, all marketplace blocks use
> gRPC (container-isolated). In v2, WASM provides an additional runtime option
> with better density for lightweight blocks. Container isolation (gRPC) provides
> equal or stronger security guarantees than WASM sandboxing — see
> [Section 11](#11-capability-based-block-security) for details.

Tier upgrades require:
- **0 → 1**: Sign the block with Cosign, provide SLSA provenance attestation,
  pass automated vulnerability scan (Trivy/Grype).
- **1 → 2**: Sign developer program agreement, submit to code review by
  Meteridian security team, pass penetration test for gRPC blocks.
- **2 → 3**: Meteridian team adoption (internal blocks only).

### 9.2 Software Supply Chain Security

Every block published to the marketplace goes through a supply chain security
pipeline:

```mermaid
flowchart LR
    Dev["Developer\nSource code"] -->|push| CI["CI/CD\n\n1. Build\n2. Test\n3. Sign\n4. Attest\n5. Scan"]
    CI -->|publish| MP["Marketplace\n\nVerify:\n- Signature\n- SLSA\n- Vuln scan\n- Schema"]
```

**SLSA (Supply-chain Levels for Software Artifacts):**
- Level 1 (Tier 0): Build process documented
- Level 2 (Tier 1): Hosted build service, signed provenance
- Level 3 (Tier 2): Hardened build platform, reproducible builds
- Level 3+ (Tier 3): All of Level 3, plus Meteridian team review and audit

See [ADR-0009](../../docs/adr/0009-slsa-sigstore-provenance.md) for the full
rationale, alternatives analysis, and tier-to-SLSA-level mapping.

**Sigstore (Cosign, Fulcio, Rekor):**
- **Cosign** signs block artifacts (WASM modules, container images)
- **Fulcio** issues short-lived certificates based on OIDC identity (no long-lived keys)
- **Rekor** provides a tamper-evident transparency log of all signatures

**in-toto attestation format:**
- Standardized metadata format for supply chain claims
- Links build inputs (source commit) to outputs (signed artifact)
- Enables automated policy enforcement

**Vulnerability scanning:**
- Every published block is scanned with Trivy and Grype
- Known CVEs block publication until patched
- Continuous rescanning of published blocks (daily)
- Automatic delisting if critical CVE found and not patched within 7 days
  (existing installations continue running but with a security warning in the
  pipeline dashboard)

Build hardening templates are documented in [METR-0006 §8](../0006-developer-experience/developer-experience.md)
(GitHub Actions, GitLab CI, and optional Konflux/Tekton for OpenShift) and
verification rules in [ADR-0009](../../docs/adr/0009-slsa-sigstore-provenance.md).
OpenShift/Konflux builders are supported when they produce equivalent SLSA Build
Level 3 attestations; the marketplace verifies attestations, not which CI platform
ran the build.

### 9.3 Marketplace Revenue Model

| Aspect | Detail |
|--------|--------|
| Default payment processor | Paddle as Merchant of Record |
| Transaction fee | 5% + $0.50 (all-in, including global tax) |
| Revenue split | 80% developer / 20% platform |
| Free tier | Open-source blocks (OSI-approved licenses) are free to list |
| Enterprise tier | Private marketplace for internal blocks (flat annual fee) |
| Payout schedule | Monthly, net-30 |
| Minimum payout | $50 |

**Pricing models for block publishers:**
- Free (open-source)
- One-time purchase
- Monthly subscription
- Usage-based (per-record processed)
- Freemium (free tier + paid premium features)

### 9.4 FinOps Action Queue Pattern (Future)

> **Status:** Phase 2+ / METR-TBD. Not in v1 core scope. See
> [ROADMAP.md](../../ROADMAP.md#finops-action-queue-phase-2-metr-tbd).

METR-0011 (Limitador integration) provides **hard enforcement** — quota
exceeded, request denied. Many FinOps actions are **discretionary**: rightsizing
a production workload, re-tagging shared resources, or approving a budget
exception requires human judgment before execution.

The **FinOps action queue** pattern complements hard enforcement:

| Layer | Mechanism | Example |
|-------|-----------|---------|
| Automated | METR-0011 / Limitador | Block API call when prepaid balance is zero |
| Discretionary | Action queue + approval blocks | Route GPU rightsizing opportunity to owner via Slack; proceed only after Jira approval |

**Planned marketplace blocks** (Phase 2+):

- **Slack / Jira approval sink** — post opportunity summary, collect approve/dismiss/snooze responses, resume pipeline on approval
- **Opportunity case state** — Postgres-backed lifecycle (Created, Under review, Dismissed, Snoozed); not a separate operational database like OpenOps Tables
- **Tag-owner / BU reference data** — admin UX for virtual tags and cost-group ownership (similar to OpenOps tag-owner mapping, implemented natively)

Design inspiration comes from [OpenOps](../../docs/competitive/competitive.md#finops-and-adjacent-tools)
Opportunities workflow. OpenOps is adjacent tooling, not a Meteridian dependency —
the pattern is adoptable via blocks without adopting Baserow or Superset.

Cross-reference: [METR-0011](../0011-enforcement-integration/enforcement-integration.md)
(automated enforcement), future METR-TBD (action queue METR).

---

## 10. Pluggable Payment Providers

### 10.1 Two Dimensions of Payment

Payment processing in the Meteridian ecosystem operates on two independent
dimensions:

**Dimension 1: Marketplace payments (developer payouts)**
How block developers get paid for their marketplace listings. This is platform
infrastructure, not customer-facing.

**Dimension 2: Customer billing (end-user invoicing)**
How Meteridian customers bill their own end users for metered usage. This is a
pluggable block interface that customers choose and configure.

```mermaid
flowchart TD
    subgraph D1["Dimension 1: Marketplace"]
        BD["Block Developer"] -->|"publishes block"| MKT["Marketplace"]
        MKT --> Paddle["Paddle (MoR)"]
        Paddle -->|"payout"| Bank["Developer\nbank account"]
    end
    subgraph D2["Dimension 2: Customer Billing"]
        Cust["Meteridian Customer"] -->|"rated usage"| PPB["Payment Provider Block\n(Stripe, Paddle,\nAdyen, custom)"]
        PPB -->|"invoice + pay"| EU["End User"]
    end
```

### 10.2 Marketplace Payment Providers

Paddle is the default Merchant of Record for marketplace transactions (zero tax
compliance burden for the platform and developers). Alternative providers
(Stripe Connect, LemonSqueezy, Adyen) are available for Verified Partners
(Tier 2+). See [ADR-0010](../../docs/adr/0010-pluggable-payment-providers.md)
for the full rationale, fee comparison, and alternatives analysis.

### 10.3 Customer Billing: PaymentProviderBlock Interface

For customer billing, the payment provider is a standard **Transform** block
(input: invoices, output: payment results). The interface receives rated
invoices as Arrow RecordBatches and emits payment status:

> **Security constraint:** Only Tier 2+ (Verified Partner) blocks may implement
> the PaymentProviderBlock interface. Implementors must undergo PCI-DSS
> compliance review before marketplace publication. Payment blocks handle
> sensitive financial data (card tokens, billing addresses, transaction amounts)
> and require the `payments` capability, which is restricted to Tier 2+.
> See [ADR-0010](../../docs/adr/0010-pluggable-payment-providers.md).

```python
class PaymentProviderBlock(Block):
    """Standard interface for payment provider blocks.

    Receives rated invoices, initiates payment collection,
    and reports payment status.
    """

    input_schema = pa.schema([
        ("invoice_id", pa.string()),
        ("tenant_id", pa.string()),
        ("customer_id", pa.string()),
        ("amount", pa.decimal128(15, 2)),
        ("currency", pa.string()),
        ("due_date", pa.date32()),
        ("line_items", pa.list_(pa.struct([
            ("description", pa.string()),
            ("quantity", pa.decimal128(15, 4)),
            ("unit_price", pa.decimal128(15, 6)),
            ("amount", pa.decimal128(15, 2)),
            ("metadata", pa.map_(pa.string(), pa.string())),
        ]))),
        ("billing_address", pa.struct([
            ("country", pa.string()),
            ("state", pa.string()),
            ("postal_code", pa.string()),
        ])),
        ("payment_method_token", pa.string()),
    ])

    output_schema = pa.schema([
        ("invoice_id", pa.string()),
        ("payment_status", pa.string()),  # pending, succeeded, failed, refunded
        ("provider_reference", pa.string()),
        ("settled_amount", pa.decimal128(15, 2)),
        ("settled_currency", pa.string()),
        ("fee_amount", pa.decimal128(15, 2)),
        ("error_code", pa.string()),
        ("error_message", pa.string()),
        ("processed_at", pa.timestamp("us", tz="UTC")),
    ])
```

Meteridian ships with built-in payment provider blocks:

| Block | Provider | Features |
|-------|----------|----------|
| `meteridian/stripe-payment` | Stripe | Cards, ACH, SEPA, invoicing, subscriptions |
| `meteridian/paddle-payment` | Paddle | MoR, global tax, chargebacks handled |
| `meteridian/adyen-payment` | Adyen | Enterprise, interchange++, local payment methods |
| `meteridian/paypal-payment` | PayPal | PayPal, Venmo, Pay Later |

Third parties can publish custom payment blocks on the marketplace (e.g.,
regional payment methods, crypto payments, BNPL providers).

### 10.4 Fee Comparison

> Detailed fee comparison (Stripe vs Adyen vs Paddle vs LemonSqueezy vs PayPal)
> and the rationale for choosing Paddle MoR as the marketplace default are
> documented in [ADR-0010](../../docs/adr/0010-pluggable-payment-providers.md).

---

## 11. Capability-Based Block Security

### 11.1 Deny-by-Default Model

Every block operates under a **deny-by-default** security model. Unless a
capability is explicitly declared in the block manifest and approved by the
platform (based on the block's trust tier), the capability is denied.

```mermaid
flowchart LR
    BM["Block manifest"] --> PV["Platform validator"]
    PV -->|"1. Check trust tier\n2. Check capability limits\n3. Check tenant policy\n4. Log decision (audit trail)"| GD["Grant / Deny"]
```

### 11.2 Capability Categories

```toml
# block.toml — capability declaration

[capabilities]

# Network access: specific hosts and ports only
network = [
    "api.maxmind.com:443",
    "geoip.internal.acme.com:8080",
]

# Arrow field access: read and write permissions
fields.read = ["source_ip", "timestamp", "event_type"]
fields.write = ["country_code", "region", "city", "latitude", "longitude"]

# Environment variables: specific names only
environment = ["MAXMIND_API_KEY", "ACME_AUTH_TOKEN"]

# Filesystem: specific paths only (gRPC blocks)
filesystem.read = ["/data/geoip/*.mmdb"]
filesystem.write = ["/tmp/cache/"]

# External service access: specific URLs
services = [
    "https://api.maxmind.com/geoip/v2.1/city",
]

# State store: opt-in to platform-managed state
state = { enabled = true, max_size_mb = 100 }

# Secrets: reference to platform-managed secrets
secrets = ["maxmind-api-key"]

# Payments: permission to implement PaymentProviderBlock interface
# Restricted to Tier 2+ blocks with PCI-DSS compliance review
payments = { enabled = true }
```

### 11.3 Enforcement by Runtime

| Capability | WASM (Extism) | gRPC (container) | Go SDK |
|-----------|--------------|-------------------|--------|
| Network | Host function allowlist | NetworkPolicy + iptables | Trusted (Tier 3) |
| Fields | Schema projection before IPC | Schema projection in Flight server | Trusted (Tier 3) |
| Environment | Explicit env var injection | Container env var injection | Trusted (Tier 3) |
| Filesystem | Not available (no FS in WASM) | Seccomp profile + mount restrictions | Trusted (Tier 3) |
| Secrets | Host function injection (decrypted) | Kubernetes Secret mount / env injection | Trusted (Tier 3) |
| Services | Host function proxy | Sidecar proxy + egress NetworkPolicy | Trusted (Tier 3) |
| Payments | Not available — PCI-DSS requires process-level isolation; WASM linear memory cannot guarantee cardholder data boundary separation | PaymentProviderBlock interface check (Tier 2+ only) | Trusted (Tier 3) |
| Memory | WASM linear memory limit | Container memory limit | Process memory limit |
| CPU | WASM fuel metering | Container CPU limit | Process CPU limit |

### 11.4 Capability Audit Trail

Every capability grant, denial, and exercise is logged:

```json
{
  "timestamp": "2026-06-18T14:30:00Z",
  "event": "capability_exercised",
  "block": "acme-corp/geo-enricher:1.2.0",
  "pipeline": "ocp-cost-enrichment",
  "tenant": "org1234567",
  "capability": "network",
  "target": "api.maxmind.com:443",
  "result": "allowed",
  "latency_ms": 45
}
```

---

## 12. Stateful Blocks

### 12.1 Platform-Managed State Store

By default, blocks are **stateless** — they process each batch independently.
Some blocks require state (counters, caches, session data, sliding windows). For
these, the platform provides a managed key-value store:

```mermaid
flowchart LR
    Block["Block\n\nctx.State().\nGet · Set · Incr"] <-->|"read/write"| Store["State Store\n\nValkey (fast)\n+\nTimescaleDB (persistent)"]
```

### 12.2 State Properties

| Property | Value |
|----------|-------|
| Scope | Per block instance, per tenant |
| Backing store (hot) | Valkey (in-memory, ~0.1ms latency) |
| Backing store (cold) | TimescaleDB (persistent, ~1-5ms latency) |
| Consistency | Eventual (async replication) or strong (sync, opt-in) |
| Max size | Configurable per block (default: 100MB) |
| Expiration | Per-key TTL supported |
| Serialization | Arrow IPC or raw bytes |

### 12.3 State Lifecycle

- State is **preserved across block restarts** (Valkey persistence + TimescaleDB
  backup).
- State is **NOT preserved across major version upgrades** by default. Blocks
  can implement a `Migrate(oldVersion, newVersion)` hook to transform state
  during upgrades.
- State is **deleted** when a block is removed from a pipeline.
- State is **tenant-isolated** — blocks cannot access another tenant's state.

### 12.4 State API

```go
// Go SDK state API
func (b *WindowedAggregator) Process(ctx block.Context, batch arrow.Record) (arrow.Record, error) {
    state := ctx.State()

    // Get current window counter
    count, err := state.GetInt64("window_count")
    if err != nil {
        count = 0
    }

    // Update counter
    count += int64(batch.NumRows())
    state.SetInt64("window_count", count)

    // Check if window is complete
    if count >= b.windowSize {
        // Emit aggregated batch
        result := b.flushWindow(ctx)
        state.SetInt64("window_count", 0)
        return result, nil
    }

    // Buffer batch for later emission
    state.Append("window_buffer", batch)
    return nil, nil  // No output yet
}
```

---

## 13. Schema Evolution

### 13.1 Backward-Compatible Changes

Arrow schemas support the following backward-compatible changes:

- **Adding nullable columns**: New columns with `nullable=true` can be added to
  a block's output schema. Downstream blocks that don't declare the new column
  in their input schema simply don't see it.

- **Widening numeric types**: `int32` → `int64`, `float32` → `float64`. The
  platform handles type promotion automatically.

### 13.2 Breaking Changes

The following are breaking changes that require a new major version:

- Removing a column from the output schema
- Changing a column's type in an incompatible way (e.g., `string` → `int64`)
- Renaming a column
- Making a nullable column non-nullable

### 13.3 Version Compatibility

```mermaid
flowchart LR
    A1["Block A v1.0"] -->|"✓ Compatible\n(same schema)"| B1["Block B v1.0"]
    A2["Block A v1.1"] -->|"✓ Compatible\n(A added nullable col)"| B2["Block B v1.0 "]
    A3["Block A v2.0"] -->|"✗ Incompatible\n(A removed a col)"| B3[" Block B v1.0"]

    style A3 stroke:#f44,stroke-width:2px
    style B3 stroke:#f44,stroke-width:2px
```

Each block declares the minimum schema version it supports:

```toml
[input]
min_version = "1.0"  # Accepts any schema >= 1.0

[output]
version = "1.1"      # Current output schema version
```

### 13.4 Pipeline Validation

At deployment time, the platform validates schema compatibility across all
connections:

1. For each connection `A → B`, check that `A.output_schema` is a superset of
   `B.input_schema` (all required columns present with compatible types).
2. If validation fails, the deployment is rejected with a clear error message
   indicating which columns are missing or incompatible.
3. Optional columns (nullable) in `B.input_schema` are allowed to be absent
   from `A.output_schema` — they will be filled with null values.

---

## 14. Error Handling and Delivery Guarantees

Meteridian provides **at-least-once delivery** with **idempotency keys** to
achieve effectively-once semantics for financial data. See ADR-0011 for the
full rationale and ADR-0012 for tenant isolation within batches.

### 14.1 Error Classification

Every error in a block is classified into one of two categories:

| Category | Behavior | Examples |
|----------|----------|---------|
| **Transient** | Automatic retry with exponential backoff | Network timeout, rate limit, temporary unavailability |
| **Permanent** | Route to dead-letter queue (DLQ) | Invalid data, schema mismatch, business logic rejection |

### 14.2 Block Crash Recovery

When a block crashes (panic, OOM, segfault in WASM):

```mermaid
flowchart LR
    Crash["Block\ncrashes"] --> Detect["Platform detects\nhealth failure"]
    Detect --> Restart["Block restarts\n(fresh init)"]
    Restart --> Resume["Block resumes\nfrom last checkpoint"]
    Detect --> BP["Backpressure applied\nto upstream blocks"]
```

1. The platform detects the crash via health check failure.
2. Upstream blocks receive backpressure (stop sending batches).
3. The block is restarted with a fresh Init cycle.
4. If the block is stateful, state is recovered from the state store.
5. Processing resumes from the last successfully processed batch.

### 14.3 Dead-Letter Queue (DLQ)

Records that fail permanent errors are routed to a per-pipeline DLQ:

```json
{
  "original_record": { /* Arrow RecordBatch as JSON */ },
  "error": {
    "block": "geo-enricher",
    "code": "LOOKUP_FAILED",
    "message": "IP address 10.0.0.1 is not routable",
    "timestamp": "2026-06-18T14:30:00Z"
  },
  "pipeline": "ocp-cost-enrichment",
  "tenant": "org1234567",
  "retry_count": 3
}
```

DLQ records can be reprocessed after the underlying issue is fixed (e.g., after
updating the GeoIP database). See ADR-0011 for the full DLQ specification.

### 14.4 Idempotency and At-Least-Once Delivery

Meteridian does NOT provide exactly-once semantics (impossible in distributed
systems without unacceptable latency). Instead, the platform provides
**at-least-once delivery + idempotency keys** (ADR-0011):

- Every CloudEvents metering event carries a unique `id` field (ADR-0004)
- Blocks that perform state-mutating operations (DB writes, balance updates)
  MUST check the idempotency key before applying changes
- The platform provides an `IdempotencyStore` interface backed by the block
  state store (section 12) for convenient deduplication
- Retry strategy: exponential backoff (base 1s, max 60s, jitter 0-500ms)
- After max retries (configurable, default 3), events route to the DLQ

**Multiple block instances:** A pipeline DAG allows multiple instances of the
same block TYPE. For example, a "sanitizer" block can appear at the beginning
AND end of a pipeline as two separate instances with independent state. This
avoids cycles (DAG constraint) while enabling multi-pass processing.

### 14.5 Tenant-Isolated Batches

Each Arrow RecordBatch contains data from exactly ONE tenant (ADR-0012). This
is a security requirement: a buggy block cannot accidentally leak data across
tenants. See ADR-0012 for the full rationale, batch sizing guidance, and the
deferred bin-packing optimization.

### 14.6 Pipeline-Level Circuit Breaker

If a block exceeds its error threshold, the platform can pause the entire
pipeline:

```yaml
blocks:
  - name: geo-enricher
    type: acme-corp/geo-enricher
    error_policy:
      max_error_rate: 5%        # Trigger circuit breaker above 5%
      evaluation_window: 60s    # Over a 60-second window
      action: pause_pipeline    # Or: skip_block, route_to_dlq
      cooldown: 300s            # Wait 5 minutes before retry
```

---

## 15. Hot Reload

### 15.1 Live Pipeline Updates

Pipelines support the following live updates without full restart:

**Adding a block:**
1. New block is initialized and validated.
2. Upstream block's output is forked to both old path and new path.
3. Once the new block catches up, the old path is removed.

**Removing a block:**
1. Block enters Draining state.
2. Upstream reconnects to the downstream block (bypassing the removed block).
3. Block completes draining and stops.

**Configuration change:**
1. New configuration is validated.
2. Block enters Draining state.
3. Block restarts with new configuration.
4. Processing resumes from the queue.

### 15.2 Rolling Updates

```mermaid
flowchart LR
    subgraph S1["100% v1.0"]
        V1A["v1.0"] --> OUT1[" "]
    end
    subgraph S2["50% v1.0 / 50% v1.1"]
        V1B["v1.0"] --> OUT2[" "]
        V1C["v1.1"] --> OUT2
    end
    subgraph S3["100% v1.1"]
        V1D["v1.1"] --> OUT3[" "]
    end
    S1 -.-> S2 -.-> S3
```

### 15.3 Canary Deployments

For block version upgrades, the platform supports canary deployments:

```yaml
deploy:
  block: geo-enricher
  from_version: "1.2.0"
  to_version: "1.3.0"
  canary:
    initial_percent: 5
    increment: 10
    interval: 300s  # 5 minutes between increments
    success_criteria:
      max_error_rate: 1%
      max_latency_p99_ms: 50
    rollback_on_failure: true
```

The platform routes a percentage of traffic through the new version, monitors
error rates and latency, and automatically rolls back if the canary fails.

---

## 16. Open Questions

The following questions remain open and will be resolved during implementation.
Some questions from earlier drafts have been resolved via ADRs:

- ~~Exactly-once semantics~~ → Resolved: at-least-once + idempotency (ADR-0011)
- ~~Tenant isolation in batches~~ → Resolved: single-tenant batches (ADR-0012)
- ~~Redpanda Connect boundary~~ → Resolved: two-layer architecture (ADR-0013)
- ~~WASM timing~~ → Resolved: deferred to v2 (ADR-0006)
- ~~Arrow Flight API surface~~ → Resolved: `DoExchange` bidirectional streaming (ADR-0005, Section 6.4)
- ~~GPU block support~~ → Resolved: gRPC runtime with GPU-enabled container (Section 6.6 decision tree)

Remaining open questions:

1. **WASM component model vs Extism (v2)** — The WASM component model
   (preview2) provides a standardized interface for WASM modules. Extism
   provides a higher-level plugin framework. This decision is deferred to v2
   (ADR-0006) when we evaluate WASM readiness.

2. **Pipeline versioning semantics** — Should pipelines use semantic versioning
   (semver), date-based versioning, or content-hash-based versioning? Semver is
   familiar but requires human judgment; content hashing is deterministic.

3. **Multi-cluster pipeline spanning** — Can a single pipeline span multiple
   Kubernetes clusters? This would enable edge-to-cloud pipelines where
   collection happens at the edge and aggregation happens in the cloud. The
   challenge is maintaining at-least-once semantics (ADR-0011) across cluster
   boundaries.

4. **Block dependency management** — Some blocks depend on shared libraries or
   databases (e.g., MaxMind GeoIP database). How should shared dependencies be
   managed? Options: bundled with block, platform-provided shared volume,
   external URL with caching.

5. **Multi-tenant pipeline isolation** — Within a single Meteridian deployment,
   how are pipelines isolated between tenants? Options: separate namespaces,
   separate processes, cgroup limits, WASM sandbox per tenant.

6. **Pipeline-as-Code GitOps** — Should pipeline YAML definitions be managed via
   GitOps (Flux, ArgoCD)? This is natural for Kubernetes-native deployments but
   adds complexity for non-Kubernetes environments.

---

## 17. Related Documents

### ADRs

- [ADR-0005](../../docs/adr/0005-arrow-flight-block-data-plane.md) — Arrow Flight as block data plane
- [ADR-0006](../../docs/adr/0006-hybrid-plugin-runtime.md) — Go SDK + gRPC now, WASM later
- [ADR-0007](../../docs/adr/0007-block-based-dataflow.md) — Block-based dataflow extensibility
- [ADR-0008](../../docs/adr/0008-ai-first-extensibility.md) — AI-first interface (MCP + A2A)
- [ADR-0009](../../docs/adr/0009-slsa-sigstore-provenance.md) — SLSA/Sigstore for marketplace provenance
- [ADR-0010](../../docs/adr/0010-pluggable-payment-providers.md) — Pluggable payment providers
- [ADR-0011](../../docs/adr/0011-at-least-once-idempotency.md) — At-least-once + idempotency keys
- [ADR-0012](../../docs/adr/0012-tenant-isolated-batches.md) — Tenant-isolated event batches
- [ADR-0013](../../docs/adr/0013-two-layer-data-architecture.md) — Two-layer architecture (Redpanda Connect + blocks)

### Related Enhancements

- [METR-0001](../0001-architecture/architecture.md) — Platform Architecture
- [METR-0003](../0003-product-catalog/product-catalog.md) — Product and Service Catalog
- [METR-0004](../0004-credit-token-billing/credit-token-billing.md) — Credit, Prepaid, and Token Billing
- [METR-0005](../0005-internal-token-economy/internal-token-economy.md) — Internal Budget Units and Chargeback
- [METR-0006](../0006-developer-experience/developer-experience.md) — Developer Experience and Tooling

## 18. External References

- [Apache Arrow Flight](https://arrow.apache.org/docs/format/Flight.html) — RPC
  framework built on Arrow IPC and gRPC
- [Extism](https://extism.org/) — Cross-language WASM plugin framework with
  capability-based security
- [vgi-rpc](https://www.mail-archive.com/dev@arrow.apache.org/msg34470.html) —
  Arrow-only RPC achieving 29 GB/s over shared memory
- [Model Context Protocol](https://modelcontextprotocol.io/) — Standard protocol
  for AI agents to interact with tools
- [A2A Protocol](https://github.com/a2aproject/a2a-go) — Agent-to-Agent
  communication protocol
- [Kubernaut](https://github.com/jordigilh/kubernaut) — Reference implementation
  of MCP + A2A gateway with Go SDK
- [SLSA](https://slsa.dev/) — Supply-chain Levels for Software Artifacts
- [Sigstore](https://www.sigstore.dev/) — Keyless signing and transparency for
  software artifacts
- [in-toto](https://in-toto.io/) — Framework for securing the integrity of
  software supply chains
- [GoRules ZEN Engine](https://gorules.io/) — Business rules engine used
  inside the built-in rating block to evaluate pricing rules (see ADR-0003)
- [Redpanda Connect](https://www.redpanda.com/connect) — Stream processing
  framework with composable processors
- [Node-RED](https://nodered.org/) — Flow-based visual programming for event
  processing
- [Apache NiFi](https://nifi.apache.org/) — Dataflow automation between systems
- [Apache Beam](https://beam.apache.org/) — Unified batch and stream processing
  model
- [Cosign](https://docs.sigstore.dev/cosign/overview/) — Container and artifact
  signing
- [Fulcio](https://docs.sigstore.dev/fulcio/overview/) — Free certificate
  authority for code signing
- [Rekor](https://docs.sigstore.dev/rekor/overview/) — Transparency log for
  supply chain metadata
- [Paddle](https://www.paddle.com/) — Merchant of Record for software
- [LemonSqueezy](https://www.lemonsqueezy.com/) — Merchant of Record
  alternative
