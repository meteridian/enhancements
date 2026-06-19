# ADR-0002: Valkey for Real-Time Balance Management

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian must enforce budget caps and prepaid balance limits in real-time during
event rating. When a metering event arrives, the rating pipeline must check the
tenant's remaining balance, verify that the rated cost does not exceed the cap, and
atomically decrement the balance — all before acknowledging the event. If this
balance check is slow or non-atomic, either the billing pipeline becomes a
bottleneck or tenants can exceed their budgets through concurrent requests that each
individually pass the check before any single decrement is written.

The rating hot path targets ≤1.3s p99 end-to-end latency from event ingestion to
rated storage. Balance checks account for approximately 40% of the operations in
the rating path (read balance, evaluate cap, decrement, write updated balance). At
peak load, hundreds of tenants may be rated concurrently, with thousands of events
per second distributed across them. Each balance check must complete in
sub-millisecond time to stay within the latency budget. Adding even 5ms of latency
per balance check would push the p99 beyond the target under load.

Balances must survive process restarts but can tolerate brief (seconds) staleness
during recovery. The event store (TimescaleDB) is the authoritative source of truth
for all financial data — balance entries in the cache are projections that can be
reconstructed by replaying rated events. This means the cache needs durability for
operational convenience (avoiding full reconstruction on routine restarts) but not
the same ACID guarantees as the event store. In the event of a cache failure, the
system can continue processing events and reconcile balances once the cache is
restored, since the event store captures every transaction.

## Decision

We will use Valkey (the Linux Foundation fork of Redis) as an in-memory store for
real-time balance tracking. Each tenant's balance will be stored as a Valkey key
with the current remaining amount as the value. Atomic check-and-decrement
operations will be implemented as Lua scripts executed server-side, ensuring that
balance reads, cap evaluation, and decrements happen as a single atomic operation
without race conditions.

The core Lua script for balance enforcement follows this pattern: read the current
balance, compare it against the rated event cost, and if sufficient funds exist,
decrement the balance and return the new value. If insufficient funds remain, return
an error code that the rating pipeline interprets as a budget cap breach. Because
Lua scripts execute atomically on the Valkey server, there is no window between
the read and decrement where a concurrent request could observe stale state. This
eliminates the race condition that would exist with separate GET and DECRBY commands.

We chose Valkey over Redis due to licensing. Redis Ltd changed Redis's license to
RSALv2/SSPL in March 2024, which prohibits offering Redis as a managed service and
has ambiguous terms for embedded use in platforms that customers may themselves
offer as services. Valkey uses the BSD-3 license under Linux Foundation governance,
with no such restrictions. Functionally, Valkey is a drop-in replacement for
Redis — same protocol, same data structures, same Lua scripting, same client
libraries.

## Consequences

### Positive

- Sub-millisecond balance reads and decrements (~0.1-0.3ms per operation), well
  within the rating pipeline's latency budget. This leaves ample headroom for other
  rating path operations (rule evaluation, event enrichment, storage write).
- Lua scripts provide atomic check-and-decrement operations — no race conditions
  between concurrent rating workers checking the same tenant's balance. This is
  critical for correctness: without atomicity, two concurrent events could each
  pass the balance check and both decrement, overdrawing the account.
- Valkey Sentinel mode provides automatic failover for high availability in
  single-site deployments; Cluster mode is available for horizontal scaling if a
  single Valkey instance cannot handle the balance operation throughput.
- BSD-3 license under Linux Foundation governance — no restrictions on
  distribution, embedding, or offering as a service. This aligns with Meteridian's
  open-source licensing requirements.
- Familiar operational model for teams already running Redis — same CLI tools
  (`valkey-cli`), same monitoring (INFO command, Prometheus exporter), same
  configuration patterns (valkey.conf). Operational runbooks and troubleshooting
  knowledge transfer directly.
- AOF (Append-Only File) persistence provides durability across restarts without
  requiring full event store reconstruction. With `appendfsync everysec`, at most
  one second of balance updates are lost on crash.
- Memory-efficient for balance data — approximately 100 bytes per balance entry.
  Even at 1 million tenant balances, total memory usage is approximately 100MB,
  well within the capacity of a modest Valkey instance.
- Natural TTL support allows inactive tenants' balance entries to expire
  automatically, providing self-managing memory usage. Expired balances are
  reconstructed on demand from the event store when the tenant becomes active again.

### Negative

- Adds another stateful service to the deployment topology, increasing operational
  complexity. However, most Kubernetes platforms already run a Redis-compatible
  cache for other purposes (session storage, API caching), so the marginal
  operational burden is lower than introducing a completely new technology.
- After a Valkey restart or data loss, balances must be reconciled from the event
  store. This reconstruction process must be automated and tested regularly. For
  large tenants with millions of events, reconstruction can take minutes.
- Memory is the scaling constraint — extremely large deployments with tens of
  millions of balance entries may need Valkey Cluster mode, which adds operational
  complexity (slot management, resharding).
- Lua scripts must be carefully written and tested — bugs in atomic balance logic
  could cause incorrect billing (overdrawing or under-drawing accounts). Scripts
  should be version-controlled and covered by integration tests.

### Neutral

- Valkey is wire-compatible with Redis — existing Redis client libraries (go-redis,
  redis-py, Jedis), monitoring tools (Redis Exporter for Prometheus), and
  operational procedures work unchanged. No code changes are needed when migrating
  from Redis to Valkey.
- Balance entries have a natural TTL model: active tenants' balances are kept warm
  in cache, while inactive tenants expire and are reconstructed on demand. This
  provides automatic memory management without explicit eviction logic.
- The reconciliation process (replay rated events to reconstruct balances) also
  serves as an audit mechanism — any discrepancy between the cached balance and the
  reconstructed balance indicates a bug in the rating pipeline or a missed event.

## Alternatives Considered

### PostgreSQL (Same Database as Event Store)

Using the same TimescaleDB instance for balance management would eliminate an
additional service dependency and keep all state in a single database. For low
concurrency scenarios, this approach works adequately — a simple `SELECT ... FOR
UPDATE` on the balance row provides atomicity.

However, row-level locks on balance rows under concurrent rating create contention.
Even with `SELECT FOR UPDATE SKIP LOCKED`, p99 latency for balance checks would be
5-15ms — an order of magnitude slower than Valkey's sub-millisecond operations.
Under high concurrency (thousands of events per second across hundreds of tenants),
database connection pool exhaustion becomes a real risk. Each balance check holds a
connection and a row lock for the duration of the transaction. The rating pipeline
would need to batch balance updates (adding complexity and latency) or accept
significantly higher tail latency. PostgreSQL advisory locks are an alternative but
add complexity and still cannot match in-memory performance for high-throughput
counter operations.

### In-Process Memory (Go map + mutex)

An in-process Go map with mutex synchronization would provide the fastest possible
balance lookups (nanoseconds, no network round trip). For a single-replica
deployment, this is the simplest and fastest option.

However, this approach fails in multi-replica deployments — each replica would have
its own balance state, leading to inconsistencies and potential budget overruns when
events for the same tenant are processed by different replicas. Sticky sessions or
consistent hashing could route all events for a tenant to the same replica, but this
creates hot spots and makes scaling difficult. Balances would also be lost on every
process restart, requiring full reconstruction from the event store. With billions
of events, this reconstruction could take minutes to hours, during which the rating
pipeline either blocks or operates without balance enforcement.

### etcd

etcd is a distributed key-value store designed for configuration management and
coordination in Kubernetes clusters. While it provides strong consistency through
Raft consensus, it is fundamentally designed for low-throughput, high-consistency
workloads — not high-throughput counter operations.

etcd's write throughput is limited to approximately 10K writes per second (vs
Valkey's 100K+), and the Raft consensus protocol adds latency to every write.
etcd's value size limit (1.5MB) and total database size limit (default 2GB,
recommended max 8GB) are unnecessarily restrictive for balance management. Using
etcd for high-frequency balance counters would conflate cluster coordination and
application data concerns, making capacity planning and troubleshooting more
difficult and risking the stability of the Kubernetes control plane if etcd is
shared.

### Redis (Original)

Redis is functionally identical to Valkey for balance management use cases — same
data structures, same Lua scripting, same performance characteristics. The decision
to use Valkey over Redis is purely a licensing concern.

Redis Ltd changed the Redis license to RSALv2/SSPL in March 2024, which prohibits
offering Redis as a managed service and contains ambiguous terms about embedded use
in platforms that customers may themselves offer as services. Since Meteridian is
designed for on-prem deployment by customers who may themselves be service
providers, the RSALv2/SSPL license creates legal uncertainty. Valkey (BSD-3, Linux
Foundation) has no such restrictions and maintains full wire compatibility with
Redis, making it a straightforward substitution with zero code changes.

## References

- [Valkey Project](https://valkey.io/)
- [Valkey GitHub Repository](https://github.com/valkey-io/valkey)
- [Redis License Change Announcement](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)
- [Valkey Lua Scripting](https://valkey.io/topics/eval-intro/)
- [Linux Foundation Valkey Announcement](https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community)
- [Valkey Persistence](https://valkey.io/topics/persistence/)
