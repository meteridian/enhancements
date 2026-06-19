# ADR-0007: Block-Based Dataflow Extensibility Model

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian needs an extensibility model that allows customers and third-party
developers to inject custom logic into the metering-to-billing pipeline. The
platform must support diverse use cases: custom metering collectors for
proprietary infrastructure, data enrichment (geo-lookup, organizational
mapping), custom rating logic (tiered pricing, volume discounts, contract-specific
rates), and custom billing integrations (payment providers, ERP systems, tax
engines).

The metering and billing industry has standardized on two extensibility patterns:
**webhooks** (Stripe, Lago, Orb) and **APIs** (Zuora, Amberflo). Both patterns
push the complexity onto the customer: webhooks require hosting and scaling
external HTTP endpoints, while APIs require writing custom application code that
polls and pushes data. Neither pattern provides schema enforcement, backpressure,
pipeline composition, or built-in observability.

The data engineering domain has a well-established alternative: **dataflow
architectures** where typed processing nodes are connected into directed acyclic
graphs (DAGs). Apache NiFi, Redpanda Connect (formerly Benthos), Node-RED, and
Apache Beam all demonstrate that dataflow composition scales from simple ETL to
complex event-driven systems. The audio processing domain (DAWs) has proven
that typed plugin architectures can sustain thriving third-party ecosystems
over decades.

No billing platform currently offers dataflow-style extensibility. This
represents both an opportunity and a risk — there is no precedent to follow
for the intersection of billing semantics and dataflow architecture.

## Decision

Meteridian adopts a **block-based dataflow extensibility model** where the
fundamental unit of computation is a **processing block**. Blocks have typed
Apache Arrow inputs and outputs, declared capabilities, and standardized
lifecycle, health, and metrics endpoints. Blocks are connected into directed
acyclic graphs (DAGs) called **pipelines**.

Five block types cover all dataflow patterns:

- **Source blocks** produce events (no input, one or more outputs)
- **Transform blocks** modify events (one input, one output)
- **Sink blocks** consume events (one input, no output)
- **Fork blocks** duplicate events to multiple paths (one input, N outputs)
- **Join blocks** merge events from multiple paths (N inputs, one output)

Pipelines are defined declaratively in YAML (or CUE for type safety). The
platform validates schema compatibility across all connections at deployment
time — a pipeline cannot be deployed if a block's output schema is
incompatible with its downstream block's input schema.

Each block declares a **capability manifest** that specifies the network hosts,
Arrow fields, environment variables, and filesystem paths it requires. The
platform grants capabilities based on the block's trust tier (Tier 0-3) and
denies everything else. This deny-by-default model ensures that third-party
blocks cannot access data or resources they haven't explicitly declared.

Blocks are automatically instrumented with OpenTelemetry. Per-block latency,
throughput, error rates, and resource consumption are available without any
instrumentation code from block authors. This makes performance bottleneck
identification trivial: the slowest block in a pipeline is immediately visible
in the observability stack.

## Consequences

### Positive

- **Composability**: Complex metering pipelines are built by connecting simple,
  well-tested blocks. A geo-enrichment block can be combined with a tag
  normalizer, a tiered rater, and a payment provider — each independently
  developed, tested, and versioned. This is fundamentally more powerful than
  webhooks, which can only observe events, not transform them in-pipeline.

- **Type safety**: Arrow schema validation at deployment time catches integration
  errors before they reach production. A block that expects a `source_ip`
  column will fail deployment (not at runtime) if its upstream block does not
  provide that column. This eliminates the "webhook received unexpected JSON"
  class of production incidents.

- **Built-in backpressure**: When a downstream block is slow, the platform
  automatically applies backpressure to upstream blocks. This prevents event
  loss and memory exhaustion without requiring block authors to implement flow
  control. Webhooks, by contrast, have no backpressure — if the receiver is
  slow, events are lost or queued without bound.

- **Marketplace ecosystem**: Typed, versioned blocks with capability manifests
  are inherently marketable. Developers can publish blocks that solve specific
  problems (geo-enrichment, tax calculation, ERP integration) and customers
  can compose them into pipelines without writing code. This creates a network
  effect: more blocks attract more customers, which attracts more block
  developers.

- **Observability for free**: Every block is instrumented automatically.
  Customers can identify exactly which block is the bottleneck, which block
  is producing errors, and which block is consuming the most resources. This
  level of observability is impossible with webhooks, where the external
  endpoint is a black box.

### Negative

- **Complexity for simple use cases**: A customer who just wants a webhook
  notification when an invoice is generated must understand the block model,
  create a pipeline with a sink block, and deploy it. This is more complex
  than a simple webhook subscription. Meteridian should provide convenience
  wrappers (e.g., "quick webhook" that auto-generates the pipeline).

- **DAG limitations**: Directed acyclic graphs cannot express feedback loops.
  Some billing scenarios (e.g., "if payment fails, retry after adjusting the
  amount") require cycles. These must be modeled as separate pipelines with
  external triggers, which is less elegant than a native cycle construct.

- **Learning curve**: The dataflow mental model is unfamiliar to most billing
  engineers. Unlike webhooks (which most developers have used), dataflow
  pipelines require understanding of schemas, connections, backpressure, and
  lifecycle management. Investment in documentation, tutorials, and the AI
  interface is critical for adoption.

- **No existing ecosystem to bootstrap**: Unlike audio plugins (where VST has
  thousands of plugins) or data engineering (where Beam has hundreds of
  transforms), the billing dataflow ecosystem starts from zero. Meteridian must
  ship a comprehensive set of official blocks to make the platform useful
  before the marketplace achieves critical mass.

### Neutral

- The block model is compatible with event-driven architectures. Source blocks
  can consume from Kafka, Valkey Pub/Sub, or CloudEvents endpoints, and sink blocks can
  produce to the same. The dataflow model does not replace event-driven
  architecture; it composes on top of it.

- Pipeline YAML definitions are GitOps-friendly and can be managed via Flux or
  ArgoCD for Kubernetes-native deployments.

## Alternatives Considered

### Webhooks and Event Subscriptions

The industry standard: users subscribe to events (e.g., `invoice.created`,
`usage.recorded`) and receive HTTP POST callbacks at their own endpoints.

**Rejected as the primary model because:** Webhooks are fire-and-forget with
no backpressure — if the receiver is slow or down, events are lost or retried
blindly. There is no schema enforcement beyond "here's some JSON." Webhooks
cannot transform data in-pipeline; they can only observe and react externally.
There is no composability — you cannot chain webhooks into a pipeline. There
is no built-in observability — the webhook endpoint is a black box.

Webhooks are adequate for notifications ("an invoice was created") but
fundamentally insufficient for data transformation ("enrich this usage event
with geolocation, apply tiered pricing, and route to the correct payment
provider"). Meteridian will still support webhooks as a specific sink block
type for backward compatibility, but webhooks are not the extensibility model.

### Middleware Chain (Express and Django Style)

A linear chain of middleware functions, each of which can modify the request
and pass it to the next middleware. Used by Express.js, Django, and many web
frameworks.

**Rejected because:** Middleware chains are strictly linear — there is no
branching, forking, or joining. Metering pipelines need to fork events to
multiple destinations (storage AND invoice AND analytics), join events from
multiple sources (AWS + Azure + OCP), and branch based on event type. A linear
chain cannot express these topologies. Additionally, middleware chains are
tightly coupled to the request/response pattern, which doesn't fit
stream-oriented metering data.

### Stored Procedures and User-Defined Functions (UDFs)

Allowing users to define custom SQL functions or stored procedures that run
inside the database (TimescaleDB).

**Rejected because:** UDFs are database-coupled and cannot run outside the
database context. They have no process isolation (a buggy UDF can crash or
stall the database), limited language support (PL/pgSQL, PL/Python, maybe
PL/Rust), and no lifecycle management (versioning, health checks, metrics).
UDFs are appropriate for simple data transformations within SQL queries but
not for the full spectrum of extensibility (HTTP integrations, ML inference,
payment processing).

### Sidecar Pattern

Each block as a sidecar container attached to the main Meteridian pod, with
communication via localhost networking.

**Rejected because:** Each sidecar container adds Kubernetes resource overhead
(scheduling, networking, memory for container runtime). A pipeline with 10
blocks would require 10 sidecar containers, consuming significant cluster
resources. Service mesh complexity (mTLS, traffic routing) adds operational
burden. The sidecar pattern is appropriate for cross-cutting concerns (logging,
monitoring proxies) but too heavyweight for per-block deployment. The gRPC
runtime (ADR-0006) provides container isolation when needed, but as an
opt-in for specific blocks, not as the default for all blocks.

## References

- [Apache NiFi](https://nifi.apache.org/) — Dataflow automation platform
- [Redpanda Connect](https://www.redpanda.com/connect) — Stream processing with composable processors
- [Node-RED](https://nodered.org/) — Flow-based programming for event processing
- [Apache Beam](https://beam.apache.org/) — Unified batch and stream processing model
- [VST Plugin Standard](https://steinbergmedia.github.io/vst3_dev_portal/) — Audio plugin architecture (precedent for block ecosystems)
- [CloudEvents](https://cloudevents.io/) — Event format specification
