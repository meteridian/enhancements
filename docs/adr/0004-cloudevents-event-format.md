# ADR-0004: CloudEvents as Canonical Event Format

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian ingests metering data from dozens of heterogeneous sources: Prometheus
metrics, Kubernetes resource events, SNMP counters, vSphere performance data, cloud
billing APIs (AWS CUR, Azure Cost Exports, GCP BigQuery), and custom collectors
built by customers. Each source produces data in its own format with different field
names, timestamp representations, unit conventions, and metadata structures. Without
a canonical format, every pipeline stage must understand every source format,
creating an N×M integration problem.

The platform needs a canonical event format for the internal pipeline that spans
ingestion, validation, deduplication, enrichment, rating, and storage. This format
must carry standardized metadata (source identity, event timestamp, event type,
tenant context) alongside the metering payload. It must support schema evolution —
new metering sources and event types will be added continuously over the platform's
lifetime — without breaking existing consumers in the pipeline. Backward
compatibility is essential: adding a new event type should not require changes to
existing pipeline stages that don't process that type.

The format must also be transport-agnostic. Meteridian supports multiple ingestion
transports: HTTP push from collectors, Kafka for high-throughput streaming, NATS for
lightweight pub/sub, and file upload for batch imports. The canonical format must
work efficiently across all of them without transport-specific serialization logic.
Finally, the format should be an industry standard where possible to reduce
integration friction for customers connecting their existing monitoring and metering
systems to Meteridian.

## Decision

We will adopt the CloudEvents v1.0 specification as the envelope format for all
metering events in Meteridian's internal pipeline. Every metering event, regardless
of its source, will be wrapped in a CloudEvents envelope with standardized context
attributes (`id`, `source`, `type`, `time`, `specversion`) before entering the
processing pipeline.

Billing-specific metadata will use CloudEvents extension attributes with the
`metr:` prefix. Core extensions include `metr:tenant` (tenant identifier),
`metr:resource_type` (the type of resource being metered), `metr:unit` (unit of
measure), and `metr:quantity` (usage amount). These extensions follow CloudEvents'
extension mechanism, which allows custom attributes without departing from the
specification. Pipeline stages that don't need billing metadata can ignore the
`metr:*` extensions and still route, filter, and dead-letter events based on
standard context attributes.

Event payloads — the actual metering data — will be carried as CloudEvents data
using Apache Arrow RecordBatches serialized in IPC format for batch events, and
JSON for single events. This combines CloudEvents' standardized metadata envelope
with Arrow's efficient columnar representation for the metering data itself. The
CloudEvents `datacontenttype` attribute will be set to
`application/vnd.apache.arrow.stream` for Arrow payloads and `application/json` for
JSON payloads, enabling consumers to handle both formats transparently.

CloudEvents is a CNCF graduated project with broad industry adoption, SDKs in all
major languages, and built-in support in event brokers (Knative, Azure Event Grid,
Google Eventarc). The specification is transport-agnostic with defined protocol
bindings for HTTP, Kafka, AMQP, NATS, and WebSockets.

## Consequences

### Positive

- Industry standard (CNCF graduated) with wide adoption — customers integrating
  with Meteridian can use familiar tooling and existing CloudEvents SDKs in Go,
  Python, Java, Rust, JavaScript, and C#.
- Standardized context attributes (`id`, `source`, `type`, `time`) provide
  consistent metadata across all event sources. Pipeline stages can route, filter,
  and dead-letter events based on these attributes without parsing the payload.
- Extension mechanism (`metr:*` attributes) allows billing-specific metadata
  without departing from the specification. New extensions can be added without
  breaking existing consumers that don't use them.
- Transport-agnostic design — the same event format works over HTTP, Kafka, AMQP,
  NATS, and file-based ingestion without format conversion. Protocol bindings
  define exactly how CloudEvents attributes map to transport-specific headers.
- SDK support in Go, Python, Java, Rust, JavaScript, C# — reduces integration
  effort for custom collectors. Customers building metering agents can use the
  CloudEvents SDK for their language rather than implementing a custom protocol.
- JSON structured content mode provides human-readable events for debugging and
  development; binary content mode with Arrow payloads provides efficient
  serialization for production batch workloads.
- Event routers, dead-letter queues, and observability middleware can inspect
  CloudEvents attributes without parsing the payload — enabling generic event
  infrastructure that works across all event types.
- The `type` attribute serves as the event schema identifier (e.g.,
  `io.meteridian.metering.cpu.v1`), providing a natural extension point for schema
  registry integration and version-aware routing.

### Negative

- Envelope overhead of approximately 200 bytes per event for CloudEvents context
  attributes. At billions of events per day, this adds measurable storage and
  bandwidth cost, though compression (gzip, zstd) largely eliminates the overhead
  in transit and at rest.
- CloudEvents SDKs vary in quality and completeness across languages — the Go and
  Java SDKs are mature and well-maintained, but less common languages may have gaps
  in protocol binding support or extension handling.
- Binary content mode with Arrow payloads requires custom `datacontenttype`
  handling — standard CloudEvents middleware that expects JSON data will not be able
  to inspect Arrow-encoded payloads without additional configuration.
- Extension attributes (`metr:*`) are not validated by generic CloudEvents
  tooling — validation of billing-specific attributes must be implemented in
  Meteridian's ingestion layer. This includes type checking, required-field
  enforcement, and semantic validation.

### Neutral

- CloudEvents does not prescribe payload format — the choice to use Apache Arrow
  RecordBatches as the batch payload format is a Meteridian-specific decision
  layered on top of the CloudEvents envelope. Single events use JSON payloads for
  simplicity.
- CloudEvents supports both structured content mode (metadata and data in a single
  JSON document) and binary content mode (metadata in transport headers, data in
  body). Meteridian will use binary mode for Arrow payloads (metadata in HTTP
  headers or Kafka headers, Arrow data in body) and structured mode for JSON
  payloads (everything in one JSON document).
- The CloudEvents `subject` attribute can optionally carry the specific resource
  identifier being metered (e.g., a pod name, VM ID, or storage volume), providing
  fine-grained routing without custom extensions.

## Alternatives Considered

### Custom Event Format (Raw JSON)

Defining a custom JSON event format would avoid any external specification
dependency and allow the format to be tailored exactly to Meteridian's needs. The
schema could include billing-specific fields as first-class citizens rather than
extensions.

However, custom formats lack standardized metadata fields — every consumer in the
pipeline must know which ad-hoc fields carry the timestamp, source identity, event
type, and tenant context. These implicit contracts become a maintenance burden as
the number of event sources and pipeline stages grows. Schema evolution with custom
JSON is also problematic: without a formal specification, there is no standard
mechanism for versioning, backward compatibility, or discovery. Every format change
requires coordinating all producers and consumers. CloudEvents solves these problems
with a stable, versioned specification, a defined extension mechanism, and
transport-agnostic protocol bindings that would need to be reinvented for a custom
format.

### Avro

Apache Avro provides compact binary serialization with schema evolution support
through a schema registry. Its binary encoding is more efficient than JSON for high-
throughput scenarios, and the schema registry ensures that producers and consumers
agree on the event structure.

However, Avro requires a running schema registry service (typically Confluent Schema
Registry), which adds operational complexity for on-prem deployments. The Avro
ecosystem is heavily tied to Kafka — using Avro outside of Kafka (e.g., over HTTP
or NATS) requires additional tooling and non-standard libraries. Avro's schema
evolution model is more rigid than CloudEvents' extension mechanism: adding new
metadata fields requires updating the schema in the registry and ensuring all
consumers can handle the new schema version before any producer starts using it.
CloudEvents extension attributes can be added freely without breaking existing
consumers. Additionally, Avro's binary format is opaque for debugging — developers
cannot inspect events without the schema and a deserialization tool.

### Pure Protobuf Events

Protocol Buffers provide efficient binary serialization with strong typing, code
generation, and mature tooling. gRPC integration provides a natural transport
binding with built-in streaming, flow control, and error handling.

However, Protobuf requires codegen for every schema change — adding a new event
type or metadata field means regenerating client libraries in all supported
languages and recompiling all consumers. This slows down the pace of adding new
metering sources, which is expected to happen frequently as Meteridian integrates
with new platforms. Protobuf also lacks a standard metadata envelope — we would need
to define our own wrapper message for context attributes (source, timestamp, type,
tenant), effectively reinventing CloudEvents in Protobuf without the benefit of the
CloudEvents ecosystem (SDKs, middleware, event brokers). Schema evolution in
Protobuf is forward-compatible only — removing or retyping fields breaks backward
compatibility.

### OpenTelemetry Log Format

OpenTelemetry provides a standardized format for logs, metrics, and traces with
broad industry adoption and a mature collector ecosystem. The OTel Collector could
serve as Meteridian's ingestion pipeline, transforming metrics from various sources
into a common format.

However, the OTel data model is designed for observability, not metering. It lacks
billing-specific concepts: usage quantity with billing precision, unit of measure
tied to pricing, rated cost, tenant identity for multi-tenant billing, and balance
impact. While OTel's `Resource` and `Attribute` fields could carry this information
as free-form key-value pairs, doing so would stretch the format far beyond its
intended purpose and lose the semantic guarantees that a billing-specific format
provides. Metering events are business events that drive financial transactions, not
observability signals — conflating the two creates ambiguity about data retention
policies, access controls, and processing semantics.

## References

- [CloudEvents Specification](https://cloudevents.io/)
- [CloudEvents GitHub](https://github.com/cloudevents/spec)
- [CloudEvents SDK — Go](https://github.com/cloudevents/sdk-go)
- [CloudEvents Primer](https://github.com/cloudevents/spec/blob/main/cloudevents/primer.md)
- [CloudEvents Kafka Protocol Binding](https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/kafka-protocol-binding.md)
- [Apache Arrow IPC Format](https://arrow.apache.org/docs/format/Columnar.html#ipc-streaming-format)
- [CNCF CloudEvents Graduation](https://www.cncf.io/projects/cloudevents/)
