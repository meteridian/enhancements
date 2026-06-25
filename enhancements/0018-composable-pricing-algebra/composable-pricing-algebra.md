# METR-0018: Composable Pricing Algebra

- **METR:** 0018
- **Title:** Composable Pricing Algebra
- **Status:** draft
- **Authors:** Meteridian Contributors
- **Created:** 2026-06-25
- **Last Updated:** 2026-06-25
- **Related:** METR-0003 (Catalog/Pricing), METR-0004 (Credits), METR-0008 (Rating Engine)

## Summary

Meteridian introduces a formal algebraic type system for pricing primitives.
Pricing models are composed from well-defined atomic operations with algebraic
laws (associativity, commutativity, distributivity for discounts), enabling
correctness proofs, automated conflict detection, and a developer SDK for
building arbitrarily complex pricing models without custom code.

This deepens Modules #2 (Pricing/Rate Cards), #3 (Pricing Simulation), and #12
(Rating Engine) from "full (basic)" to "full (defensible moat)" — no competitor
offers formally verified pricing composition.

## Motivation

### The Problem with Ad-Hoc Pricing

Current billing platforms (including Meteridian's existing rate engine) define
pricing models as configuration — JSON/YAML that describes tiers, volumes, flat
fees. Problems:

- No guarantee that combining two valid pricing rules produces a valid result
- Discount stacking order matters but is implicit (first-applied vs
  last-applied)
- Complex models (hybrid of tiered + time-of-day + volume + committed) require
  custom code
- No way to formally verify that a pricing change won't produce negative charges
- Simulation requires executing the full engine rather than algebraic reasoning

### Competitive Landscape

- **Orb:** "Composable pricing" marketing but no formal algebra — just flexible
  configuration
- **Metronome:** Type-safe pricing SDK (TypeScript) but no algebraic laws or
  composition proofs
- **Zuora:** Complex pricing matrix but rule conflicts silently resolved by
  priority ordering
- **Monetize360:** Telco-grade rating with CDR mediation but imperative rule
  evaluation

### Vision

A pricing model in Meteridian becomes a typed expression in a pricing algebra,
where:

- Every expression evaluates to a `Money` value (non-negative, currency-aware)
- Composition operators have defined algebraic properties
- The type system prevents invalid compositions at definition time
- Formal properties enable algebraic simplification and optimization

## Algebraic Foundations

### Pricing Primitives (Atoms)

```
Flat(amount, currency)              → Fixed charge
PerUnit(rate, unit, currency)       → Linear usage charge
Tiered(tiers[], unit)               → Graduated/volume tiers
StepFunction(steps[], dimension)    → Step-based (time, quantity, etc.)
Percentage(rate, base)              → Percentage of another expression
Minimum(amount, currency)           → Floor guarantee
Maximum(amount, currency)           → Cap/ceiling
Zero                                → Identity element
```

### Composition Operators

```
Add(a, b)         → a + b                (commutative, associative)
Multiply(a, k)    → a * k                (scalar multiplication)
Min(a, b)         → min(a, b)            (commutative, associative)
Max(a, b)         → max(a, b)            (commutative, associative)
Choose(cond, a, b) → if cond then a else b (conditional)
Discount(a, d)    → a * (1 - d)          (NOT commutative with Add)
Sequence(a, b)    → apply a, then apply b to result (associative, NOT commutative)
```

### Algebraic Laws

```
-- Additive identity
Add(p, Zero) = p

-- Additive commutativity
Add(a, b) = Add(b, a)

-- Additive associativity
Add(Add(a, b), c) = Add(a, Add(b, c))

-- Discount distributivity over Add
Discount(Add(a, b), d) = Add(Discount(a, d), Discount(b, d))

-- Discount composition
Discount(Discount(a, d1), d2) = Discount(a, d1 + d2 - d1*d2)

-- Min/Max absorption
Min(a, Maximum(cap)) = Min(a, cap)  -- cap dominates
Max(a, Minimum(floor)) = Max(a, floor)  -- floor dominates

-- Non-negativity invariant
forall p: eval(p) >= Zero  -- pricing expressions cannot produce negative values
```

### Type System

```go
type PricingType string
const (
    TypeMoney      PricingType = "money"
    TypeRate       PricingType = "rate"
    TypePercentage PricingType = "percentage"
    TypeCondition  PricingType = "condition"
    TypeDimension  PricingType = "dimension"
)

type PricingExpr struct {
    Kind       string          `json:"kind"`
    Type       PricingType     `json:"type"`
    Children   []PricingExpr   `json:"children,omitempty"`
    Params     json.RawMessage `json:"params,omitempty"`
    ResultType PricingType     `json:"result_type"`
}
```

Type checking rules:

- `Add(a, b)` requires `a.ResultType == Money && b.ResultType == Money`
- `Discount(a, d)` requires `a.ResultType == Money && d.ResultType == Percentage`
- `PerUnit(rate, unit)` requires `rate.Type == Rate` and produces `Money`
- `Choose(cond, a, b)` requires `cond.Type == Condition && a.ResultType == b.ResultType`

## Pricing Expression Language (PEL)

### Syntax

```
// Simple per-unit pricing
compute_cost = PerUnit($0.10, "cpu_hour")

// Tiered with volume discount
storage_cost = Tiered([
  { up_to: 100, rate: $0.05/GB },
  { up_to: 1000, rate: $0.03/GB },
  { above: 1000, rate: $0.01/GB }
], "storage_gb")

// Composed: compute + storage with 20% enterprise discount, minimum $100
total = Max(
  Discount(Add(compute_cost, storage_cost), 0.20),
  Minimum($100)
)

// Time-of-day modifier
peak_compute = Sequence(
  compute_cost,
  Choose(
    time_in_range("09:00-17:00", "UTC"),
    Multiply(_, 1.5),  // 50% peak surcharge
    _                   // off-peak: no change
  )
)
```

### Compilation to GoRules

PEL expressions compile to GoRules JDM decision tables for runtime evaluation:

1. Parse PEL → AST (Abstract Syntax Tree)
2. Type-check AST (reject type errors at definition time)
3. Verify algebraic invariants (non-negativity, no discount > 100%)
4. Optimize (algebraic simplification: collapse nested discounts, lift constants)
5. Emit GoRules decision table rows

### Conflict Detection

When composing pricing rules, the algebra detects conflicts:

- **Discount stacking conflict:** Two discounts on the same base that exceed
  100% combined
- **Floor/ceiling contradiction:** Minimum > Maximum on the same expression
- **Infinite recursion:** Circular references between expressions
- **Currency mismatch:** Adding Money in different currencies without conversion

Detected at definition time, not runtime.

## Developer SDK

### Go SDK

```go
import "github.com/meteridian/pricing-algebra"

plan := pricing.NewPlan("enterprise-gpu").
    Add(pricing.PerUnit(0.10, "cpu_hour", "USD")).
    Add(pricing.Tiered("gpu_hour", "USD",
        pricing.Tier{UpTo: 100, Rate: 2.50},
        pricing.Tier{UpTo: 1000, Rate: 2.00},
        pricing.Tier{Above: 1000, Rate: 1.50},
    )).
    Add(pricing.Flat(500, "USD").Monthly()).
    WithDiscount(pricing.VolumeDiscount(
        pricing.Threshold{Spend: 10000, Discount: 0.10},
        pricing.Threshold{Spend: 50000, Discount: 0.20},
    )).
    WithMinimum(pricing.Flat(1000, "USD").Monthly()).
    Build()

// Type-check and verify at build time
if err := plan.Verify(); err != nil {
    // e.g., "discount can exceed 100% at threshold boundary"
}

// Evaluate
result := plan.Evaluate(usage)
```

### TypeScript SDK

```typescript
import { plan, perUnit, tiered, flat, discount } from '@meteridian/pricing';

const gpuPlan = plan('enterprise-gpu')
  .add(perUnit(0.10, 'cpu_hour', 'USD'))
  .add(tiered('gpu_hour', 'USD', [
    { upTo: 100, rate: 2.50 },
    { upTo: 1000, rate: 2.00 },
    { above: 1000, rate: 1.50 },
  ]))
  .add(flat(500, 'USD').monthly())
  .withDiscount(discount.volume([
    { spend: 10_000, rate: 0.10 },
    { spend: 50_000, rate: 0.20 },
  ]))
  .withMinimum(flat(1000, 'USD').monthly())
  .verify(); // throws TypeError if invalid composition
```

### REST API

```
POST /api/v1/pricing/expressions/validate    # Type-check a PEL expression
POST /api/v1/pricing/expressions/evaluate    # Evaluate against sample usage
POST /api/v1/pricing/expressions/compile     # Compile PEL → GoRules table
GET  /api/v1/pricing/expressions/{id}/graph  # Visualize expression tree
POST /api/v1/pricing/expressions/diff        # Compare two expressions (semantic diff)
POST /api/v1/pricing/expressions/simulate    # Monte Carlo simulation with expression
```

## Simulation with Algebra

### Algebraic What-If

Because pricing expressions have defined algebraic properties, simulation can be
done algebraically rather than by re-executing the full metering + rating
pipeline:

```
// "What if we change discount from 20% to 25%?"
old = Discount(Add(compute, storage), 0.20)
new = Discount(Add(compute, storage), 0.25)
diff = old - new = Add(compute, storage) * (0.20 - 0.25) = Add(compute, storage) * (-0.05)
// Impact: 5% reduction, calculable without re-metering
```

### Sensitivity Analysis

Partial derivatives of pricing expressions with respect to usage dimensions:

```
d(total_cost) / d(cpu_hours) = rate for current tier
d(total_cost) / d(gpu_hours) = rate for current tier * (1 - discount)
```

Enables instant sensitivity charts in the UI without re-executing the pipeline.

## Migration from Existing Rate Engine

### Compatibility

Every existing rate plan (JSON/YAML in METR-0003 format) is expressible as a PEL
expression. The migration is:

1. Parse existing rate plan JSON
2. Generate equivalent PEL expression
3. Verify: `eval(PEL_expr, usage) == eval(old_engine, usage)` for all
   historical usage
4. Replace (optional — both engines can coexist during transition)

### Coexistence

The algebra engine and the existing GoRules rating engine are not mutually
exclusive:

- Simple plans: Continue using JSON rate plans (backward compatible)
- Complex plans: Author in PEL, compile to GoRules
- Hybrid: Some products use old engine, some use algebra

## Integration with Existing METRs

| METR | Integration |
|------|-------------|
| METR-0003 | Catalog stores PEL expressions as the canonical pricing definition |
| METR-0004 | Credit drawdown is a PEL primitive (Debit wallet, ceiling at balance) |
| METR-0008 | Rating engine evaluates compiled PEL expressions via GoRules |
| METR-0016 | SSP allocation uses algebraic decomposition of multi-element pricing |

## Non-Goals

- Replacing GoRules as the runtime engine (PEL compiles TO GoRules)
- Supporting arbitrary programming (PEL is intentionally not Turing-complete)
- Real-time pricing negotiation (quotes use the algebra but negotiation is
  METR-0004 section 4)
- Currency conversion (handled by a separate FX module)

## Phased Delivery

| Phase | Scope | Target |
|-------|-------|--------|
| Phase 1 | Core primitives, type system, non-negativity proof, Go SDK | Q1 2027 |
| Phase 2 | PEL syntax, compiler to GoRules, conflict detection | Q2 2027 |
| Phase 3 | TypeScript SDK, REST API, algebraic simulation | Q3 2027 |
| Phase 4 | Migration tool, sensitivity analysis, visual expression editor | Q4 2027 |
