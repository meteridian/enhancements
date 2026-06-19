# ADR-0011: At-Least-Once Delivery with Idempotency Keys

## Status

**Accepted** — 2026-06-18

## Deciders

- @pgarciaq
- @jordigilh

## Context

Meteridian's block-based dataflow (see [METR-0002](../../enhancements/0002-extensibility/extensibility.md))
processes financial metering events where data loss is unacceptable. Every usage
event ultimately maps to revenue: a dropped event is lost revenue, an
under-counted resource is an under-billed customer. The system must guarantee
that every event is processed.

True "exactly-once" semantics are impossible in distributed systems. The FLP
impossibility result proves that no deterministic protocol can guarantee
consensus in an asynchronous system where even a single process may crash.
Kafka's widely marketed "exactly-once" is, under the hood, at-least-once
delivery combined with idempotent producers and transactional consumers — it
achieves *effectively-once* semantics, not true exactly-once.

This is a well-understood problem in billing systems. Stripe, AWS Billing,
and Google Cloud Billing all use the same fundamental approach: deliver events
at least once and deduplicate at the consumer using idempotency keys. This ADR
formalizes that pattern for Meteridian's block-based pipeline.

### Problem Statement

Given that blocks in the pipeline perform state-mutating operations (database
writes, balance updates, invoice generation), and that network partitions,
process crashes, and redeliveries are inevitable, the platform needs a
consistent strategy for:

1. Ensuring no event is silently dropped.
2. Ensuring duplicate deliveries do not cause double-counting or double-billing.
3. Handling events that persistently fail processing.
4. Supporting multi-instance block topologies without introducing cycles.

## Decision

### At-Least-Once Delivery

All inter-block communication in the pipeline uses at-least-once delivery
semantics. An event is only acknowledged (removed from the upstream block's
outbox) after the downstream block has durably received it. If acknowledgement
fails, the event is redelivered.

### Idempotency Keys

Every metering event carries an idempotency key via the CloudEvents `id` field
(see [ADR-0004](0004-cloudevents-event-format.md)). This key is a globally unique
identifier assigned at event ingress and immutable throughout the pipeline.

Blocks that perform state-mutating operations — database writes, balance
updates, invoice generation, ledger entries — **MUST** be idempotent. Before
applying a change, the block checks the idempotency key against its state store.
If the key has already been processed, the block returns the cached result
without re-applying the mutation.

### IdempotencyStore Interface

The platform provides an `IdempotencyStore` interface backed by the block's
state store (see METR-0002, section on stateful blocks). This gives block
authors a consistent API for idempotency checks without requiring each block
to implement its own deduplication logic.

The interface supports:

- **Check**: Has this idempotency key been processed?
- **Mark**: Record that this key has been processed, along with the result.
- **Expiry**: Keys expire after a configurable TTL (default: 7 days) to bound
  storage growth.

### Dead-Letter Queues (DLQ)

Events that fail processing after configurable retries are routed to a
tenant-scoped Dead-Letter Queue. DLQ behavior:

- **Retry strategy**: Exponential backoff with base interval of 1 second,
  maximum interval of 60 seconds, and random jitter of 0–500ms.
- **Max retries**: Configurable per block (default: 3).
- **DLQ event payload**: The original event, error details (message, stack
  trace), the block ID where failure occurred, and the retry count.
- **DLQ operations**: Events can be replayed back into the pipeline, inspected
  via API, or manually resolved (acknowledged without reprocessing).
- **Tenant isolation**: Each tenant has its own DLQ namespace. A noisy tenant's
  DLQ cannot affect other tenants' processing.

### DAG Topology Constraints

The block pipeline is a Directed Acyclic Graph (DAG). Multiple instances of the
same block **type** may appear in a single pipeline — for example, a "sanitizer"
block at the beginning and end of a pipeline as two separate instances with
independent state and independent idempotency stores. Loops are prohibited at
the DAG validation layer to prevent infinite reprocessing cycles.

### Retry Strategy Details

| Parameter      | Default | Configurable |
|----------------|---------|--------------|
| Base interval  | 1s      | Yes          |
| Max interval   | 60s     | Yes          |
| Jitter range   | 0–500ms | Yes          |
| Max retries    | 3       | Yes (per block) |
| Idempotency TTL| 7 days  | Yes          |

## Consequences

### Positive

- **No data loss**: At-least-once delivery guarantees every event reaches every
  block in the pipeline, even across crashes and restarts.
- **Effectively-once semantics**: The combination of at-least-once delivery and
  idempotency keys achieves the practical equivalent of exactly-once processing
  without the latency cost of distributed transactions.
- **Consistent deduplication**: The platform-provided `IdempotencyStore` ensures
  all blocks use the same deduplication mechanism, reducing the surface area for
  block-author errors.
- **Operational visibility**: DLQs provide a clear mechanism for inspecting and
  resolving failed events rather than silently dropping them.
- **Flexible topology**: Allowing multiple instances of the same block type
  enables multi-pass processing patterns without introducing cycles.

### Negative

- **Storage overhead**: Idempotency keys must be stored for the TTL window. For
  high-throughput tenants, this can be significant (millions of keys × 7 days).
  The TTL-based expiry bounds this cost.
- **Latency overhead**: Every state-mutating operation requires an idempotency
  check (one read) before processing. This adds a constant per-event overhead,
  mitigated by batching checks where possible.
- **Block author responsibility**: Block authors must correctly implement
  idempotency for state-mutating blocks. The `IdempotencyStore` interface
  reduces but does not eliminate this burden.

### Neutral

- **DLQ management**: Tenants (or operators) must monitor and manage DLQ events.
  This is an operational concern that exists regardless of the delivery
  guarantee chosen.
- **Ordering**: At-least-once delivery does not guarantee ordering. Blocks that
  require ordered processing must implement their own ordering logic (e.g.,
  sequence numbers, event timestamps). This is orthogonal to this ADR.

## Alternatives Considered

### Exactly-Once via Two-Phase Commit

Two-phase commit (2PC) can provide exactly-once semantics across distributed
participants. However, 2PC requires synchronous coordination between all
participants, is not partition-tolerant (blocks progress during network
partitions), and introduces unacceptable latency for a high-throughput metering
pipeline. Kafka's "exactly-once" avoids 2PC by using idempotent producers and
transactional consumers — which is effectively what this ADR proposes at the
block level.

**Rejected**: Unacceptable latency and availability trade-offs.

### At-Most-Once Delivery

At-most-once delivery (fire-and-forget) eliminates the need for idempotency
checks entirely. However, it means events can be silently dropped during
failures. For a billing system, lost events directly translate to lost revenue
or incorrect invoices. No production billing system uses at-most-once delivery
for financial events.

**Rejected**: Unacceptable for financial data; lost events = lost revenue.

### Manual Duplicate Detection per Block

Each block author could implement their own duplicate detection logic without
a platform-provided interface. This leads to inconsistent deduplication
strategies, duplicated effort, and a higher probability of bugs. The
`IdempotencyStore` interface standardizes this concern.

**Rejected**: Error-prone and inconsistent across blocks.

## References

- [Kafka Exactly-Once Semantics](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) — Confluent's explanation of how Kafka achieves effectively-once via idempotent producers
- [CloudEvents Specification](https://cloudevents.io/) — The `id` attribute used as the idempotency key
- [Stripe Idempotency Keys](https://stripe.com/docs/api/idempotent_requests) — Industry reference for idempotency in financial APIs
- [FLP Impossibility Theorem](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) — Fischer, Lynch, Paterson (1985): impossibility of distributed consensus with one faulty process
- [ADR-0004: CloudEvents Envelope](0004-cloudevents-event-format.md)
- [METR-0002: Block-Based Extensibility](../../enhancements/0002-extensibility/extensibility.md)
