# ADR-0021: Unlimited Custom Dimensions via CloudEvents Extension Attributes

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq, @jordigilh
- **Related:** METR-0013 (Unit Economics), METR-0002 (Platform Extensibility), ADR-0004 (CloudEvents)

## Context and Problem Statement

Meteridian needs to support rich, multi-dimensional cost attribution across
infrastructure metering events, business telemetry, and unit cost calculations.
Users must be able to slice costs by customer, feature, team, environment,
cost center, project, and any other business-relevant dimension — without
being constrained by a fixed set of supported tags.

Traditional cost management platforms impose dimension limits:

| Platform | Approach | Practical Limit |
|----------|----------|-----------------|
| Koku (Red Hat Cost Management) | Tag keys must be enabled per provider via API | ~40 enabled tags per provider type |
| CloudZero | Cost groups defined in UI | ~20 allocation dimensions |
| Kubecost | Kubernetes labels and annotations | Limited to K8s label vocabulary |
| Apptio | Business mapping rules with fixed taxonomy | Fixed by implementation |
| Vantage | Virtual tags with rule-based derivation | ~30 per workspace |

These limits stem from implementation choices: fixed database columns for each
dimension, tag-enablement configuration, or finite UI-driven taxonomy. They
force users to choose which dimensions matter *before* they need them, and make
retroactive addition expensive.

Meteridian's CloudEvents-based architecture (ADR-0004) offers a natural
alternative: **CloudEvents extension attributes** provide an unlimited,
schema-free mechanism for attaching custom metadata to events. The question
is whether to adopt this mechanism for cost attribution dimensions or to
implement a more structured tag system.

## Decision Drivers

- **User flexibility** — Users should be able to add new dimensions at any time
  without configuration changes, schema migrations, or redeployment.
- **Query performance** — Dimensions must be efficiently queryable for GROUP BY,
  filtering, and aggregation in SQL-based metrics and unit cost calculations.
- **Interoperability** — The dimension mechanism should work naturally with
  CloudEvents-producing systems (Kubernetes event bridges, OpenTelemetry, custom
  applications).
- **Backward compatibility** — Existing events without custom dimensions must
  continue to work.
- **Governance** — Operators should have optional tools (validation, cardinality
  warnings, display names) without mandatory registration.

## Considered Options

### Option 1: CloudEvents Extension Attributes (Schemaless, JSONB-Stored)

Use CloudEvents extension attributes with a `mtrn` prefix for custom
dimensions. Store them in a JSONB column on the event table. Index with
PostgreSQL GIN indexes for efficient querying.

### Option 2: Fixed Tag Table with Enablement (Koku Pattern)

Create a dedicated `tags` table with foreign-key relationships to events.
Require operators to "enable" tag keys before they become queryable. This is
the approach used by Koku and most cloud cost management tools.

### Option 3: EAV (Entity-Attribute-Value) Pattern

Store dimensions in a separate `event_dimensions` table with (event_id, key,
value) rows. Index on key-value pairs.

## Decision Outcome

**Chosen option: Option 1 — CloudEvents Extension Attributes with JSONB
storage and GIN indexing.**

### Rationale

1. **Natural fit with CloudEvents.** The CloudEvents specification explicitly
   defines extension attributes for domain-specific metadata. Using them for
   cost dimensions is idiomatic — any CloudEvents producer can add Meteridian
   dimensions without a Meteridian-specific SDK.

2. **Zero-configuration dimension addition.** Users add a dimension by
   including a `mtrn`-prefixed extension attribute on events. No enablement
   step, no migration, no configuration file change. The dimension is
   immediately available for queries.

3. **JSONB with GIN indexing.** PostgreSQL's JSONB type with GIN indexes
   provides efficient containment queries (`@>`) and key-existence checks
   (`?`). For cost management workloads with typical dimension cardinality
   (< 10,000 values per dimension), GIN performance is comparable to dedicated
   B-tree indexes on fixed columns.

4. **Optional governance.** The dimension registry (METR-0013 §7.3) provides
   opt-in governance: display names, type validation, cardinality hints.
   Unregistered dimensions work without registration — governance is additive,
   not gatekeeping.

5. **Simpler data model.** Events are self-contained — all dimensions are
   embedded in the event record. No joins to separate tag tables. This
   simplifies SQL-based metrics, unit cost calculations, and cross-dimension
   aggregation queries.

### Convention: `mtrn` Prefix

Custom dimension attributes use the `mtrn` prefix to distinguish them from
standard CloudEvents attributes and other extensions:

```json
{
  "specversion": "1.0",
  "type": "compute.cpu.usage",
  "source": "cluster-01/ns-a",
  "mtrncustomer": "acme-corp",
  "mtrnfeature": "image-processing",
  "mtrnteam": "platform",
  "mtrnenvironment": "production",
  "mtrncostcenter": "CC-4521",
  "data": { "cpu_core_hours": 24.5 }
}
```

The `mtrn` prefix:
- Avoids collisions with standard CloudEvents attributes (`source`, `type`, etc.)
- Avoids collisions with other extensions (`meteridiancloud`, `meteridianregion`)
- Is short enough for practical use (4 characters)
- Follows the CloudEvents naming convention (lowercase, no separators)

### Storage Schema

Extension attributes are stored in a JSONB column on the event table:

```sql
ALTER TABLE events.metering_events
    ADD COLUMN custom_dimensions JSONB DEFAULT '{}';

CREATE INDEX idx_events_custom_dimensions
    ON events.metering_events USING GIN (custom_dimensions);
```

The ingestion pipeline extracts `mtrn`-prefixed attributes from the
CloudEvent envelope and stores them in `custom_dimensions`.

### Query Patterns

```sql
-- Filter by custom dimension
SELECT * FROM events.metering_events
WHERE custom_dimensions @> '{"customer": "acme-corp"}';

-- GROUP BY custom dimension
SELECT
    custom_dimensions->>'team' AS team,
    SUM(rated_amount_usd) AS total_spend
FROM events.rated_events
GROUP BY custom_dimensions->>'team';

-- Multi-dimension aggregation
SELECT
    custom_dimensions->>'customer' AS customer,
    custom_dimensions->>'feature' AS feature,
    SUM(rated_amount_usd) AS spend
FROM events.rated_events
WHERE custom_dimensions ? 'customer'
  AND custom_dimensions ? 'feature'
GROUP BY 1, 2;
```

## Trade-offs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Flexibility | Unlimited dimensions, zero configuration | Potential dimension sprawl without governance |
| Performance | GIN index efficient for typical cardinality | Slower than B-tree for very high cardinality (> 100K values) |
| Simplicity | No joins, self-contained events | Larger event payloads (embedded vs normalized) |
| Interoperability | Standard CloudEvents mechanism | Requires `mtrn` prefix convention awareness |
| Governance | Optional registry for display names and validation | No mandatory registration may lead to inconsistent naming |

### Performance Mitigation

For known high-cardinality dimensions that appear in hot-path queries, the
dimension registry can trigger creation of a dedicated expression index:

```sql
CREATE INDEX idx_events_dim_customer
    ON events.metering_events ((custom_dimensions->>'customer'));
```

This provides B-tree performance for specific dimensions while maintaining
GIN flexibility for ad-hoc dimensions.

## Consequences

### Positive

- Users can add cost attribution dimensions at any time with zero platform
  changes.
- SQL-based metrics and unit cost definitions can reference any dimension
  without prior registration.
- CloudEvents producers (Kubernetes event bridges, OpenTelemetry exporters,
  custom applications) can natively emit Meteridian dimensions.
- The dimension registry provides optional governance without mandatory
  gatekeeping.

### Negative

- Without governance, dimension naming may become inconsistent across teams
  (e.g., `customer` vs `customer_id` vs `client`).
- JSONB storage uses more space than normalized tag tables for high-repetition
  dimensions.
- Query planner may not optimize JSONB key extraction as well as fixed columns
  for complex multi-join queries.

### Mitigations

- The dimension registry provides validation rules and canonical names for
  common dimensions. Operators can enforce naming conventions via the registry
  without blocking unregistered dimensions.
- For high-volume, high-repetition dimensions, dedicated expression indexes
  provide equivalent performance to fixed columns.
- Documentation and examples establish naming conventions (`customer`,
  `feature`, `team`, `environment`, `cost_center`) that serve as community
  standards.

## Links

- [METR-0013: Unit Economics and Business Telemetry](../../enhancements/0013-unit-economics/unit-economics.md)
- [METR-0002: Platform Extensibility](../../enhancements/0002-extensibility/extensibility.md)
- [ADR-0004: CloudEvents as Canonical Event Format](0004-cloudevents-event-format.md)
- [CloudEvents Extension Attributes](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md#extension-context-attributes)
