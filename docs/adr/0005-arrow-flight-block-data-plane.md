# ADR-0005: Apache Arrow Flight as the Block Data Plane

- **Status:** Accepted
- **Date:** 2026-06-18
- **Updated:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian's block-based extensibility model (METR-0002) requires a serialization
format that serves two roles simultaneously: (1) the wire format for RPC calls
between processing blocks, and (2) the in-memory representation for event
payloads within blocks.

The initial design proposed a dual-format approach: Protocol Buffers (Protobuf)
for RPC envelope/control messages and Apache Arrow for columnar event data, with
developers managing both schema languages. This was rejected as unnecessarily
complex for block authors.

However, it is important to understand what "Arrow everywhere" actually means in
practice. **Apache Arrow Flight — the framework we adopt — internally uses
Protobuf for its control plane messages** (FlightDescriptor, FlightInfo, Ticket,
Action, Result). This is by design: Flight's control messages are small,
structured, and well-served by Protobuf. The data plane (DoGet, DoPut,
DoExchange) uses Arrow IPC — zero-copy, columnar, and batch-oriented. The key
insight is that Arrow Flight **hides the Protobuf from developers**. Block
authors interact only with Arrow APIs. The Protobuf control messages are internal
to the Flight SDK, not part of the block interface.

Arrow Flight is production-proven. Dremio, InfluxDB, Apache Spark, DuckDB, and
the Apache Iceberg ecosystem all use it for high-throughput data transfer. ACM-
published benchmarks show 20-100x performance improvements over JDBC/ODBC for
large analytical datasets. It is not experimental technology.

Meteridian processes financial metering data where precision is non-negotiable.
Arrow provides native `Decimal128` and `Decimal256` types that map directly to
financial precision requirements, unlike Protobuf which has no decimal type.

The metering workload is inherently columnar: aggregating usage across time
windows, summing costs per tenant, filtering by resource type. Arrow's columnar
layout keeps all values for a given column in contiguous memory, enabling SIMD
vectorization and cache-friendly access patterns.

## Decision

**Apache Arrow Flight is the block data plane framework. Arrow RecordBatch is the
single developer-facing data format.** Block authors never write Protobuf
definitions, never run `protoc`, and never manage two schema systems. Specifically:

- **Arrow Flight RPC** is the transport for out-of-process (gRPC) blocks. Blocks
  exchange Arrow RecordBatches via the `DoExchange` bidirectional streaming RPC.
  Arrow Flight is built on gRPC and uses Protobuf internally for control
  messages (FlightDescriptor, Ticket, Action), but this is encapsulated within
  the Flight SDK — block developers interact only with Arrow types.

- **Arrow Flight custom actions** handle block lifecycle operations (health
  checks, configuration updates, graceful shutdown). These use Flight's `Action`
  / `Result` mechanism, which is already part of the Arrow Flight specification.
  No separate gRPC service definition is needed.

- **Arrow IPC over shared memory** is planned for WASM block boundaries (v2).
  The host process writes an Arrow RecordBatch to shared memory in IPC format,
  and the WASM guest reads it. See ADR-0006 for the phasing of WASM support.

- **Arrow RecordBatch** is the canonical in-memory event batch representation.
  All block processing logic operates on RecordBatches. There is no
  intermediate "event" struct that blocks serialize to/from; the RecordBatch IS
  the event batch.

- **Arrow schemas** are defined at runtime (not compile time), eliminating the
  code generation step. Adding a new field to a RecordBatch is a runtime schema
  change, not a recompile.

This decision is opinionated and deliberately limits format choice. The benefit
is a single, consistent developer-facing data model from source to sink, with
zero conversion overhead at block boundaries. The fact that Arrow Flight uses
Protobuf internally is an implementation detail, not a developer concern.

## Consequences

### Positive

- **Zero format conversion**: Data flows from source to sink without ever
  changing serialization format. This eliminates an entire class of bugs
  (conversion errors, precision loss, schema drift between formats) and removes
  the CPU overhead of format translation at every block boundary.

- **Native financial precision**: Arrow's `Decimal128` type provides exact
  decimal arithmetic for currency amounts, tax rates, and prorated charges.
  No string-encoded decimals, no integer-cents conventions, no floating-point
  surprises.

- **Columnar performance**: Analytical operations (SUM, GROUP BY, FILTER) on
  Arrow columnar data are 10-100x faster than equivalent operations on
  row-oriented formats. This is critical for the rating and aggregation blocks
  that process millions of metering events per hour.

- **Field-level security**: Arrow's columnar layout enables zero-cost field
  projection. The platform can expose only the columns a block has declared in
  its capability manifest, physically preventing access to sensitive fields.
  This is enforced at the memory layout level, not via runtime checks.

- **No code generation**: Adding a new field or modifying a schema does not
  require regenerating stubs in every supported language. Schemas are
  self-describing and travel with the data.

- **Multi-language ecosystem**: Arrow has production-quality implementations in
  Go, Rust, Python (PyArrow), Java, C++, C#, JavaScript, Ruby, and R. Every
  major data processing framework (Pandas, Polars, DuckDB, DataFusion) speaks
  Arrow natively.

### Negative

- **Arrow learning curve**: Developers familiar with Protobuf or JSON will need
  to learn Arrow's columnar data model, schema system, and IPC format. The
  mental model shift from row-oriented to columnar processing is non-trivial.

- **Overhead for small messages**: Arrow IPC has a fixed overhead for schema
  metadata and alignment padding. For very small payloads (< 100 bytes), Arrow
  is less efficient than compact binary formats. Meteridian mitigates this by
  batching events into RecordBatches (typically 1,000-10,000 rows), which
  amortizes the fixed overhead.

- **Arrow Flight Go maturity**: While Arrow Flight is production-proven in
  Python, Java, and C++ (Dremio, InfluxDB, Spark), the Go implementation has
  fewer large-scale production deployments. The Arrow project is actively
  decoupling Flight serialization from gRPC internals (completed in Arrow
  v24.0.0, March 2026), which improves the Go story. We may need to contribute
  upstream fixes for edge cases.

- **Single point of coupling**: The entire block ecosystem depends on Arrow. If
  Arrow's direction diverges from Meteridian's needs, migration would be
  extremely costly. We mitigate this by: (a) Arrow is an Apache Software
  Foundation project with broad industry backing, (b) wrapping Arrow in a thin
  SDK abstraction layer, (c) the Arrow IPC format specification is stable and
  versioned independently of implementations.

### Neutral

- Storage format (Parquet) is also Arrow-native, providing seamless
  serialization to disk. This aligns with but does not depend on the RPC
  format decision.

- Existing Meteridian components that currently use Protobuf will need migration
  to Arrow. This work is tracked separately.

## Alternatives Considered

### Raw gRPC + Protobuf (Developer-Facing Protobuf)

Using hand-written Protobuf service definitions for block-to-block RPC, with
Arrow only for data payloads. Block developers would define `.proto` files for
their block's lifecycle methods and convert between Protobuf and Arrow at
boundaries.

**Rejected because:** This forces block developers to manage two schema
systems — Protobuf for RPC and Arrow for data. Every new block requires
`.proto` files, `protoc` code generation, and conversion logic. This is
exactly the complexity that Arrow Flight was designed to eliminate. Arrow
Flight already provides a complete RPC framework (DoGet, DoPut, DoExchange,
Actions) that is sufficient for all block operations, without requiring
developers to write any Protobuf. The Protobuf used internally by Arrow
Flight is hidden inside the SDK — developers interact only with Arrow types.

Note: Arrow Flight does use Protobuf internally for control messages. This is
an implementation detail of the Flight SDK, not a developer burden. The `vgi-rpc`
project on the Arrow mailing list demonstrates that even this internal Protobuf
can be eliminated for in-process transports (achieving 29 GB/s over shared
memory), but Arrow Flight's gRPC-based approach is the proven production choice.

### Protobuf Only

Using Protocol Buffers as the sole serialization format for both RPC and event
data.

**Rejected because:** Protobuf is row-oriented, which makes it fundamentally
unsuitable for the columnar analytical workloads (aggregation, windowing,
GROUP BY) that dominate metering and billing. Protobuf has no native decimal
type — a dealbreaker for financial data. Code generation is required for every
language, adding build complexity and slowing down the development cycle for
block authors. Batch processing requires wrapping repeated messages, which
adds overhead and prevents zero-copy access patterns.

### Cap'n Proto

A zero-copy serialization format designed for high-performance RPC.

**Rejected because:** While Cap'n Proto is fast and supports zero-copy reads,
its language ecosystem is narrow compared to Arrow. Python support is
community-maintained and lacks the depth of PyArrow. The format is
row-oriented (like Protobuf), providing no benefit for columnar analytics.
Cap'n Proto schemas are compile-time only, requiring code generation. The
community and tooling ecosystem is an order of magnitude smaller than Arrow's.

### FlatBuffers

Google's zero-copy serialization format, widely used in games and mobile
applications.

**Rejected because:** FlatBuffers is designed for random access to individual
records (loading a game asset, reading a single configuration entry), not for
columnar batch processing. There is no concept of a "RecordBatch" or columnar
layout. Analytical operations would require reading every record individually,
defeating the purpose of batch processing. FlatBuffers is excellent at what
it does, but it solves a different problem.

### JSON

Using JSON as the wire format, with optional JSON Schema for validation.

**Rejected because:** JSON has no binary encoding, making it 10-100x larger
than Arrow for numeric data. Parsing JSON is CPU-intensive and dominates
processing time for high-throughput pipelines. JSON has no native types for
timestamps, decimals, or nested structures with enforced schemas. There is
no zero-copy reading — every field must be parsed into application-specific
types. JSON is useful for human-readable configuration (which Meteridian uses
for pipeline YAML and block manifests) but unsuitable as a high-performance
wire format.

## References

- [Apache Arrow Flight](https://arrow.apache.org/docs/format/Flight.html)
- [Apache Arrow IPC Format](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc)
- [vgi-rpc: Arrow-only RPC](https://www.mail-archive.com/dev@arrow.apache.org/msg34470.html) — 29 GB/s shared memory throughput
- [Arrow Decimal128 Type](https://arrow.apache.org/docs/format/Columnar.html#fixed-size-primitive-layout)
- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [Cap'n Proto](https://capnproto.org/)
- [FlatBuffers](https://flatbuffers.dev/)
