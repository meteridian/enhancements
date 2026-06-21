# ADR-0003: GoRules ZEN Engine for Rating Rules

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian's rating engine must evaluate complex pricing rules to transform raw
metering events into rated (priced) events. These rules encompass tiered rates
(different prices at different usage levels), time-of-day pricing (peak vs
off-peak), volume discounts, committed spend drawdown, bundle allocation, tag-based
rates, and custom formulas. The pricing model varies by customer, product, and
contract — a single Meteridian deployment may simultaneously evaluate hundreds of
different rate cards across different tenants.

Rules change frequently. Customers update price lists when contracts renew, new
products are added to the catalog, and promotional pricing is applied for specific
time windows. These changes must be deployable without restarting the rating engine
and ideally without developer involvement. A visual interface for authoring and
testing pricing rules is essential — billing analysts and product managers need to
define, validate, and audit rules without writing code. The ability to test a rule
change against historical data before activating it is critical for preventing
billing errors.

The engine processes millions of events per hour at peak load. Rule evaluation must
be fast — sub-millisecond per event — to avoid becoming the bottleneck in the rating
pipeline. The rule engine is embedded in the rating worker process, so it must be
callable as a library (not a separate service) to avoid network round-trip overhead.
The engine must also support deterministic evaluation — given the same input and the
same rule version, the output must always be identical, which is a regulatory
requirement for billing auditability.

## Decision

We will use the GoRules ZEN Engine as the embedded rule evaluation engine for
Meteridian's rating pipeline. Pricing rules will be authored as JSON Decision
Models (JDM) using GoRules' visual decision table editor, stored in the database
alongside cost model metadata, and loaded into the ZEN engine at runtime. The
engine evaluates rules in-process with sub-millisecond latency per event.

The ZEN engine provides a natural model for pricing logic. Decision tables map input
conditions (resource type, usage tier, time of day, tags) to output values (unit
price, discount percentage, cost type). Each row in a decision table is a pricing
rule with clear conditions and outputs, evaluated top-to-bottom with first-match
semantics. Expression nodes handle calculations that don't fit neatly into tables —
pro-rata adjustments, currency conversion, rounding to billing precision. Function
graphs compose multiple decision nodes into complex pricing workflows: evaluate
base rate → apply volume discount → check committed spend → apply drawdown →
compute final rated cost.

JDM files are JSON documents that can be version-controlled, diffed, and audited.
Each cost model version in Meteridian is an immutable JDM snapshot, enabling
historical rate reconstruction for billing disputes and regulatory audits. When a
customer contests a charge, the exact rule version that was active at the time of
rating can be retrieved and re-evaluated against the original event. The MIT license
permits unrestricted use, modification, and distribution.

## Consequences

### Positive

- JDM visual editor enables non-developer rule authoring — billing analysts can
  create and modify pricing rules through decision tables without writing code.
  The editor provides immediate visual feedback, syntax validation, and test
  execution against sample data.
- Sub-millisecond evaluation per event (Rust core with Go bindings via CGo) — rule
  evaluation does not become a bottleneck in the rating pipeline. Benchmarks show
  <100μs for typical decision tables with 50-100 rows.
- Native decision table model maps naturally to pricing concepts: rows are pricing
  tiers, columns are conditions and outputs, evaluation order is top-to-bottom with
  first-match semantics. This is the same mental model billing analysts already use
  in spreadsheets.
- Expression language supports arithmetic, string operations, date functions, and
  conditional logic for complex pricing formulas. Decimal arithmetic is supported
  natively, avoiding floating-point rounding errors in billing calculations.
- Function graphs compose multiple decision nodes into multi-step pricing workflows
  with clear data flow. Each node's inputs and outputs are explicitly connected,
  making complex pricing logic visual and debuggable.
- JSON-based rule format is versionable in Git, auditable, and diffable — each cost
  model version is an immutable snapshot. This satisfies regulatory requirements
  for billing auditability.
- MIT license permits unrestricted use, modification, and distribution.
- Embedded in-process execution — no network round trips, no separate service to
  deploy and operate. The ZEN engine is loaded as a Go library via CGo bindings.
- Deterministic evaluation — given the same input and rule version, the output is
  always identical. This is essential for billing reproducibility and dispute
  resolution.

### Negative

- Dependency on the GoRules project, which has a relatively small maintainer team
  compared to larger CNCF projects. However, the MIT license means the project can
  be forked and maintained independently if necessary.
- JDM is a GoRules-specific format without an industry standard equivalent (there
  is no widely adopted standard for decision model interchange). Rules authored in
  JDM are not portable to other rule engines without conversion tooling.
- Expression language has a learning curve for complex formulas, particularly around
  type coercion, null handling, and decimal precision. Documentation and examples
  must be provided for billing analysts.
- CGo bridge between Rust core and Go bindings adds build complexity — the Rust
  toolchain is required for development builds, and cross-compilation requires
  pre-built native libraries for each target architecture (amd64, arm64).
- Large JDM files (hundreds of rules across multiple decision tables) can be
  difficult to review in pull requests. Tooling for structured JDM diffs may need
  to be developed.

### Neutral

- Decision tables are evaluated in definition order (top-to-bottom, first match) —
  rule authors must understand evaluation semantics to avoid incorrect pricing. The
  visual editor helps by showing evaluation flow, but training is still required.
- JDM files can be large for complex pricing models (hundreds of rules across
  multiple tables) but compress well and are efficiently parsed by the Rust core.
  Loading a typical JDM file takes <1ms.
- The visual editor is a separate web application (React-based) that can be
  deployed alongside Meteridian or used standalone during development. It
  communicates with the backend via a REST API for rule storage and retrieval.
- Rule versioning is a Meteridian feature built on top of JDM — the ZEN engine
  itself evaluates a single JDM document without version awareness.

## Alternatives Considered

### OPA (Open Policy Agent) / Rego

OPA is a mature, CNCF-graduated policy engine with strong adoption in the
Kubernetes ecosystem for authorization and admission control. Rego, OPA's query
language, is powerful and expressive, with support for complex logic, set
operations, and data aggregation.

However, OPA is designed for authorization policies (allow/deny decisions), not
business rules with numeric outputs. Rego's learning curve is steep — billing
analysts and product managers would need significant training to author pricing
rules. OPA lacks a native visual editor for decision tables; while third-party
tools like Styra DAS exist, they are designed for policy visualization, not pricing
rule authoring. More critically, Rego does not have native decimal arithmetic — all
numbers are IEEE 754 floating point, which introduces rounding errors in billing
calculations (e.g., `0.1 + 0.2 ≠ 0.3`). Rule evaluation latency is also higher
for complex decision tables (~2-5ms vs <1ms for ZEN), which would measurably impact
the rating pipeline under sustained load.

### Custom DSL

Building a custom domain-specific language for pricing rules would provide maximum
control over semantics, performance, and tooling. The language could be designed
specifically for billing concepts — tiers, discounts, bundles, commitments — with
first-class support for decimal arithmetic and billing periods.

However, the development effort is enormous: lexer, parser, type checker,
evaluator, visual editor, test framework, documentation, and debugging tools. This
represents 6-12 months of focused development to reach feature parity with GoRules,
during which the core billing functionality is delayed. Every billing platform that
has built a custom rule language (Zuora, Stripe, Chargebee) spent years developing
and maintaining it with dedicated teams. For a new project, this upfront investment
is not justifiable when a ready-made solution covers the vast majority of use cases.

### Embedded Lua / WASM Rules

Embedding Lua or WASM modules for rule evaluation would provide fast execution and
language flexibility. Lua in particular has extremely low overhead when embedded in
a Go process (via gopher-lua or similar), and WASM provides sandboxing with
near-native performance.

However, neither approach includes a visual editor, a decision table model, or an
audit trail. Business users cannot modify pricing rules without developer
assistance — every rate change requires writing code, testing it, and deploying it
through the standard software release process. This contradicts the requirement for
non-developer rule authoring. Debugging complex pricing logic in Lua is painful
without visual tools for tracing why a specific event received a specific price.
WASM adds compilation steps and makes debugging even more opaque, as stack traces
reference WASM instruction offsets rather than source code locations.

### Drools / jBPM

Drools is a mature business rule management system with a visual editor (Business
Central) and strong enterprise adoption, particularly in financial services and
insurance. It supports decision tables, DRL (Drools Rule Language), and DMN
(Decision Model and Notation) standards.

However, Drools is JVM-based with a heavy runtime footprint (~500MB+ heap for a
modest rule set). Embedding Drools in a Go process would require either a JNI
bridge (fragile, complex, crash-prone) or running Drools as a separate service
(network latency, operational overhead, deployment complexity). The Drools/jBPM
ecosystem (WildFly, Business Central, KIE Server) is designed for enterprise Java
deployments, not cloud-native Go microservices. Its RETE algorithm is designed for
complex event processing with many interdependent rules — over-engineered for the
decision table evaluation pattern that billing requires.

## References

- [GoRules](https://gorules.io/)
- [GoRules ZEN Engine GitHub](https://github.com/gorules/zen)
- [JDM Editor](https://github.com/gorules/jdm-editor)
- [GoRules Decision Model Documentation](https://gorules.io/docs/rules-engine/decision-model)
- [GoRules Go SDK](https://gorules.io/docs/rules-engine/engines/go)
- [GoRules Expression Language](https://gorules.io/docs/rules-engine/expression-language)
