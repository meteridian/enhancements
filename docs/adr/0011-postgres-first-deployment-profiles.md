# ADR-0011: Postgres-First Deployment Profiles and Upgrade Triggers

- **Status:** Accepted
- **Date:** 2026-06-25
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian targets both sovereign on-prem operators with minimal infrastructure
([FR-1102](../../enhancements/0000-requirements/requirements.md), [US-020](../../enhancements/0000-requirements/requirements.md))
and telco-grade service providers that need sub-millisecond balance enforcement
([FR-302](../../enhancements/0000-requirements/requirements.md)) at high throughput
([FR-1103](../../enhancements/0000-requirements/requirements.md)).

Early architecture drafts stated an invariant of "PostgreSQL + Valkey for all
profiles." That contradicted the Micro requirement (single binary, no Redis/Valkey)
and the product goal of on-premises-first deployment. Operators should be able to
stand up Meteridian on one RHEL server with PostgreSQL alone, then add Valkey, an
event bus, TimescaleDB, or ClickHouse when scale demands it — not on day one.

The [Postgres Is Enough](https://postgresisenough.dev/) community documents
successful production systems built on PostgreSQL alone. Meteridian adopts that
**start simple** philosophy for Micro while preserving [ADR-0002](0002-valkey-balance-management.md)
(Valkey balances) and [ADR-0001](0001-timescaledb-event-store.md) (TimescaleDB
event store) as the **Standard+** defaults. These decisions are complementary, not
contradictory: Micro proves correctness with PostgreSQL backends; Standard+ optimizes
latency and throughput with specialized stores.

## Decision

### 1. Formalize Micro as PG-only mode

The **Micro** profile runs as a single Go binary plus PostgreSQL. No Valkey, NATS,
Kafka, or other stateful services are required. Hot-path concerns — balances,
idempotency, enrichment cache — use PostgreSQL tables and in-process caches.

Micro satisfies air-gapped, single-server deployments for clusters up to roughly
50 nodes and 5K events/sec. Balance query latency may exceed FR-302 p99 targets;
that is an accepted trade-off documented in [METR-0001 §3](../../enhancements/0001-architecture/architecture.md).

### 2. Interface-driven backends

The rating pipeline depends on three pluggable interfaces:

| Interface | Responsibility | Micro default | Standard+ default |
|-----------|----------------|---------------|-------------------|
| `BalanceStore` | Wallet read, atomic check-and-decrement | PostgreSQL row locks | Valkey Lua ([ADR-0002](0002-valkey-balance-management.md)) |
| `IdempotencyStore` | Exactly-once deduplication keys | PostgreSQL table + TTL job | Valkey SET NX + TTL |
| `EventTransport` | Decouple ingest from rating workers | In-process channel | NATS JetStream or Kafka |

Implementations are selected via deployment configuration. Conformance tests must
verify identical rated output regardless of backend.

### 3. Upgrade triggers (when to add components)

| Component | Add when | Do not add preemptively |
|-----------|----------|-------------------------|
| **Valkey** | Multi-replica rating; PG wallet lock contention; balance p99 > 50ms; concurrent prepaid races | Single-replica Micro with low event rate |
| **Event bus (NATS/Kafka)** | Ingest p99 > 500ms; sustained > 10K events/sec; need replay buffer for rolling upgrades | Micro single-process ingest+rate |
| **TimescaleDB** | Event volume > 100M rows; rollup queries > 5s; need chunk compression | Micro with < 1 year retention on one PG node |
| **ClickHouse** | Enterprise OLAP; sub-second ad-hoc analytics over billions of rows | Operational billing queries served by PG/Timescale |
| **Citus** | Write TPS > 50K sustained; PG storage > 10TB | Standard multi-tenant on single CNPG cluster |

Upgrades are additive: swap interface implementations, migrate warm state (e.g.,
PostgreSQL balance projections → Valkey), enable new Helm subcharts. No application
rewrite.

### 4. Profile storage matrix

| Profile | PostgreSQL | TimescaleDB | Valkey | Event bus | ClickHouse |
|---------|------------|-------------|--------|-----------|------------|
| Micro | Required (all state) | — | — | — | — |
| Small | Required | Extension on same instance | Required | Optional | — |
| Standard | Required (CNPG HA) | Required | Required | Required | — |
| Enterprise | Required (multi-AZ) | Required | Cluster | Required | Optional OLAP |
| Hyperscale | Citus shard | Per shard | Cluster | Required | Optional |

## Consequences

### Positive

- Micro deployments match operator expectations: two processes (PG + binary), no
  Redis licensing or ops burden on day one.
- Clear upgrade path preserves investment — PostgreSQL remains system of record;
  Valkey is a projection layer, not a second source of truth.
- Interface contracts enable testing Micro correctness without Valkey in CI.
- Aligns with postgres-is-enough adoption without abandoning telco-grade Standard target.

### Negative

- Team must implement and maintain PostgreSQL `BalanceStore` / `IdempotencyStore`
  backends with acceptable correctness under concurrency (row locks, advisory locks).
- Micro balance latency will not meet FR-302 at high concurrency — must document
  profile limits in runbooks and capacity planning guides.
- Two code paths per interface increase test matrix size; conformance tests are mandatory.

### Neutral

- ADR-0002 and ADR-0001 remain valid for Small+; this ADR scopes their applicability.
- Existing Helm charts gain profile-aware conditionals; Micro chart is minimal (PG + app).

## Alternatives Considered

### Require Valkey for all profiles (rejected)

Simplest implementation (one balance backend) but violates FR-1102/US-020 and blocks
air-gapped single-server pilots. Operators would reject the platform before seeing value.

### Drop Valkey entirely; PostgreSQL for all profiles (rejected)

Aligns with postgres-is-enough maximalism but cannot meet FR-302 and atomic prepaid
enforcement at Standard throughput. Valkey Lua scripts remain the right tool for
multi-replica hot-path balance updates.

### Embedded SQLite for Micro instead of PostgreSQL (rejected)

Reduces external dependencies further but breaks schema-per-tenant model, HA story,
and operator familiarity. PostgreSQL is already the mandatory system of record.

## References

- [METR-0001: Platform Architecture §3](../../enhancements/0001-architecture/architecture.md)
- [Postgres Is Enough](https://postgresisenough.dev/)
- [ADR-0001: TimescaleDB Event Store](0001-timescaledb-event-store.md)
- [ADR-0002: Valkey Balance Management](0002-valkey-balance-management.md)
- [ADR-0004: CloudEvents Event Format](0004-cloudevents-event-format.md)
