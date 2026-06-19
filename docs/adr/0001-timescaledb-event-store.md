# ADR-0001: TimescaleDB as Primary Event Store

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian needs a database capable of ingesting, storing, and querying metering
events at scale — billions of immutable, append-heavy rows that are always queried
by time range and tenant. The workload is a hybrid of high-throughput writes (event
ingestion during rating) and analytical reads (hourly, daily, and monthly rollups
for billing). This combination of OLTP and OLAP characteristics rules out databases
optimized for only one pattern.

The platform must also store relational metadata alongside events: tenant
configurations, cost models, rate cards, balance state, and audit logs. A solution
that handles only time-series data would require a second database for everything
else, doubling operational complexity for on-prem deployments where minimizing the
service footprint is critical. Billing queries frequently join event data with
metadata — for example, rating a CPU usage event requires joining with the tenant's
current cost model, the resource's tag assignments, and the applicable rate tier.
These joins are fundamental to the billing workflow, not optional enrichment steps.

On-premises deployment is a hard requirement. Managed-only or cloud-exclusive
solutions are not viable. The database must run on Kubernetes with automated
failover, backup, and restore — ideally through a mature operator. Licensing must
permit unrestricted on-prem deployment, including by customers who may themselves
be service providers.

## Decision

We will use TimescaleDB as the primary event store, deployed via CloudNativePG on
Kubernetes. Metering events will be stored in hypertables partitioned by time, with
continuous aggregates providing pre-computed hourly, daily, and monthly rollups.
Tenant metadata, cost models, and balance state will live in standard PostgreSQL
tables within the same database instance, using schema-per-tenant isolation.

TimescaleDB provides the best combination of time-series performance and full
relational capabilities. Hypertable compression delivers 10-20x storage savings on
historical event data by applying columnar compression to immutable chunks. This is
particularly important for on-prem deployments where storage is a fixed cost.
Continuous aggregates automatically maintain materialized rollups as new events
arrive, eliminating the need for custom batch jobs to compute billing summaries —
when a new event is inserted, the affected hourly and daily aggregates are
incrementally refreshed without re-scanning the entire dataset.

CloudNativePG provides Kubernetes-native high availability with streaming
replication, automated failover, and point-in-time recovery. It manages the full
lifecycle of PostgreSQL clusters including provisioning, scaling, backup to S3 or
object storage, and rolling upgrades. The Apache 2.0 license (adopted by Timescale
in November 2024) removes any restrictions on distribution, embedding, or offering
as a service, aligning with Meteridian's goal of being fully open-source.

## Consequences

### Positive

- Full PostgreSQL compatibility — Django ORM, pgvector, pg_partman, FDW, PostGIS,
  and the entire PostgreSQL extension ecosystem work unchanged. This is critical for
  Meteridian's Go codebase which uses `pgx` and `sqlc` for database access.
- Hypertable compression reduces event storage by 10-20x through columnar
  compression on immutable chunks. A year of metering data that would require 10TB
  uncompressed fits in under 1TB compressed.
- Continuous aggregates pre-compute daily and monthly rollups incrementally,
  avoiding expensive full-table scans at billing time. Invoice generation queries
  hit the aggregate, not the raw events.
- Chunk exclusion in query planning dramatically improves time-range query
  performance compared to vanilla partitioning — the planner skips irrelevant
  chunks entirely rather than scanning partition metadata.
- CloudNativePG provides automated failover, streaming replication, and
  backup/restore on Kubernetes without manual intervention. Recovery time objective
  (RTO) is typically under 30 seconds.
- Single database for events, metadata, tenant schemas, and balance state — no
  cross-database joins, no distributed transactions, no data synchronization.
- Apache 2.0 license permits unrestricted use, modification, and distribution.
- Mature ecosystem: monitoring (pg_stat_statements, pganalyze), tooling (pgAdmin,
  psql, DBeaver), and operational knowledge are widely available.

### Negative

- Not as fast as ClickHouse for pure OLAP scan workloads — column-oriented storage
  is inherently better for wide analytical scans across many columns. For
  Meteridian's typical queries (filter by time + tenant, aggregate a few columns),
  this difference is modest, but it exists.
- Compression requires careful `chunk_time_interval` tuning — chunks that are too
  small waste metadata overhead and increase the number of chunks the planner must
  consider. Chunks that are too large delay compression (only completed chunks are
  compressed) and increase query latency for recent data.
- CloudNativePG operator is an operational dependency that must be upgraded and
  maintained alongside TimescaleDB. Operator version compatibility with Kubernetes
  versions must be tracked.
- TimescaleDB extensions must be included in the PostgreSQL container image, adding
  build complexity compared to vanilla PostgreSQL. The container image must be
  rebuilt when TimescaleDB releases security patches.

### Neutral

- Hypertables are transparent to the ORM — standard INSERT, SELECT, and JOIN
  operations work without modification. Application code should be aware of chunk
  boundaries for optimal query planning but is not required to handle them
  explicitly.
- Continuous aggregates must be explicitly defined and have their refresh policies
  configured. They are not automatic views over arbitrary queries — each aggregate
  is a deliberate optimization choice.
- Schema-per-tenant isolation is a PostgreSQL pattern, not a TimescaleDB feature.
  The same isolation model would work with vanilla PostgreSQL. TimescaleDB adds
  hypertable and compression benefits within each tenant schema.

## Alternatives Considered

### ClickHouse

ClickHouse excels at pure analytical workloads — columnar storage and vectorized
execution make it significantly faster than TimescaleDB for full-table scans and
aggregations over billions of rows. For read-heavy analytics dashboards, ClickHouse
would deliver lower query latency, especially for queries that scan many columns
across large time ranges.

However, ClickHouse's transactional guarantees are weaker than PostgreSQL's. It
lacks full ACID transactions, making it unsuitable for balance updates, cost model
mutations, and other operations that require read-modify-write consistency. The
rating pipeline needs to atomically insert a rated event and update the tenant's
balance — a pattern that requires transactions spanning multiple tables.

Using ClickHouse for analytics while keeping PostgreSQL for metadata and state
would require operating two databases, implementing cross-database joins (or data
duplication), and managing two backup/restore pipelines. For on-prem deployments,
this doubles operational complexity. The Kubernetes operator ecosystem for
ClickHouse (Altinity Operator) is less mature than CloudNativePG. Additionally,
ClickHouse's license changed to the "ClickHouse Community License" which restricts
offering the software as a managed service — a concern for customers who may
themselves be service providers.

### Vanilla PostgreSQL (with Declarative Partitioning)

Standard PostgreSQL with declarative range partitioning can store time-series data
and is the simplest option operationally. It avoids any extension dependency and
uses only features built into PostgreSQL core. For small deployments (millions of
events), vanilla PostgreSQL with monthly partitions would work adequately.

However, it lacks several features critical for Meteridian's workload at scale:
hypertable compression (manual compression would require custom pg_cron jobs and
careful partition management), continuous aggregates (materialized views must be
manually refreshed and don't support incremental updates natively), and automatic
data retention policies (partition dropping must be scripted). Query planning for
time-range queries is also significantly worse without TimescaleDB's chunk exclusion
optimization — the planner must consider all partitions rather than efficiently
skipping irrelevant chunks. While each missing feature can be individually worked
around, the cumulative effort of building and maintaining these workarounds exceeds
the cost of adopting TimescaleDB.

### CockroachDB

CockroachDB provides distributed SQL with strong consistency across multiple nodes,
which would be valuable for geo-distributed deployments. However, it is designed
for OLTP workloads, not time-series analytics. It has no native compression for
time-series data, no continuous aggregates, and no hypertable abstraction. The
distributed consensus protocol (Raft) adds latency to every write — unnecessary
overhead when CloudNativePG provides sufficient high availability through streaming
replication for single-cluster deployments.

CockroachDB's Business Source License (BSL) restricts offering it as a managed
database service, which conflicts with Meteridian's fully open-source approach. The
operational complexity of managing a distributed consensus cluster is not justified
when the deployment target is a single Kubernetes cluster with HA provided by
CloudNativePG's streaming replication and automated failover.

### QuestDB / InfluxDB

Pure time-series databases like QuestDB and InfluxDB are optimized for ingestion
throughput and time-range queries but lack relational join capabilities. Meteridian's
rating pipeline requires joining events with tenant configurations, cost models,
rate cards, and resource metadata — these joins are fundamental to the billing
workflow, not optional enrichment.

Using a time-series database alongside PostgreSQL would split the data model across
two systems with different query languages, different transactional semantics, and
different operational procedures. This is the same dual-database problem as the
ClickHouse alternative, with the additional downside that these databases have less
mature Kubernetes operators than CloudNativePG. InfluxDB's license (MIT for v1,
proprietary for v3) also presents concerns for an open-source project.

## References

- [TimescaleDB Columnar Compression](https://www.timescale.com/blog/building-columnar-compression-in-a-row-oriented-database/)
- [CloudNativePG](https://cloudnative-pg.io/)
- [Timescale Apache 2.0 License Announcement](https://www.timescale.com/blog/timescale-is-now-free-open-source-and-available-under-the-apache-2-license/)
- [TimescaleDB Continuous Aggregates](https://docs.timescale.com/use-timescale/latest/continuous-aggregates/)
- [CloudNativePG Architecture](https://cloudnative-pg.io/documentation/current/architecture/)
- [TimescaleDB vs ClickHouse Benchmark](https://www.timescale.com/blog/timescaledb-vs-clickhouse-for-time-series-data/)
