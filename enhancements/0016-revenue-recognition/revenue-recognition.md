# METR-0016: Revenue Recognition Pipeline

## Metadata
- **METR:** 0016
- **Title:** Revenue Recognition Pipeline
- **Status:** Draft
- **Authors:** Meteridian Contributors
- **Created:** 2026-06-25
- **Related:** METR-0003 (Catalog), METR-0004 (Credit/Token Billing), METR-0007 (Compliance), METR-0009 (E-Invoicing)

## 1. Summary

Meteridian implements a native revenue recognition engine compliant with ASC 606 (US GAAP) and IFRS 15 (international). The engine tracks performance obligations from contract inception through satisfaction, calculates Standalone Selling Price (SSP) allocation for multi-element arrangements, manages deferred revenue schedules, and produces dual-standard journal entries suitable for ERP integration.

This METR transforms Module #24 (Revenue Recognition) from "planned" to "full" — filling the single largest enterprise gap identified in competitive analysis. Zuora built a $100M+ product line (Zuora Revenue) around this capability; Meteridian delivers it as an open-source, on-premises-deployable module.

## 2. Motivation

### 2.1 Business Need
- Public cloud providers, SaaS companies, and telcos with usage-based pricing must recognize revenue per ASC 606's five-step model
- Multi-element arrangements (platform + support + professional services) require SSP allocation
- Committed-spend contracts with variable consideration need constraint estimates
- Auditors require traceable journal entries linked to source transactions

### 2.2 Competitive Gap
- **Zuora Revenue:** Full ASC 606/IFRS 15 with waterfall schedules — proprietary, $150K+/year
- **Monetize360:** Integrated rev rec — closed-source, telco-focused
- **OpenMeter, Lago, Metronome, Orb:** None offer revenue recognition
- **Meteridian (current):** Module #24 is "planned" with no detailed design

### 2.3 Design Principles
- **Event-sourced:** Every rev rec decision is an immutable event, auditable forever
- **Dual-standard:** ASC 606 and IFRS 15 simultaneously (differences handled via policy configuration)
- **Real-time waterfall:** Deferred revenue updates on every billing event, not just month-end
- **ERP-agnostic:** Journal entries exported as standardized payloads (SAP, Oracle, NetSuite, Xero)

## 3. ASC 606 Five-Step Model Implementation

### 3.1 Step 1: Identify the Contract

```go
type RevRecContract struct {
    ID                uuid.UUID              `json:"id"`
    SourceContractID  uuid.UUID              `json:"source_contract_id"`
    CustomerID        uuid.UUID              `json:"customer_id"`
    InceptionDate     time.Time              `json:"inception_date"`
    TermEndDate       time.Time              `json:"term_end_date"`
    TotalConsideration decimal.Decimal       `json:"total_consideration"`
    Currency          string                 `json:"currency"`
    Status            ContractStatus         `json:"status"`
    Modifications     []ContractModification `json:"modifications,omitempty"`
    CreatedAt         time.Time              `json:"created_at"`
}

type ContractStatus string

const (
    ContractActive     ContractStatus = "active"
    ContractModified   ContractStatus = "modified"
    ContractCompleted  ContractStatus = "completed"
    ContractImpaired   ContractStatus = "impaired"
)
```

A RevRecContract is created automatically when a CommittedSpendContract (METR-0004) or subscription becomes active. Combination rules (ASC 606-10-25-9):
- Contracts entered at or near the same time with the same customer
- Negotiated as a package with a single commercial objective
- Consideration in one depends on the other

### 3.2 Step 2: Identify Performance Obligations

```go
type PerformanceObligation struct {
    ID                uuid.UUID       `json:"id"`
    ContractID        uuid.UUID       `json:"contract_id"`
    Description       string          `json:"description"`
    Type              POType          `json:"type"`
    SatisfactionPattern SatisfactionPattern `json:"satisfaction_pattern"`
    SSP               decimal.Decimal `json:"ssp"`
    AllocatedPrice    decimal.Decimal `json:"allocated_price"`
    Progress          decimal.Decimal `json:"progress"`
    Status            POStatus        `json:"status"`
}

type POType string

const (
    PODistinct        POType = "distinct"
    POSeriesDistinct  POType = "series_of_distinct"
    POCombined        POType = "combined"
)

type SatisfactionPattern string

const (
    OverTime  SatisfactionPattern = "over_time"
    PointInTime SatisfactionPattern = "point_in_time"
)
```

Distinct assessment (ASC 606-10-25-19):
- Customer can benefit from the good/service on its own or with readily available resources
- Entity's promise to transfer is separately identifiable from other promises

Series guidance (ASC 606-10-25-14b): Usage-based services where each time increment is a distinct service satisfied over time → recognized as billed (variable consideration allocation exception).

### 3.3 Step 3: Determine Transaction Price

```go
type TransactionPrice struct {
    ContractID            uuid.UUID       `json:"contract_id"`
    FixedConsideration    decimal.Decimal `json:"fixed_consideration"`
    VariableConsideration decimal.Decimal `json:"variable_consideration"`
    VariableConstraint    decimal.Decimal `json:"variable_constraint"`
    SignificantFinancing   decimal.Decimal `json:"significant_financing"`
    NonCashConsideration  decimal.Decimal `json:"non_cash_consideration"`
    PayableToCustomer     decimal.Decimal `json:"payable_to_customer"`
    TotalPrice            decimal.Decimal `json:"total_price"`
}
```

Variable consideration estimation methods:
- **Expected value:** Probability-weighted amounts (for large portfolios)
- **Most likely amount:** Single most likely outcome (for binary outcomes)

Constraint (ASC 606-10-32-11): Include variable consideration only to the extent it is "highly probable" (IFRS) or "probable" (GAAP) that a significant reversal will not occur.

For usage-based contracts (common in Meteridian):
- Usage fees allocated entirely to the period of usage (allocation exception per ASC 606-10-32-40)
- Minimum commitments treated as fixed consideration
- Overage above commitment = variable consideration recognized as invoiced

### 3.4 Step 4: Allocate Transaction Price

```go
type SSPAllocation struct {
    ContractID       uuid.UUID                `json:"contract_id"`
    Method           SSPMethod                `json:"method"`
    Allocations      []ObligationAllocation   `json:"allocations"`
    ResidualApplied  bool                     `json:"residual_applied"`
}

type SSPMethod string

const (
    SSPObservable       SSPMethod = "observable"
    SSPAdjustedMarket   SSPMethod = "adjusted_market"
    SSPExpectedCostPlus SSPMethod = "expected_cost_plus"
    SSPResidual         SSPMethod = "residual"
)

type ObligationAllocation struct {
    ObligationID   uuid.UUID       `json:"obligation_id"`
    SSP            decimal.Decimal `json:"ssp"`
    AllocatedPrice decimal.Decimal `json:"allocated_price"`
    Percentage     decimal.Decimal `json:"percentage"`
}
```

Allocation algorithm:
1. Determine SSP for each obligation (observable price preferred)
2. If total SSP != transaction price, allocate proportionally: `allocated = (ssp_i / sum_ssp) * transaction_price`
3. Discount allocation: proportional unless observable evidence that discount relates to specific obligations
4. Residual approach: permitted only when SSP is highly variable or uncertain

### 3.5 Step 5: Recognize Revenue

```go
type RecognitionEvent struct {
    ID              uuid.UUID       `json:"id"`
    ObligationID    uuid.UUID       `json:"obligation_id"`
    ContractID      uuid.UUID       `json:"contract_id"`
    EventType       RecEventType    `json:"event_type"`
    Amount          decimal.Decimal `json:"amount"`
    Currency        string          `json:"currency"`
    RecognitionDate time.Time       `json:"recognition_date"`
    PeriodStart     time.Time       `json:"period_start"`
    PeriodEnd       time.Time       `json:"period_end"`
    Standard        AccountingStd   `json:"standard"`
    JournalEntryID  uuid.UUID       `json:"journal_entry_id"`
}

type AccountingStd string

const (
    StdASC606  AccountingStd = "ASC_606"
    StdIFRS15  AccountingStd = "IFRS_15"
)
```

Recognition triggers:
- **Over time (time-based):** Straight-line over service period (e.g., annual support contract)
- **Over time (output-based):** Units delivered / total units promised
- **Over time (input-based):** Costs incurred / total expected costs
- **Point in time:** Transfer of control indicators met (ASC 606-10-25-30)
- **Usage-based (series):** As usage occurs, per allocation exception

## 4. Contract Modifications

### 4.1 Modification Types

| Scenario | Treatment | ASC 606 Reference |
|----------|-----------|-------------------|
| New distinct goods/services at SSP | Separate contract | 606-10-25-12 |
| New distinct goods/services NOT at SSP | Prospective (terminate + new) | 606-10-25-13a |
| Not distinct from original | Cumulative catch-up | 606-10-25-13b |

```go
type ContractModification struct {
    ID              uuid.UUID          `json:"id"`
    ContractID      uuid.UUID          `json:"contract_id"`
    EffectiveDate   time.Time          `json:"effective_date"`
    ModificationType ModificationType  `json:"modification_type"`
    NewObligations  []PerformanceObligation `json:"new_obligations,omitempty"`
    PriceAdjustment decimal.Decimal    `json:"price_adjustment"`
    CatchUpAmount   decimal.Decimal    `json:"catch_up_amount,omitempty"`
}

type ModificationType string

const (
    ModSeparateContract  ModificationType = "separate_contract"
    ModProspective       ModificationType = "prospective"
    ModCumulativeCatchUp ModificationType = "cumulative_catch_up"
)
```

### 4.2 Modification Workflow (Fluxo)

```yaml
workflow: contract_modification_assessment
triggers:
  - event: contract.amended
steps:
  - id: assess_distinct
    action: evaluate_distinctness
    inputs:
      new_services: "{{ event.new_services }}"
      existing_obligations: "{{ contract.obligations }}"
  - id: determine_treatment
    action: gorules_decision
    table: rev_rec_modification_treatment
    inputs:
      is_distinct: "{{ steps.assess_distinct.result }}"
      at_ssp: "{{ event.price_at_ssp }}"
  - id: apply_modification
    action: apply_rev_rec_modification
    inputs:
      treatment: "{{ steps.determine_treatment.result }}"
      contract_id: "{{ event.contract_id }}"
```

## 5. Deferred Revenue Waterfall

### 5.1 Schedule Model

```go
type DeferredRevenueSchedule struct {
    ID              uuid.UUID              `json:"id"`
    ObligationID    uuid.UUID              `json:"obligation_id"`
    ContractID      uuid.UUID              `json:"contract_id"`
    TotalDeferred   decimal.Decimal        `json:"total_deferred"`
    TotalRecognized decimal.Decimal        `json:"total_recognized"`
    Balance         decimal.Decimal        `json:"balance"`
    Periods         []WaterfallPeriod      `json:"periods"`
    Standard        AccountingStd          `json:"standard"`
}

type WaterfallPeriod struct {
    PeriodStart     time.Time       `json:"period_start"`
    PeriodEnd       time.Time       `json:"period_end"`
    OpeningBalance  decimal.Decimal `json:"opening_balance"`
    Additions       decimal.Decimal `json:"additions"`
    Recognized      decimal.Decimal `json:"recognized"`
    Adjustments     decimal.Decimal `json:"adjustments"`
    ClosingBalance  decimal.Decimal `json:"closing_balance"`
}
```

### 5.2 Real-Time Updates

Unlike traditional month-end batch rev rec, Meteridian updates the waterfall on every billing event:
- Invoice finalized → additions to deferred revenue
- Usage metered (series PO) → immediate recognition
- Payment received (point-in-time PO with payment condition) → recognition trigger
- Credit note issued → reversal against recognized or deferred as appropriate

### 5.3 Breakage Recognition

For prepaid/credit contracts (METR-0004 integration):
- **Proportional method:** Recognize breakage proportionally as credits are consumed
- **Remote method:** Recognize only when likelihood of exercise becomes remote
- Configuration per contract or product category via GoRules policy

## 6. Journal Entry Generation

### 6.1 Entry Templates

```go
type JournalEntry struct {
    ID              uuid.UUID       `json:"id"`
    EventID         uuid.UUID       `json:"event_id"`
    EntryDate       time.Time       `json:"entry_date"`
    PostingPeriod   string          `json:"posting_period"`
    Standard        AccountingStd   `json:"standard"`
    Lines           []JournalLine   `json:"lines"`
    Status          JournalStatus   `json:"status"`
    ERPExportID     *string         `json:"erp_export_id,omitempty"`
}

type JournalLine struct {
    AccountCode     string          `json:"account_code"`
    AccountName     string          `json:"account_name"`
    Debit           decimal.Decimal `json:"debit"`
    Credit          decimal.Decimal `json:"credit"`
    Dimensions      map[string]string `json:"dimensions,omitempty"`
}
```

Standard entries:

| Event | Debit | Credit |
|-------|-------|--------|
| Invoice raised | Accounts Receivable | Deferred Revenue |
| Revenue recognized (over time) | Deferred Revenue | Revenue |
| Revenue recognized (point-in-time) | Deferred Revenue | Revenue |
| Credit note | Revenue / Deferred Revenue | Accounts Receivable |
| Breakage (proportional) | Deferred Revenue | Breakage Revenue |
| Contract asset (unbilled) | Contract Asset | Revenue |

### 6.2 Dual-Standard Differences

| Topic | ASC 606 | IFRS 15 | Meteridian Handling |
|-------|---------|---------|---------------------|
| Constraint threshold | "Probable" (>75%) | "Highly probable" (>85%) | Configurable probability threshold per standard |
| Onerous contracts | No specific guidance | IAS 37 applies | Flag for IFRS; separate provision entry |
| Interim disclosures | ASC 270 quarterly | IAS 34 interim | Generate both period-end reports |
| License revenue | Functional vs symbolic | Similar but different wording | Policy flag per product type |

### 6.3 ERP Export

Supported export formats:
- **SAP S/4HANA:** IDoc BKPFF (FI posting) or Journal Entry API
- **Oracle Financials:** FBDI CSV for GL Journal Import
- **NetSuite:** SuiteQL Journal Entry or CSV
- **Xero:** Manual Journals API (OAuth2)
- **Generic:** CSV/JSON with configurable account mapping

```
POST /api/v1/rev-rec/journal-entries/export
{
  "period": "2026-06",
  "standard": "ASC_606",
  "format": "sap_idoc",
  "account_mapping_id": "map-001"
}
```

## 7. Multi-Entity Consolidation

### 7.1 Intercompany Transfers

For organizations with multiple legal entities:
- Revenue recognized in the entity that holds the contract
- Transfer pricing entries generated for shared-service cost allocation
- Intercompany eliminations flagged for consolidation

### 7.2 Multi-Currency

- Transaction currency: Currency of the contract
- Functional currency: Entity's operating currency
- Reporting currency: Group consolidation currency
- FX rates applied at recognition date (not inception date) per ASC 830

```go
type CurrencyTranslation struct {
    FromCurrency    string          `json:"from_currency"`
    ToCurrency      string          `json:"to_currency"`
    RateDate        time.Time       `json:"rate_date"`
    Rate            decimal.Decimal `json:"rate"`
    Source          string          `json:"source"`
}
```

## 8. Reporting and Disclosures

### 8.1 Required Disclosures (ASC 606-10-50)

| Disclosure | Implementation |
|-----------|----------------|
| Disaggregation of revenue | By product, geography, timing (over-time vs point-in-time) |
| Contract balances | Receivables, contract assets, contract liabilities (deferred) |
| Performance obligations | Remaining POs and expected timing of satisfaction |
| Significant judgments | SSP methods, variable consideration constraints, modification treatment |
| Transaction price allocated to remaining POs | Aggregate amount and expected recognition timing |

### 8.2 API Endpoints

```
GET    /api/v1/rev-rec/contracts                      # List rev rec contracts
GET    /api/v1/rev-rec/contracts/{id}                 # Contract detail with obligations
GET    /api/v1/rev-rec/contracts/{id}/waterfall       # Deferred revenue waterfall
GET    /api/v1/rev-rec/obligations                    # All performance obligations
POST   /api/v1/rev-rec/recognize                     # Trigger recognition for period
GET    /api/v1/rev-rec/journal-entries                # Journal entries for period
GET    /api/v1/rev-rec/reports/disaggregation         # Revenue disaggregation
GET    /api/v1/rev-rec/reports/remaining-po           # Remaining performance obligations
GET    /api/v1/rev-rec/reports/waterfall-summary      # Consolidated waterfall
POST   /api/v1/rev-rec/simulation                    # What-if modification simulation
```

## 9. Integration with Existing METRs

| METR | Integration Point |
|------|-------------------|
| METR-0003 (Catalog) | Product catalog provides SSP references and obligation templates |
| METR-0004 (Credits) | Prepaid drawdown triggers proportional breakage recognition |
| METR-0007 (Compliance) | Retention policies, audit trail requirements |
| METR-0009 (E-Invoicing) | Invoice finalization triggers deferred revenue additions |
| METR-0015 (Orders) | Order activation creates rev rec contract |

## 10. Event-Sourced Architecture

All rev rec state changes are stored as immutable events in an append-only log:

```go
type RevRecEvent struct {
    ID          uuid.UUID       `json:"id"`
    StreamID    uuid.UUID       `json:"stream_id"`
    Version     int64           `json:"version"`
    Type        string          `json:"type"`
    Payload     json.RawMessage `json:"payload"`
    OccurredAt  time.Time       `json:"occurred_at"`
    CausationID uuid.UUID       `json:"causation_id"`
}
```

Event types: `contract.created`, `obligation.identified`, `ssp.allocated`, `revenue.recognized`, `modification.applied`, `journal.posted`, `period.closed`

This enables:
- Full audit trail for SOX/SOC2 compliance
- Point-in-time reconstruction (what did the waterfall look like on any past date?)
- Correction without deletion (reversal events, not updates)
- Parallel dual-standard processing from the same event stream

## 11. Non-Goals

- **Tax filing:** Tax calculation and filing are handled by METR-0009 (§15)
- **Cash collection:** Payment receipt tracking is METR-0009 (§17.4)
- **Cost accounting:** COGS and margin analysis are METR-0005 (Unit Economics)
- **Budgeting:** Revenue forecasting is handled by the forecasting module
- **Full ERP:** Meteridian generates journal entries but does not replace a general ledger

## 12. Phased Delivery

| Phase | Scope | Target |
|-------|-------|--------|
| Phase 1 | Core five-step model, single-entity, single-currency, straight-line recognition | Q3 2026 |
| Phase 2 | Multi-element SSP allocation, contract modifications, breakage | Q4 2026 |
| Phase 3 | Multi-entity, multi-currency, ERP exports | Q1 2027 |
| Phase 4 | Dual-standard (IFRS 15), consolidation reporting | Q2 2027 |
