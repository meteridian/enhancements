# METR-0006: Developer Experience and Tooling

- **Status:** provisional
- **Authors:** @pgarciaq, @jordigilh
- **Created:** 2026-06-18
- **Last Updated:** 2026-06-18
- **Depends on:** METR-0002 (Platform Extensibility)
- **Related:** METR-0003 (Product Catalog), METR-0004 (Credit, Prepaid, and Token Billing), METR-0005 (Internal Budget Units)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Meteridian CLI](#3-meteridian-cli)
4. [Local Development Emulator](#4-local-development-emulator)
5. [Block SDK](#5-block-sdk)
6. [Testing Framework](#6-testing-framework)
7. [Documentation](#7-documentation)
8. [CI/CD Integration](#8-cicd-integration)
9. [Performance Profiling](#9-performance-profiling)
10. [Block Scaffolding Templates](#10-block-scaffolding-templates)
11. [Marketplace Publishing Workflow](#11-marketplace-publishing-workflow)
12. [Versioning and Compatibility](#12-versioning-and-compatibility)
13. [Open Questions](#13-open-questions)
14. [References](#14-references)

---

## 1. Summary

Meteridian provides a first-class developer experience for block authors through an
integrated suite of tools that spans the entire lifecycle of block development — from
initial scaffolding to marketplace publication. This enhancement defines:

- A **CLI** (`meteridian`) that serves as the single entry point for all developer
  workflows: project initialization, building, testing, running, linting, benchmarking,
  profiling, and publishing.
- A **local development emulator** that faithfully simulates the Meteridian block
  runtime on a developer's machine — no Kubernetes cluster, no cloud credentials,
  no external dependencies.
- **Language-specific SDKs** (Go, Python, with Rust and TypeScript planned) that
  expose ergonomic, type-safe APIs for implementing blocks, all generated from a
  single Block Interface Specification.
- A **testing framework** covering unit, integration, performance, snapshot, and
  contract testing — each with tight CLI integration.
- **CI/CD integration** via official GitHub Actions, GitLab CI templates, and
  pre-commit hooks so that block quality gates are automated from day one.
- **Performance profiling tools** that surface per-block latency, throughput, and
  memory behavior during both local development and production operation.
- A **documentation suite** — tutorials, API references, cookbooks, and
  architecture guides — hosted at `docs.meteridian.io`.

The overarching goal is to make block development **delightful by default**: a new
developer should be able to go from zero to a tested, published block in under an
hour, with guardrails that prevent common mistakes and tools that reward best
practices.

---

## 2. Motivation

### 2.1 Why Developer Experience Determines Platform Success

Extensibility platforms live and die by the quality of their developer experience.
The technical sophistication of the runtime is irrelevant if third-party developers
cannot — or do not want to — build for it. History offers clear precedents:

- **Stripe** became the dominant payments platform not because its API was technically
  superior, but because its documentation, SDKs, and CLI were a generation ahead of
  competitors. Developers chose Stripe because it *felt good* to integrate with.
- **Terraform** built the largest infrastructure-as-code ecosystem because HashiCorp
  invested heavily in the Provider SDK, the acceptance testing framework, and the
  plugin development workflow. Writing a Terraform provider is straightforward even
  for developers who are not infrastructure experts.
- **VS Code** overtook established editors partly through its extension API, which
  provides a rich, well-documented, well-typed surface with built-in testing and
  debugging support. The barrier to entry for a VS Code extension is remarkably low.
- Conversely, platforms with powerful runtimes but poor DX — such as early OpenStack
  or certain Kubernetes operator frameworks — struggled to attract contributors despite
  significant backing.

### 2.2 The Current Gap

METR-0002 defines the block model, runtime semantics, capability system, and
lifecycle hooks. It does not, however, specify how a developer *actually* builds a
block. Without this METR, a block author would need to:

1. Read the runtime source code to understand the block interface.
2. Copy-paste from an existing block and modify.
3. Manually construct Arrow RecordBatches for testing.
4. Deploy to a full Meteridian cluster to verify behavior.
5. Hand-craft CI pipelines with no official guidance.

This friction compounds: each missing tool is not an isolated inconvenience, but a
multiplier on the overall pain of block development. The difference between "an
afternoon to build a block" and "a week to build a block" determines whether the
marketplace has ten blocks or ten thousand.

### 2.3 Design Principles

This enhancement is guided by five design principles:

1. **Zero-to-working in five minutes.** `meteridian init && meteridian run` should
   produce a running block with hot-reload, no configuration required.
2. **One tool, one binary.** The CLI is a single binary with no runtime dependencies.
   No JVMs, no interpreters, no Docker daemons required for basic workflows.
3. **Parity between local and production.** The emulator faithfully reproduces
   runtime behavior so that blocks tested locally behave identically in production.
4. **Progressive disclosure.** Simple blocks require simple code. Advanced features
   (custom state stores, multi-output routing, performance budgets) are opt-in and
   additive — they never complicate the happy path.
5. **Convention over configuration.** Sensible defaults everywhere: project
   structure, manifest fields, test setup, CI configuration. Override when needed,
   never when not.

---

## 3. Meteridian CLI

### 3.1 Overview

The `meteridian` CLI is a single statically-linked Go binary that serves as the
primary interface for all block development workflows. It is designed to be
self-contained: no plugins, no runtime dependencies, no Docker daemon required for
core commands.

**Distribution channels:**

| Method | Command |
|---|---|
| Homebrew (macOS/Linux) | `brew install meteridian` |
| Go install | `go install github.com/meteridian/cli@latest` |
| Direct download | Architecture-specific binaries from `releases.meteridian.io` |
| Container image | `ghcr.io/meteridian/cli:latest` (for CI environments) |

### 3.2 Command Reference

#### `meteridian init`

Scaffold a new block project.

```
meteridian init [--template=<template>] [--lang=<go|python|rust>] [--name=<block-name>]
```

- Creates project directory with canonical structure (see Section 10).
- Generates `block.toml` manifest with sensible defaults.
- Initializes Go module / Python package / Cargo project as appropriate.
- Creates initial test file with a passing example test.
- Generates `.github/workflows/ci.yml` for immediate CI setup.
- Runs `meteridian lint` to validate the scaffolded project.

#### `meteridian build`

Compile the block into a deployable artifact.

```
meteridian build [--output=<path>] [--target=<binary|wasm|container>]
```

- **Go blocks:** Cross-compiles to `linux/amd64` and `linux/arm64` by default.
  Output is a statically-linked binary.
- **Python blocks:** Packages into a container image with a locked dependency set
  (`pip-compile` output). The Dockerfile is generated automatically.
- **WASM blocks:** Compiles to a `.wasm` module via TinyGo or Rust `wasm32-wasi`
  target.
- **Container target:** Builds an OCI container image using Buildpacks or a
  generated multi-stage Dockerfile. No Docker daemon required — uses
  [ko](https://ko.build/) for Go and
  [buildpacks](https://buildpacks.io/) for Python.

#### `meteridian test`

Run block tests.

```
meteridian test [--unit] [--integration] [--update-snapshots] [--compare-baseline]
```

- Invokes the language-native test runner (`go test`, `pytest`) with the correct
  flags and environment.
- `--integration` starts the local emulator, deploys the block, sends test events,
  and verifies output.
- `--update-snapshots` regenerates snapshot baselines (see Section 6.4).
- `--compare-baseline` compares performance metrics against stored baselines and
  fails if regressions exceed configurable thresholds.

#### `meteridian run`

Start a block locally with hot-reload.

```
meteridian run [--port=<port>] [--emulator] [--debug]
```

- Launches the block process with file-watching enabled.
- On file change: rebuilds, restarts, and replays the last batch (for stateless
  blocks) or restores from the state checkpoint (for stateful blocks).
- `--emulator` starts the local emulator and connects the block to it.
- `--debug` starts the block with the debugger listener enabled (Delve for Go,
  `debugpy` for Python), prints the attach command.

#### `meteridian publish`

Publish a block to the Meteridian marketplace.

```
meteridian publish [--registry=<url>] [--sign] [--dry-run]
```

- Orchestrates the full publish workflow (see Section 11).
- `--dry-run` validates everything without uploading.
- `--sign` (default: true) signs the artifact with Sigstore (keyless signing via
  OIDC, as specified in ADR-0009).

#### `meteridian lint`

Validate block correctness.

```
meteridian lint [--verify-schema] [--strict]
```

- Validates `block.toml` manifest against the manifest JSON Schema.
- Checks capability declarations: are all declared capabilities actually used?
  Are any used but undeclared?
- `--verify-schema` runs the block against synthetic data and verifies that the
  actual output Arrow schema matches the declared schema in the manifest.
- `--strict` treats warnings as errors (recommended for CI).

#### `meteridian bench`

Run performance benchmarks.

```
meteridian bench [--duration=<seconds>] [--batch-size=<rows>] [--output=<json|table|markdown>]
```

- Generates synthetic Arrow RecordBatches matching the block's declared input schema.
- Measures: per-batch latency (p50, p95, p99), throughput (batches/sec, rows/sec),
  memory peak, allocation rate (bytes/op).
- `--output=json` for machine-readable results (CI integration).
- Results can be stored as baselines for regression detection.

#### `meteridian doctor`

Diagnose common development issues.

```
meteridian doctor
```

- Checks: Go, Python, and Rust toolchain versions, required dependencies, port
  availability (emulator ports), Docker or Podman availability (for container builds),
  network connectivity to marketplace registry.
- Outputs: pass or fail for each check, actionable fix suggestions for failures.
- Example output:
  ```
  ✓ Go 1.25.0 (>= 1.24 required)
  ✓ Python 3.13.1 (>= 3.11 required)
  ✗ Port 9090 in use (emulator default) — run: lsof -ti :9090 | xargs kill
  ✓ Sigstore cosign available
  ✓ Marketplace registry reachable
  ```

#### `meteridian profile`

Launch the real-time performance dashboard.

```
meteridian profile [--port=<port>]
```

- Starts a local web UI (default: `http://localhost:9091`) that displays real-time
  performance data from running blocks.
- See Section 9 for details.

### 3.3 Configuration

The CLI reads configuration from (in order of precedence):

1. Command-line flags
2. Environment variables (`METERIDIAN_*`)
3. Project-level `.meteridian.yaml`
4. User-level `~/.config/meteridian/config.yaml`

---

## 4. Local Development Emulator

### 4.1 Overview

The local development emulator is a lightweight, single-binary process that
simulates the Meteridian block runtime on a developer's machine. It enables
end-to-end block development without access to a Kubernetes cluster, cloud
services, or any external infrastructure.

**Inspired by:** LocalStack (AWS emulator), Stripe CLI (webhook testing), Firebase
Emulator Suite (multi-service local simulation).

### 4.2 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Emulator Process                        │
│                                                          │
│  ┌────────────┐   ┌──────────────┐   ┌───────────────┐  │
│  │  Synthetic  │──▶│  Block       │──▶│  Output       │  │
│  │  Event      │   │  Router      │   │  Collector    │  │
│  │  Generator  │   │              │   │  & Validator  │  │
│  └────────────┘   └──────┬───────┘   └───────────────┘  │
│                          │                               │
│  ┌────────────┐   ┌──────▼───────┐   ┌───────────────┐  │
│  │  State      │   │  Block       │   │  Observability│  │
│  │  Store      │◀──│  Sandbox     │──▶│  UI           │  │
│  │  (in-memory)│   │  (process)   │   │  (web)        │  │
│  └────────────┘   └──────────────┘   └───────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Components:**

- **Synthetic Event Generator:** Produces Arrow RecordBatches according to
  configurable schemas, rates, and distributions. Supports deterministic replay
  from recorded data files (Parquet or Arrow IPC).
- **Block Router:** Manages block lifecycle (init → process → close), routes
  batches to the correct block instance, handles errors and retries per the
  runtime contract.
- **Output Collector & Validator:** Captures block output, validates against the
  declared output schema, stores results for inspection.
- **State Store:** In-memory key-value store that implements the same API as the
  production state store. Supports optional persistence to disk for stateful block
  development across restarts.
- **Observability UI:** A web-based dashboard (served on `localhost:9091`) showing
  block logs, metrics, traces, and data flow visualization.
- **Block Sandbox:** The block process itself, managed by the emulator. Supports
  both in-process (Go plugin) and out-of-process (gRPC and Arrow Flight) execution
  models, matching the production runtime.

### 4.3 Starting the Emulator

```
meteridian emulator start [--config=<path>] [--port=<port>] [--ui-port=<port>]
```

- Default ports: gRPC `9090`, UI `9091`.
- Configuration file (`emulator.yaml`) specifies synthetic data schemas, event
  rates, and block topology.
- `--config` defaults to `./emulator.yaml` if present, or uses auto-generated
  config based on the block's manifest.

### 4.4 Hot-Reload

The emulator watches the block source directory for file changes. On change:

1. Block receives a graceful shutdown signal (calls `Close()`).
2. Block is rebuilt (if compiled language) or module is reloaded (if interpreted).
3. Block is restarted with `Init()`.
4. For stateless blocks: the last N batches are replayed for immediate feedback.
5. For stateful blocks: state is restored from the in-memory store checkpoint.

Typical reload latency: < 2 seconds for Go blocks, < 500ms for Python blocks.

### 4.5 Debugger Integration

When started with `meteridian run --debug`, the block process is launched with
debugger support:

- **Go blocks:** Delve (`dlv`) is started in headless mode. The CLI prints the
  `dlv connect` command and the VS Code launch configuration.
- **Python blocks:** `debugpy` is started on a configurable port. The CLI prints
  the VS Code and PyCharm attach configuration.
- **Rust blocks:** `rust-lldb` or `rust-gdb` attach instructions are printed.

The emulator pauses event delivery until the debugger is attached, preventing
missed breakpoints during startup.

### 4.6 Data Replay

Developers can feed real-world data through the emulator for realistic testing:

```
meteridian emulator replay --input=<parquet-file|arrow-ipc-file|csv-file>
```

- Parquet and Arrow IPC files are replayed at their original schema.
- CSV files are inferred or mapped to a declared schema via `--schema=<path>`.
- Replay supports rate limiting (`--rate=1000/s`) for simulating production
  throughput.

---

## 5. Block SDK

### 5.1 Design Philosophy

All SDKs are generated from a single **Block Interface Specification** — an
Arrow-based IDL that defines the block lifecycle, data contracts, and capability
system. This ensures behavioral consistency across languages: a block written in
Go and a block written in Python, given the same input, must produce the same
output.

The specification is authoritative. When a language's idioms conflict with the
specification, the SDK wraps the specification in idiomatic sugar but preserves
exact behavioral semantics underneath.

### 5.2 Go SDK (`github.com/meteridian/sdk-go`)

**Status:** v1.0 (stable, covered by semver compatibility guarantees)

#### Core Interface

```go
package block

import (
    "context"

    "github.com/apache/arrow-go/v18/arrow"
    "github.com/apache/arrow-go/v18/arrow/array"
)

type Block interface {
    Init(ctx context.Context, config Config) error
    Process(ctx context.Context, batch arrow.Record) (arrow.Record, error)
    Close(ctx context.Context) error
}

type Config struct {
    Parameters map[string]any
    State      StateStore
    Logger     Logger
    Metrics    MetricsReporter
}
```

#### Utilities

- **Arrow schema builders:** Fluent API for constructing Arrow schemas without
  low-level allocator management.
  ```go
  schema := arrowutil.NewSchemaBuilder().
      AddField("timestamp", arrow.TIMESTAMP, arrowutil.NotNull).
      AddField("usage_kwh", arrow.FLOAT64, arrowutil.NotNull).
      AddField("meter_id", arrow.STRING, arrowutil.NotNull).
      Build()
  ```
- **RecordBatch helpers:** Convenience functions for creating test batches, slicing,
  filtering, and transforming records.
- **State store client:** Type-safe wrapper around the runtime state store API.
  ```go
  var counter int64
  err := config.State.Get(ctx, "event_count", &counter)
  counter++
  err = config.State.Put(ctx, "event_count", counter)
  ```
- **Structured logging:** `config.Logger.Info("processed batch", "rows", batch.NumRows())`
  emits structured JSON logs compatible with the Meteridian observability stack.
- **Metrics:** `config.Metrics.Histogram("process_latency_ms", elapsed.Milliseconds())`
  records metrics that are exported as OpenTelemetry spans.

### 5.3 Python SDK (`meteridian-sdk`)

**Status:** v1.0 (stable)

#### Decorator-Based API

```python
import pyarrow as pa
from meteridian import block, on_batch, on_error

@block(
    name="usage-normalizer",
    version="1.0.0",
    input_schema=pa.schema([
        pa.field("timestamp", pa.timestamp("ms")),
        pa.field("raw_kwh", pa.float64()),
    ]),
    output_schema=pa.schema([
        pa.field("timestamp", pa.timestamp("ms")),
        pa.field("usage_kwh", pa.float64()),
        pa.field("normalized", pa.bool_()),
    ]),
)
class UsageNormalizer:

    def __init__(self, config):
        self.threshold = config.get("threshold", 1000.0)

    @on_batch
    def process(self, ctx, batch: pa.RecordBatch) -> pa.RecordBatch:
        raw = batch.column("raw_kwh").to_pylist()
        normalized = [min(v, self.threshold) for v in raw]
        return pa.RecordBatch.from_pydict({
            "timestamp": batch.column("timestamp"),
            "usage_kwh": normalized,
            "normalized": [v >= self.threshold for v in raw],
        }, schema=self.output_schema)

    @on_error
    def handle_error(self, ctx, error, batch):
        ctx.logger.error("batch processing failed", error=str(error))
        ctx.metrics.counter("errors_total", 1)
```

#### Features

- **Type-safe configuration:** Block manifest parameters are parsed and validated
  against Python type hints at startup.
- **PyArrow-native:** No serialization boundary — Arrow RecordBatches are the
  native data type throughout.
- **Async support:** `@on_batch` handlers can be `async def` for I/O-heavy blocks
  (enrichers calling external APIs, sinks writing to databases).
- **gRPC and Arrow Flight transport:** Python blocks run as gRPC servers, communicating
  with the runtime via Arrow Flight. The SDK handles all transport boilerplate.

### 5.4 Future SDKs

| Language | Package | Status | Notes |
|---|---|---|---|
| Rust | `meteridian-sdk-rs` | Planned (v0.1) | Native Arrow2/arrow-rs, WASM target support |
| TypeScript | `@meteridian/sdk` | Planned (v0.1) | For Arrow.js-based blocks, WASM interop |
| Java | `io.meteridian:sdk` | Under consideration | For Kafka/Flink ecosystem integration |

### 5.5 Block Interface Specification

The specification is defined as a versioned document (`block-interface-spec.yaml`)
that describes:

- Block lifecycle states and transitions
- Method signatures with Arrow-typed parameters and returns
- Capability declarations and their runtime effects
- Error handling contracts (retryable vs fatal errors)
- State store API semantics (consistency, isolation, TTL)
- Logging and metrics API contracts
- Configuration schema validation rules

SDK code generators consume this specification to produce language-specific
implementations that are tested for behavioral conformance.

---

## 6. Testing Framework

### 6.1 Unit Testing

Each SDK provides a test harness that creates mock runtime contexts, generates
test RecordBatches, and provides assertion helpers tailored to Arrow data.

#### Go: `blocktest` Package

```go
package myblock_test

import (
    "testing"

    "github.com/meteridian/sdk-go/blocktest"
    "github.com/example/my-filter-block"
)

func TestFilterBlock(t *testing.T) {
    runner := blocktest.NewTestRunner(t, &myblock.FilterBlock{})
    runner.WithConfig(map[string]any{"min_usage": 100.0})

    input := blocktest.NewBatchBuilder(myblock.InputSchema).
        AddRow("2026-06-18T10:00:00Z", 150.0, "meter-1").
        AddRow("2026-06-18T10:01:00Z", 50.0, "meter-2").
        AddRow("2026-06-18T10:02:00Z", 200.0, "meter-3").
        Build()

    output := runner.Process(input)

    blocktest.AssertBatchEqual(t, output, blocktest.ExpectedBatch{
        Rows: 2,
        Columns: map[string][]any{
            "usage_kwh": {150.0, 200.0},
            "meter_id":  {"meter-1", "meter-3"},
        },
    })
}
```

**Assertion helpers:**

- `AssertBatchEqual` — deep equality on RecordBatch contents
- `AssertSchemaEqual` — structural equality on Arrow schemas
- `AssertRowCount` — verify batch size
- `AssertColumnValues` — verify a single column's values
- `AssertNoNulls` — verify no null values in specified columns

#### Python: `meteridian.testing` Module

```python
from meteridian.testing import BlockTestCase, make_batch

class TestUsageNormalizer(BlockTestCase):
    block_class = UsageNormalizer
    config = {"threshold": 1000.0}

    def test_values_below_threshold_pass_through(self):
        batch = make_batch(
            schema=self.block.input_schema,
            data={"timestamp": ["2026-06-18T10:00:00"], "raw_kwh": [500.0]},
        )
        result = self.process(batch)

        self.assert_column_values(result, "usage_kwh", [500.0])
        self.assert_column_values(result, "normalized", [False])

    def test_values_above_threshold_are_capped(self):
        batch = make_batch(
            schema=self.block.input_schema,
            data={"timestamp": ["2026-06-18T10:00:00"], "raw_kwh": [1500.0]},
        )
        result = self.process(batch)

        self.assert_column_values(result, "usage_kwh", [1000.0])
        self.assert_column_values(result, "normalized", [True])
```

### 6.2 Integration Testing

Integration tests run a block inside the local emulator with real Arrow data flow.

```
meteridian test --integration
```

**What happens:**

1. The emulator starts (or connects to a running instance).
2. The block is built and deployed into the emulator.
3. Test events are sent from fixtures defined in `testdata/` or generated from
   the block's input schema.
4. Block output is captured and compared against expected results.
5. State store contents are verified (for stateful blocks).
6. The emulator shuts down (or remains running for rapid iteration).

Integration test fixtures are defined in `testdata/integration/`:

```
testdata/
  integration/
    scenario_basic/
      input.parquet          # Input data
      expected_output.parquet # Expected output
      config.yaml            # Block configuration for this scenario
    scenario_edge_cases/
      input.parquet
      expected_output.parquet
      config.yaml
```

### 6.3 Performance Testing

```
meteridian bench --duration=30 --batch-size=10000 --output=json
```

**Output metrics:**

| Metric | Description |
|---|---|
| `latency_p50_ms` | Median per-batch processing latency |
| `latency_p95_ms` | 95th percentile latency |
| `latency_p99_ms` | 99th percentile latency |
| `throughput_batches_sec` | Batches processed per second |
| `throughput_rows_sec` | Rows processed per second |
| `memory_peak_mb` | Peak memory usage during benchmark |
| `memory_alloc_bytes_op` | Memory allocated per batch (Go only) |
| `gc_pause_total_ms` | Total GC pause time (Go only) |

**Baseline comparison:**

```
meteridian bench --compare-baseline=.bench-baseline.json --fail-on-regression=10%
```

- Stores baseline results in `.bench-baseline.json`.
- `--fail-on-regression=10%` fails if any metric degrades by more than 10%.
- Designed for CI integration: exits non-zero on regression, outputs diff as JSON.

### 6.4 Snapshot Testing

Snapshot testing records a block's output for known inputs and detects unexpected
changes in future runs — particularly useful for blocks with complex transformation
logic where "correct" is difficult to express as assertions.

```
# Record initial snapshots
meteridian test --update-snapshots

# Verify against recorded snapshots
meteridian test --snapshots
```

Snapshots are stored as Arrow IPC files in `testdata/snapshots/`, committed to
version control. The diff output highlights exactly which rows and columns changed.

### 6.5 Contract Testing

Contract testing verifies that a block's actual runtime behavior matches its
declared manifest — specifically, that the output Arrow schema produced by the
block matches the schema declared in `block.toml`.

```
meteridian lint --verify-schema
```

**What it checks:**

- Field names, types, and nullability match the declared output schema.
- No undeclared fields are present in the output.
- No declared fields are missing from the output.
- Nested types (structs, lists, maps) are structurally compatible.

This is distinct from unit tests: contract tests verify the *declaration* against
*behavior*, catching drift between the manifest and the implementation.

---

## 7. Documentation

### 7.1 Documentation Architecture

Documentation is hosted at `docs.meteridian.io`, built with a static site
generator (Docusaurus or equivalent), and versioned alongside the SDK releases.

**Documentation tiers:**

| Tier | Audience | Content |
|---|---|---|
| Getting Started | New developers | 15-minute tutorial, prerequisites, installation |
| Block Developer Guide | Active developers | Step-by-step from init to publish |
| API Reference | All developers | Auto-generated from SDK source (godoc, Sphinx) |
| Cookbook | Intermediate developers | Patterns, recipes, worked examples |
| Architecture Guide | Advanced developers | Runtime internals, data model, lifecycle |
| Troubleshooting | All developers | Common errors, FAQ, debugging techniques |

### 7.2 Block Developer Guide

A structured tutorial that walks a developer from "I have never seen Meteridian"
to "I published a block to the marketplace":

1. **Install the CLI** — one command per platform.
2. **Initialize a project** — `meteridian init --template=filter --lang=go`.
3. **Understand the manifest** — annotated walkthrough of `block.toml`.
4. **Write the block logic** — implement `Process()` with Arrow data.
5. **Write tests** — unit test with `blocktest`, integration test with emulator.
6. **Run locally** — `meteridian run --emulator` with hot-reload.
7. **Benchmark** — `meteridian bench` to verify performance.
8. **Lint and validate** — `meteridian lint --verify-schema --strict`.
9. **Publish** — `meteridian publish` with signing and provenance.

Each step includes full code samples, expected CLI output, and "what can go wrong"
sidebars.

### 7.3 Cookbook

Common patterns with complete, copy-paste-ready examples:

- **Filter block:** Drop events matching a condition (e.g., test meters).
- **Enricher block:** Add columns from an external lookup (e.g., customer name
  from meter ID, rate schedule from tariff code).
- **Aggregator block:** Window events by time, group by meter, compute sums and
  averages.
- **Splitter block:** Route events to different outputs based on content (e.g.,
  residential vs. commercial meters).
- **Merger block:** Combine events from multiple inputs into a unified stream.
- **Error handler block:** Dead-letter queue pattern with retry and backoff.
- **Stateful block:** Maintain running totals across batches using the state store.
- **Multi-language pipeline:** Go block → Python block → Go block, demonstrating
  Arrow Flight interop.

### 7.4 API Reference

Auto-generated from SDK source code:

- **Go:** Generated via `godoc` and rendered as HTML with cross-references.
- **Python:** Generated via Sphinx with `autodoc` and type annotation rendering.
- **Rust (future):** Generated via `cargo doc`.

API references are versioned: `docs.meteridian.io/sdk-go/v1.2/` provides the
reference for SDK v1.2.x.

### 7.5 Troubleshooting Guide

Organized as a searchable database of symptoms, causes, and fixes:

| Symptom | Cause | Fix |
|---|---|---|
| `schema mismatch: expected 5 fields, got 4` | Block output missing a declared field | Add the missing field or update manifest |
| `state store: key not found` | Accessing state before first write | Use `GetOrDefault()` instead of `Get()` |
| `emulator: port 9090 in use` | Another process on the port | Run `meteridian doctor` to diagnose |
| `publish: signature verification failed` | OIDC token expired | Re-authenticate with `meteridian auth login` |
| `bench: throughput below baseline` | Performance regression | Use `meteridian profile` to identify bottleneck |

---

## 8. CI/CD Integration

### 8.1 GitHub Actions

Official GitHub Actions for block CI/CD pipelines:

#### `meteridian/build-action@v1`

```yaml
- uses: meteridian/build-action@v1
  with:
    language: go
    target: binary
    platforms: linux/amd64,linux/arm64
```

- Installs the `meteridian` CLI.
- Runs `meteridian build` with the specified target and platforms.
- Caches build artifacts across runs.
- Outputs: artifact path, artifact digest (SHA256).

#### `meteridian/test-action@v1`

```yaml
- uses: meteridian/test-action@v1
  with:
    unit: true
    integration: true
    snapshots: true
    bench: true
    bench-compare-baseline: true
    bench-fail-threshold: "10%"
```

- Runs `meteridian test` with the specified flags.
- For integration tests: starts the emulator as a service container.
- For benchmarks: runs on a dedicated runner profile for consistent results.
- Outputs: test results (JUnit XML), coverage report, benchmark results (JSON).

#### `meteridian/publish-action@v1`

```yaml
- uses: meteridian/publish-action@v1
  with:
    registry: marketplace.meteridian.io
    sign: true
    provenance: true
  env:
    METERIDIAN_API_TOKEN: ${{ secrets.METERIDIAN_API_TOKEN }}
```

- Orchestrates the full publish workflow (see Section 11).
- Uses GitHub's OIDC provider for keyless Sigstore signing.
- Generates SLSA provenance attestation using the GitHub Actions builder.
- Uploads to the marketplace registry.

#### Complete Example Workflow

```yaml
name: Block CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: meteridian/build-action@v1
        with:
          language: go
      - uses: meteridian/test-action@v1
        with:
          unit: true
          integration: true
          bench: true
          bench-compare-baseline: true

  publish:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: meteridian/publish-action@v1
        with:
          sign: true
          provenance: true
        env:
          METERIDIAN_API_TOKEN: ${{ secrets.METERIDIAN_API_TOKEN }}
```

### 8.2 GitLab CI Templates

Equivalent templates for GitLab CI/CD:

```yaml
include:
  - remote: "https://registry.meteridian.io/ci/gitlab/block-pipeline.yml"

variables:
  METERIDIAN_LANGUAGE: go
  METERIDIAN_BENCH_THRESHOLD: "10%"
```

The template defines `build`, `test`, `bench`, and `publish` stages with sensible
defaults. Each stage can be overridden or extended.

### 8.3 Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/meteridian/pre-commit-hooks
    rev: v1.0.0
    hooks:
      - id: meteridian-lint
        name: Validate block manifest and schemas
      - id: meteridian-test-unit
        name: Run block unit tests
```

Pre-commit hooks run `meteridian lint` and optionally `meteridian test --unit`
before each commit, catching manifest errors and test failures early.

---

## 9. Performance Profiling

### 9.1 Local Profiling Dashboard

```
meteridian profile --port=9091
```

Launches a browser-based dashboard at `http://localhost:9091` that displays
real-time performance data from blocks running in the local emulator.

**Dashboard panels:**

| Panel | Visualization | Data Source |
|---|---|---|
| Batch Latency | Histogram (p50/p95/p99) | OpenTelemetry spans |
| Throughput | Time-series gauge (batches/sec) | Block router metrics |
| Memory | Timeline (RSS, heap, stack) | Runtime metrics (pprof and pymalloc) |
| Data Flow | Sankey diagram (input → block → output) | Emulator routing |
| Error Rate | Counter and timeline | Block error handler |
| State Store | Operations/sec, key count | State store instrumentation |

### 9.2 Block-Level OpenTelemetry Spans

As defined in METR-0002 Section 7, each block invocation emits an OpenTelemetry
span with the following attributes:

- `block.name`, `block.version`, `block.instance_id`
- `batch.num_rows`, `batch.size_bytes`
- `processing.duration_ms`, `processing.status`
- `state.operations_count`, `state.bytes_read`, `state.bytes_written`

These spans are collected by the emulator's built-in OpenTelemetry collector and
displayed in the profiling dashboard. In production, they are exported to the
configured observability backend (Jaeger, Grafana Tempo, Datadog, etc.).

### 9.3 Production Profiling

Opt-in continuous profiling for deployed blocks:

- **Go blocks:** pprof endpoints exposed at `/debug/pprof/`. The runtime can be
  configured to periodically capture and export CPU and memory profiles.
- **Python blocks:** py-spy integration for sampling-based CPU profiling without
  code modification. Enabled via block manifest flag:
  ```toml
  [profiling]
  enabled                  = true
  cpu_sample_rate          = 100   # Hz
  memory_snapshot_interval = 60    # seconds
  ```
- Profile data is stored in the Meteridian observability backend and accessible
  via the platform's UI for flame graph visualization.

### 9.4 Performance Budgets

Block manifests can declare performance constraints that the emulator enforces:

```toml
[performance]
max_latency_p99_ms     = 10
min_throughput_rows_sec = 100000
max_memory_mb          = 256
```

- The emulator warns (yellow) when a budget is approached (>80% of limit).
- The emulator errors (red) when a budget is exceeded.
- `meteridian bench` fails if performance budgets are violated.
- In production, budget violations emit alerts (not failures) to allow for
  transient spikes.

---

## 10. Block Scaffolding Templates

### 10.1 Template Registry

The CLI ships with built-in templates for the most common block patterns. Each
template generates a complete, working project that passes `meteridian lint`,
`meteridian test`, and `meteridian build` out of the box.

| Template | Command | Description |
|---|---|---|
| Filter | `meteridian init --template=filter` | Drops events matching a condition |
| Enricher | `meteridian init --template=enricher` | Adds columns from external data |
| Aggregator | `meteridian init --template=aggregator` | Windows, groups, and aggregates |
| Sink | `meteridian init --template=sink` | Writes events to an external system |
| Source | `meteridian init --template=source` | Generates or imports events |
| Passthrough | `meteridian init --template=passthrough` | Minimal no-op (identity function) |

### 10.2 Generated Project Structure (Go)

```
my-filter-block/
├── block.toml                    # Block manifest
├── main.go                       # Block implementation
├── main_test.go                  # Unit tests
├── testdata/
│   ├── input.parquet             # Test input data
│   └── snapshots/                # Snapshot baselines
├── emulator.yaml                 # Emulator configuration
├── .github/
│   └── workflows/
│       └── ci.yml                # GitHub Actions CI pipeline
├── .pre-commit-config.yaml       # Pre-commit hooks
├── .bench-baseline.json          # Performance baseline
├── go.mod                        # Go module definition
├── go.sum                        # Go dependency checksums
└── README.md                     # Generated documentation
```

### 10.3 Generated Project Structure (Python)

```
my-enricher-block/
├── block.toml                    # Block manifest
├── src/
│   └── my_enricher_block/
│       ├── __init__.py
│       ├── block.py              # Block implementation
│       └── config.py             # Configuration schema
├── tests/
│   ├── __init__.py
│   ├── test_block.py             # Unit tests
│   └── conftest.py               # Test fixtures
├── testdata/
│   ├── input.parquet
│   └── snapshots/
├── emulator.yaml
├── .github/
│   └── workflows/
│       └── ci.yml
├── .pre-commit-config.yaml
├── .bench-baseline.json
├── pyproject.toml                # Python project config (PEP 621)
├── requirements.lock             # Locked dependencies
└── README.md
```

### 10.4 Block Manifest (`block.toml`)

Every template generates a `block.toml` manifest — the declarative specification
of a block's identity, interface, capabilities, and requirements. The structure
below extends the canonical schema defined in METR-0002 (Section 3.4) with
additional `[block.runtime]` detail:

```toml
# block.toml — canonical block manifest (TOML chosen over YAML for
# strict typing, no implicit coercion, and zero code-execution risk;
# see METR-0002 Section 3.4 for the authoritative schema)

[block]
api_version = "meteridian.io/v1"
name        = "my-filter-block"
version     = "0.1.0"
description = "Filters metering events below a usage threshold"
author      = "developer@example.com"
license     = "Apache-2.0"
tags        = ["filter", "usage", "metering"]

[block.runtime]
language = "go"
type     = "go"  # "go" | "grpc" | "wasm"

[[input.fields]]
name     = "timestamp"
type     = "timestamp[ms]"
nullable = false

[[input.fields]]
name     = "usage_kwh"
type     = "float64"
nullable = false

[[input.fields]]
name     = "meter_id"
type     = "utf8"
nullable = false

[[output.fields]]
name     = "timestamp"
type     = "timestamp[ms]"
nullable = false

[[output.fields]]
name     = "usage_kwh"
type     = "float64"
nullable = false

[[output.fields]]
name     = "meter_id"
type     = "utf8"
nullable = false

[capabilities]
state   = ["read"]
metrics = ["write"]

[[config.parameters]]
name        = "min_usage"
type        = "float64"
default     = 0.0
description = "Minimum usage threshold (events below this are dropped)"

[performance]
max_latency_p99_ms      = 5
min_throughput_rows_sec  = 500000

[health]
liveness_path  = "/healthz"
readiness_path = "/readyz"

[metrics]
endpoint = "/metrics"
format   = "otlp"
```

### 10.5 Custom Templates

Developers and organizations can create and share custom templates:

```
meteridian init --template=https://github.com/myorg/meteridian-templates/tree/main/custom-block
```

Custom templates are standard project directories with `.template.yaml` metadata
that defines parameterization (block name, author, schema fields, etc.).

---

## 11. Marketplace Publishing Workflow

### 11.1 Publish Pipeline

When a developer runs `meteridian publish`, the following steps execute
sequentially. Each step must pass before the next begins.

```
┌────────┐   ┌────────┐   ┌───────┐   ┌──────┐   ┌──────────┐
│  Lint  │──▶│  Test  │──▶│ Build │──▶│ Sign │──▶│ Provenance│
└────────┘   └────────┘   └───────┘   └──────┘   └──────┬────┘
                                                         │
┌────────────┐   ┌──────────┐   ┌────────────────┐       │
│  Security  │◀──│  Upload  │◀──│  Package       │◀──────┘
│  Scan      │   │          │   │  (OCI artifact) │
└─────┬──────┘   └──────────┘   └────────────────┘
      │
      ▼
┌──────────────┐   ┌────────────┐
│  Pending     │──▶│  Published │
│  Review      │   │            │
└──────────────┘   └────────────┘
```

### 11.2 Step Details

**Step 1: Lint (`meteridian lint --strict`)**
- Validates manifest, capability declarations, and schema compatibility.
- Fails on any warning in strict mode.

**Step 2: Test (`meteridian test --unit --integration`)**
- Runs the full test suite including integration tests.
- Fails on any test failure.

**Step 3: Build (`meteridian build`)**
- Produces the deployable artifact (binary, container image, or WASM module).
- Cross-compiles for supported platforms.

**Step 4: Sign (Sigstore)**
- Signs the artifact using Sigstore keyless signing (OIDC-based identity).
- The developer authenticates via their identity provider (GitHub, Google, etc.).
- Signature and certificate are stored in Rekor (Sigstore transparency log).
- See ADR-0009 for cryptographic details.

**Step 5: Provenance (SLSA)**
- Generates an SLSA v1.0 provenance attestation documenting the build environment,
  source commit, build command, and dependencies.
- Attestation is signed and stored alongside the artifact.

**Step 6: Package (OCI Artifact)**
- The signed artifact, provenance attestation, and manifest are packaged as an
  OCI artifact for distribution via the marketplace registry.

**Step 7: Upload**
- The OCI artifact is pushed to `marketplace.meteridian.io` (an OCI-compliant
  container registry).
- Upload requires an API token obtained via `meteridian auth login`.

**Step 8: Security Scan**
- Automated dependency audit: checks all dependencies against known vulnerability
  databases (OSV, NVD, GitHub Advisory).
- Static analysis (SAST): scans block source code for common security issues.
- Container scanning (if container target): scans base image for vulnerabilities.
- Results are attached to the block's marketplace listing.

**Step 9: Review**
- The block enters "Pending Review" status.
- Automated review: a bot validates manifest completeness, test coverage minimums,
  and security scan results.
- Optional manual review: for blocks requesting elevated capabilities (e.g.,
  network access, filesystem access), a human reviewer inspects the block.
- On approval: status transitions to "Published" and the block becomes available
  in the marketplace.

### 11.3 Versioning in the Marketplace

Each publish creates a new version entry. The marketplace maintains the full
version history, and operators can pin to specific versions or version ranges.

---

## 12. Versioning and Compatibility

### 12.1 Semantic Versioning

All Meteridian developer-facing artifacts follow [Semantic Versioning 2.0.0](https://semver.org/):

| Artifact | Versioning Policy |
|---|---|
| `meteridian` CLI | MAJOR.MINOR.PATCH. MAJOR bumps on breaking CLI flag/output changes. |
| Go SDK | MAJOR.MINOR.PATCH per Go module conventions. |
| Python SDK | MAJOR.MINOR.PATCH per PEP 440. |
| Block Interface Spec | MAJOR.MINOR. MAJOR bumps on breaking interface changes. |
| Block Manifest Schema | MAJOR.MINOR. MAJOR bumps on breaking schema changes. |
| Published Blocks | MAJOR.MINOR.PATCH. Operators choose version constraints. |

### 12.2 Runtime Backward Compatibility

The Meteridian block runtime maintains backward compatibility with **N-2 SDK
major versions**. This means:

- A block built with SDK v1.x will continue to work on runtimes that support
  SDK v1, v2, and v3.
- When SDK v4 is released, SDK v1 support is deprecated (but not immediately
  removed).
- Deprecation is signaled 6 months in advance via SDK release notes, CLI warnings,
  and marketplace notifications.

### 12.3 Deprecation Policy

| Phase | Duration | Actions |
|---|---|---|
| Announcement | T+0 | Feature marked deprecated in docs, SDK, and CLI output |
| Warning period | T+0 to T+6 months | CLI emits deprecation warnings, build succeeds |
| Soft removal | T+6 months | CLI emits errors, build succeeds with `--allow-deprecated` |
| Hard removal | T+12 months | Feature removed from SDK and CLI |

### 12.4 Block Manifest Versioning

The `block.toml` manifest includes an `apiVersion` field (e.g.,
`meteridian.io/v1`). When the manifest schema changes:

- **Minor changes** (new optional fields): backward-compatible, no `apiVersion`
  bump.
- **Breaking changes** (field removals, type changes, semantic changes): new
  `apiVersion` (e.g., `meteridian.io/v2`).
- The CLI and runtime support multiple `apiVersion` values simultaneously,
  following the N-2 policy.
- `meteridian lint` warns when a block uses a deprecated `apiVersion`.

### 12.5 SDK Release Cadence

- **Patch releases:** As needed for bug fixes and security patches. No minimum
  interval.
- **Minor releases:** Monthly, aligned with Meteridian platform releases. Include
  new features, performance improvements, and non-breaking enhancements.
- **Major releases:** Annually (at most). Require migration guide and 6-month
  deprecation period for removed features.

---

## 13. Open Questions

### 13.1 Pipeline-Level Testing

**Question:** Should the emulator support pipeline-level testing (multiple blocks
chained together) or only single-block testing?

**Arguments for pipeline-level testing:**
- Blocks are designed to be composed. Testing in isolation misses integration
  issues (schema mismatches between blocks, ordering assumptions, state
  dependencies).
- Pipeline-level tests give developers confidence that their block works in
  context, not just in isolation.
- The emulator already has a block router — extending it to support multi-block
  pipelines is architecturally straightforward.

**Arguments against (or for deferring):**
- Single-block testing is simpler to implement and understand.
- Pipeline-level testing requires defining pipeline topology, which is an
  operator concern, not a developer concern.
- Scope creep: the emulator could become a mini-Meteridian, duplicating platform
  complexity.

**Recommendation:** Support pipeline-level testing in v1.1 (not v1.0). v1.0
focuses on single-block testing with well-defined input and output contracts. Pipeline
testing is explicitly a follow-up feature.

### 13.2 Block Dependencies

**Question:** How should block dependencies be handled? If block A depends on
library X at version Y, how is this managed?

**Considerations:**
- Go blocks vendor dependencies (standard Go module practice). No runtime
  dependency resolution needed.
- Python blocks declare dependencies in `pyproject.toml`. The `meteridian build`
  step produces a container image with locked dependencies — no runtime pip
  installs.
- WASM blocks are self-contained by definition.
- The real question is about **block-to-block dependencies**: should block A be
  able to declare that it requires block B to be present in the pipeline?

**Recommendation:** For v1.0, blocks are independent units with no inter-block
dependency declarations. Pipeline topology is an operator responsibility. Block-to-
block dependency declarations are a potential v2.0 feature.

### 13.3 Web-Based Playground

**Question:** Should we provide a web-based playground (like Go Playground or Rust
Playground) for trying blocks without local setup?

**Arguments for:**
- Lowers the barrier to entry dramatically — no CLI installation, no toolchain
  setup.
- Excellent for documentation: code examples can have a "Run in Playground" button.
- Marketing value: potential users can try Meteridian before committing.

**Arguments against:**
- Significant infrastructure investment (sandboxed execution environment, resource
  limits, abuse prevention).
- The playground must be kept in sync with the SDK and emulator — another
  maintenance surface.
- Security: running arbitrary user code in a web environment requires careful
  sandboxing (WASM-based execution, resource quotas, network isolation).

**Recommendation:** Defer to v2.0. For v1.0, invest in making the CLI installation
as frictionless as possible (single binary, `brew install`). Consider a WASM-based
playground as a v2.0 feature once the SDK stabilizes.

### 13.4 IDE Extensions

**Question:** Should Meteridian provide official IDE extensions (VS Code, JetBrains)?

**Considerations:**
- VS Code extension could provide: `block.toml` schema validation with
  autocompletion, inline Arrow schema visualization, one-click emulator start,
  integrated profiling dashboard, block template snippets.
- JetBrains plugin would serve the Go developer audience.
- IDE extensions are high-maintenance but high-impact.

**Recommendation:** Provide a VS Code extension in v1.1 with manifest validation
and emulator integration. JetBrains plugin is a community contribution opportunity.

---

## 14. References

1. **Terraform Provider SDK** — [github.com/hashicorp/terraform-plugin-sdk](https://github.com/hashicorp/terraform-plugin-sdk)
   Established patterns for plugin SDK design, acceptance testing framework, and
   documentation generation. Meteridian's `blocktest` package draws directly from
   Terraform's `resource.Test` patterns.

2. **Stripe CLI** — [github.com/stripe/stripe-cli](https://github.com/stripe/stripe-cli)
   Best-in-class developer CLI with local webhook testing, event replay, and
   integrated documentation. Inspiration for `meteridian run` and
   `meteridian emulator`.

3. **Firebase Emulator Suite** — [firebase.google.com/docs/emulator-suite](https://firebase.google.com/docs/emulator-suite)
   Multi-service local emulator with UI, data import and export, and CI integration.
   Architecture reference for the Meteridian local development emulator.

4. **VS Code Extension API** — [code.visualstudio.com/api](https://code.visualstudio.com/api)
   Extension development lifecycle: scaffolding (`yo code`), testing (`@vscode/test-electron`),
   and marketplace publishing. Model for the Meteridian block lifecycle.

5. **LocalStack** — [github.com/localstack/localstack](https://github.com/localstack/localstack)
   AWS cloud emulator for local development and testing. Architecture reference for
   simulating cloud services without external dependencies.

6. **Backstage Developer Portal** — [backstage.io](https://backstage.io)
   Software catalog, templates, and documentation platform. Potential future
   integration point for organizations managing large numbers of Meteridian blocks.

7. **Sigstore** — [sigstore.dev](https://sigstore.dev)
   Keyless code signing and transparency log. Used in the marketplace publish
   workflow (ADR-0009) for artifact signing and verification.

8. **SLSA Framework** — [slsa.dev](https://slsa.dev)
   Supply-chain Levels for Software Artifacts. SLSA provenance attestations are
   generated during `meteridian publish` to establish build integrity.

9. **Apache Arrow** — [arrow.apache.org](https://arrow.apache.org)
   Columnar in-memory data format. The foundation of Meteridian's data model —
   all block inputs and outputs are Arrow RecordBatches.

10. **OpenTelemetry** — [opentelemetry.io](https://opentelemetry.io)
    Observability framework for traces, metrics, and logs. Each block emits
    OpenTelemetry spans for performance monitoring and debugging.
