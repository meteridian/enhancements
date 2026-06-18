# ADR-0006: Hybrid Plugin Runtime (Go SDK + gRPC now, WASM later)

- **Status:** Accepted
- **Date:** 2026-06-18
- **Updated:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian's block-based extensibility model (METR-0002) requires blocks to run
in multiple languages with varying security and performance characteristics. The
platform must simultaneously support three distinct use cases that have
conflicting requirements:

1. **Core platform blocks** (collectors, raters, storage adapters) need
   zero-overhead execution with full access to Meteridian internal APIs. These
   are written and maintained by the Meteridian team and are fully trusted.
   Performance is paramount — the rating engine processes millions of events per
   hour and any serialization overhead at block boundaries is unacceptable.

2. **Marketplace blocks** from third-party developers need strong sandboxing.
   These blocks process sensitive billing data (tenant IDs, usage amounts,
   payment tokens) and must be isolated from the host process. A malicious or
   buggy marketplace block must not be able to read another tenant's data,
   exfiltrate secrets, or crash the host process. Languages should include Rust,
   C, and TinyGo (anything that compiles to WASM).

3. **Data science and ML blocks** need access to full language runtimes with
   rich library ecosystems. Python with NumPy, Pandas, and scikit-learn for
   anomaly detection; Java with ML libraries for fraud scoring. These blocks
   may require GPU access for inference. WASM cannot support these workloads
   because PyArrow, NumPy, and JVM libraries do not run in WASM.

A single runtime model cannot satisfy all three use cases. An in-process model
(like Go plugins) provides performance but no isolation. A WASM-only model
provides isolation but excludes Python/Java ecosystems. A gRPC-only model
provides language flexibility but adds unnecessary latency for core blocks.

## Decision

Meteridian provides **three runtime models**, phased over two releases, that
share a single logical interface (receive Arrow RecordBatches, process, emit
Arrow RecordBatches). The choice of runtime determines the performance/isolation
trade-off.

### Phase 1 (v1.0): Go SDK + gRPC (Arrow Flight)

**1. Go SDK (in-process):**
Blocks compiled with the Go SDK are linked directly into the Meteridian engine
binary (or loaded as Go plugins in development). They share the same address
space and memory, providing zero serialization overhead. They have full access
to Meteridian internal APIs (database, cache, configuration). This runtime is
reserved for Tier 3 (Official) blocks only — it requires full trust because
there is no memory isolation.

**2. gRPC Out-of-Process (Arrow Flight):**
Blocks run as separate processes (or containers) communicating via Arrow Flight
RPC. This provides the strongest isolation (process boundary or container
boundary) and supports any language with Arrow Flight bindings (Python, Java,
Node.js, Ruby, C++, Rust, Go, C#). Overhead is ~1-5ms per call due to
network serialization. This runtime is used for marketplace blocks, blocks
requiring full language runtimes (Python ML, Java enterprise libraries), and
blocks requiring GPU access.

These two runtimes cover every production use case at launch. Go SDK handles
the performance-critical hot path. Arrow Flight handles everything that needs
language flexibility, isolation, or third-party sandboxing. Arrow Flight is
production-proven in Dremio, InfluxDB, and Apache Spark.

### Phase 2 (v2.0): Add WASM Runtime

**3. WASM Runtime (Extism, in-process sandboxed):**
Blocks compiled to WebAssembly run inside the Extism runtime within the
Meteridian process. Extism provides capability-based security with deny-by-default
network, filesystem, and memory access. Data crosses the WASM boundary via Arrow
IPC over shared memory, with ~50-200μs overhead per call (estimated, not yet
benchmarked). Supported languages include Rust, TinyGo, C, AssemblyScript, and
Zig — any language that targets WASM.

WASM is deferred to v2 because:

- **Arrow + WASM integration is immature.** There is no production-proven library
  for zero-copy Arrow IPC through a WASM boundary. Arrow libraries for Rust
  (arrow-rs) and C++ work outside WASM but their WASM compilation targets have
  not been battle-tested at scale. We would be building novel infrastructure.
- **WASI Preview 2 is not yet finalized.** Component model and WASI networking
  are still evolving. Building on these now risks rework when specifications
  stabilize.
- **gRPC fills the same niche for v1.** Any block that would use WASM (Rust,
  TinyGo) can also run as a gRPC/Arrow Flight process with stronger isolation
  (process boundary vs. in-process sandbox). The latency difference
  (~50-200μs WASM vs. ~1-5ms gRPC) is acceptable for v1 workloads.
- **v2 investment is justified by scale.** Once the marketplace has enough block
  authors that the per-block overhead of a gRPC sidecar becomes a resource
  concern, WASM's in-process model provides meaningful density improvements.

### Interface Stability

All runtimes implement the same block interface. A pipeline can mix blocks from
different runtimes without any changes to the pipeline definition. The platform
handles the serialization/transport differences transparently. A block written
for gRPC in v1 will continue to work in v2 without changes; WASM is additive.

## Consequences

### Positive

- **Right tool for each use case**: Core platform blocks get zero overhead, marketplace
  blocks get strong sandboxing, and data science blocks get full language runtimes.
  No single use case is penalized by another's requirements.

- **Gradual trust escalation**: A block can start as WASM (Tier 0, untrusted),
  graduate to WASM with capabilities (Tier 1-2), and eventually be promoted to
  Go SDK (Tier 3) if adopted into the core platform. The migration path is
  clear because the logical interface is identical across runtimes.

- **Broad language support**: Go (SDK), Rust/TinyGo/C/AssemblyScript/Zig (WASM),
  Python/Java/Node.js/Ruby/C++ (gRPC). This covers virtually every language
  used in cloud-native development and data engineering.

- **Incremental adoption**: New runtimes can be added in the future (e.g., a
  JavaScript V8 runtime, a .NET runtime) without changing the block interface or
  the pipeline definition format. The runtime is a deployment detail, not an
  architectural constraint.

### Negative

- **Two codepaths at launch (three in v2)**: Each runtime has its own data
  transfer mechanism (in-process for Go SDK, Arrow Flight for gRPC, Arrow IPC
  for future WASM), lifecycle management, and error handling. Bugs may manifest
  in one runtime but not others, increasing the test matrix. Phasing WASM to v2
  reduces the initial maintenance burden.

- **Performance cliff between runtimes**: The jump from Go SDK (~0 overhead) to
  gRPC (~1-5ms) can surprise developers. A block that performs well in
  development (Go SDK) may underperform in production when deployed as gRPC.
  Clear documentation and performance benchmarks for each runtime are essential.
  When WASM is added in v2, the middle tier (~50-200μs) provides a stepping
  stone, but those numbers are estimates that require validation.

- **WASM unknowns deferred, not eliminated**: Deferring WASM to v2 avoids
  premature investment but does not eliminate the technical risk. When v2
  development begins, Arrow IPC in WASM may still be immature, and WASI
  networking may still have gaps. We accept this risk because gRPC provides a
  permanent fallback — WASM is an optimization, not a requirement.

- **Go SDK vendor lock-in**: Blocks written with the Go SDK are tightly coupled
  to Meteridian's internal APIs and cannot be reused in other platforms.
  Developers should be aware that Go SDK blocks are platform-specific, while
  gRPC (and future WASM) blocks are portable.

### Neutral

- The Extism WASM runtime (v2 dependency) is a third-party project. If Extism
  development stalls or changes direction, Meteridian can migrate to another
  WASM runtime (Wasmtime, Wasmer) because the block interface is defined at the
  Arrow IPC level, not at the Extism API level. This decision can be revisited
  when v2 development begins.

- Arrow Flight adds a gRPC dependency for out-of-process blocks. This is
  acceptable because gRPC is already a transitive dependency of many
  Kubernetes-native applications and Arrow Flight is the Arrow project's
  official RPC mechanism.

- Deferring WASM means the v1 marketplace uses gRPC/Arrow Flight for all
  third-party blocks. This is slightly more resource-intensive (each block is
  a separate process) but provides stronger isolation (process boundary) and
  simpler debugging (standard process tooling, gRPC reflection).

## Alternatives Considered

### WASM Only

Using WebAssembly as the single runtime for all blocks, including core platform
blocks.

**Rejected because:** Go does not compile well to WASM for production workloads.
TinyGo targets WASM but lacks the full Go standard library (no `net/http`, no
`database/sql`, no `reflect`). Core platform blocks that need database access,
HTTP clients, and full Go stdlib cannot run in WASM. Additionally, Python and
Java do not have production-quality WASM targets with full library support
(no PyArrow in WASM, no JVM in WASM). A WASM-only model would exclude the
most important languages for both core development (Go) and data science
(Python/Java).

### WASM from Day One (v1)

Including WASM as a v1 runtime alongside Go SDK and gRPC.

**Rejected because:** Arrow IPC over WASM boundaries is not yet production-proven.
The `arrow-rs` crate compiles to WASM but has not been battle-tested for zero-copy
IPC through WASM shared memory at scale. WASI Preview 2 (component model,
networking) is still evolving, risking rework. Since gRPC/Arrow Flight already
provides multi-language support with stronger isolation (process boundary),
shipping WASM in v1 would add engineering risk without providing capabilities
that gRPC doesn't already cover. WASM is deferred to v2 as an optimization for
higher block density, not as a missing capability.

### gRPC Only

Using gRPC (Arrow Flight) as the single runtime, with all blocks running as
separate processes.

**Rejected because:** The ~1-5ms per-call overhead of gRPC is unacceptable for
core platform blocks on the hot path. A pipeline with 5 blocks would add
5-25ms of pure serialization overhead per event batch — significant for a
metering system processing millions of events. Running every block as a
separate process also adds Kubernetes resource overhead (pod scheduling,
networking, service discovery) and complicates deployment. For core blocks
that are fully trusted and tightly integrated, in-process execution is
vastly superior.

### Go Plugins (plugin package)

Using Go's built-in `plugin` package for native extensibility.

**Rejected because:** Go plugins have severe limitations: (1) not supported on
all operating systems (no Windows, limited macOS support), (2) ABI
compatibility requires plugins to be compiled with the exact same Go version
and dependency versions as the host, (3) plugins cannot be unloaded once
loaded (memory leak for hot-reload), (4) no sandboxing — a plugin has full
access to the host process. The ABI compatibility requirement alone makes
plugins impractical for a marketplace where block authors use different Go
versions and dependency trees.

### Lua / JavaScript Embedding

Embedding a scripting language (Lua via GopherLua, JavaScript via V8/Goja) for
lightweight blocks.

**Rejected because:** Scripting runtimes lack type safety for financial data
processing. Neither Lua nor JavaScript has native decimal types or Arrow
integration. Performance for numeric workloads is 10-100x worse than compiled
languages. The scripting model is adequate for simple transformations (field
renaming, basic filtering) but insufficient for the full spectrum of metering
operations (windowed aggregation, statistical analysis, ML inference). If we
ever need a lightweight scripting layer for simple rules, it can be
implemented as a specific block type (using GoRules ZEN engine or similar)
rather than as a general-purpose runtime.

## References

- [Extism](https://extism.org/) — Cross-language WASM plugin framework
- [Arrow Flight](https://arrow.apache.org/docs/format/Flight.html) — Arrow-native RPC
- [Arrow IPC](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc) — Shared memory format
- [TinyGo WASM](https://tinygo.org/docs/guides/webassembly/) — Go subset targeting WASM
- [Go plugin package](https://pkg.go.dev/plugin) — Go native plugin support
- [WASI Preview 2](https://github.com/WebAssembly/WASI/blob/main/preview2/README.md) — WASM System Interface
