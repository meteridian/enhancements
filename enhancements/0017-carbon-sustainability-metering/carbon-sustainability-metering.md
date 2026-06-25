# METR-0017: Carbon & Sustainability Metering

## Metadata
- METR: 0017
- Title: Carbon and Sustainability Metering
- Status: Draft
- Authors: Meteridian Contributors
- Created: 2026-06-25
- Related: METR-0005 (Chargeback/FinOps), METR-0010 (AI/ML Metering), METR-0012 (Multi-Cloud Metering)

## 1. Summary

Meteridian adds carbon emissions as a first-class billing and chargeback dimension. Every metered resource (compute, storage, network, GPU) is annotated with its carbon intensity in gCO2eq, enabling operators to:
- Bill customers with carbon surcharges or carbon-inclusive pricing
- Allocate Scope 2/3 emissions to tenants via chargeback
- Report per-customer/per-project emissions for CSRD, SEC Climate, ISO 14064
- Signal carbon-aware scheduling decisions to orchestrators

No billing platform currently offers native carbon metering. This creates a unique competitive moat at the intersection of FinOps and GreenOps.

## 2. Motivation

### 2.1 Regulatory Drivers
- EU CSRD (2024+): Large enterprises must report Scope 1/2/3 emissions including supply chain (cloud = Scope 3 for customers)
- SEC Climate Rules (2024+): Material climate risks and GHG emissions disclosure for US-listed
- ISO 14064-1: Quantification and reporting of GHG emissions
- EU Taxonomy Regulation: Classification of sustainable economic activities
- France BEGES: Mandatory carbon footprint for organizations above 500 employees

### 2.2 Market Gap
- Cloud providers publish aggregate carbon data (AWS Customer Carbon Footprint, Azure Emissions Impact Dashboard, GCP Carbon Footprint) but NOT at invoice-line-item granularity
- FinOps tools (CloudCarbon Footprint, Greenpixie) estimate but don't integrate into billing
- No billing platform offers carbon as a pricing dimension
- Operators running sovereign/on-prem clouds have zero visibility without custom tooling

### 2.3 Meteridian Advantage
- Already meters at infrastructure level (METR-0005 collectors: bare metal, VMs, K8s, network)
- Already has cost allocation engine with virtual tags (METR-0005 chargeback)
- GoRules engine can evaluate carbon intensity rules alongside pricing rules
- Redpanda Connect pipeline can enrich events with carbon data in real-time

## 3. Carbon Attribution Model

### 3.1 Data Model

```go
type CarbonIntensity struct {
    Region          string          `json:"region"`
    Provider        string          `json:"provider"`
    Timestamp       time.Time       `json:"timestamp"`
    GramsCO2PerKWh  decimal.Decimal `json:"grams_co2_per_kwh"`
    Source          CarbonSource    `json:"source"`
    EnergyMix       EnergyMix       `json:"energy_mix,omitempty"`
}

type EnergyMix struct {
    Solar       float64 `json:"solar"`
    Wind        float64 `json:"wind"`
    Nuclear     float64 `json:"nuclear"`
    Gas         float64 `json:"gas"`
    Coal        float64 `json:"coal"`
    Hydro       float64 `json:"hydro"`
    Other       float64 `json:"other"`
}

type CarbonSource string
const (
    SourceElectricityMaps CarbonSource = "electricity_maps"
    SourceWattTime        CarbonSource = "watttime"
    SourceOperatorDefined CarbonSource = "operator_defined"
    SourceCloudProvider   CarbonSource = "cloud_provider"
)

type ResourceCarbonFootprint struct {
    ResourceID      string          `json:"resource_id"`
    ResourceType    string          `json:"resource_type"`
    PeriodStart     time.Time       `json:"period_start"`
    PeriodEnd       time.Time       `json:"period_end"`
    EnergyKWh       decimal.Decimal `json:"energy_kwh"`
    CarbonGrams     decimal.Decimal `json:"carbon_grams_co2eq"`
    Scope           EmissionScope   `json:"scope"`
    Attribution     CarbonAttribution `json:"attribution"`
}

type EmissionScope string
const (
    Scope2Market   EmissionScope = "scope2_market"
    Scope2Location EmissionScope = "scope2_location"
    Scope3         EmissionScope = "scope3"
)

type CarbonAttribution struct {
    Method          string          `json:"method"`
    TenantID        string          `json:"tenant_id"`
    ProjectID       string          `json:"project_id"`
    CostCenterID    string          `json:"cost_center_id,omitempty"`
}
```

### 3.2 Emission Scopes

| Scope | For Cloud Operator | For End Customer |
|-------|-------------------|-----------------|
| Scope 1 | On-site generators (diesel backup) | N/A |
| Scope 2 (location) | Grid average carbon intensity at datacenter location | N/A |
| Scope 2 (market) | Contracted energy (PPA, RECs, green tariff) | N/A |
| Scope 3 (upstream) | Embodied carbon of hardware, cooling infrastructure | Cloud usage = customer's Scope 3 |

Meteridian reports **Scope 2 (both methods) + embodied Scope 3** for operator's own reporting, and provides the data as **customer's Scope 3** in tenant-facing reports.

### 3.3 Calculation Pipeline

```
Resource Metric (CPU-hours, GPU-hours, storage-TB-hours, network-GB)
    │
    ▼
Energy Model (TDP-based for compute, measured for storage/network)
    │
    ▼
PUE Factor (datacenter-specific, time-varying)
    │
    ▼
Carbon Intensity (region + time → gCO2/kWh from data source)
    │
    ▼
Embodied Carbon Amortization (hardware lifecycle allocation)
    │
    ▼
Total Carbon (operational + embodied) → gCO2eq per resource-unit
    │
    ▼
Tenant Attribution (proportional to usage, using METR-0005 allocation)
```

## 4. Energy Modeling

### 4.1 Compute Energy

For servers and VMs:
```
energy_kwh = (TDP_watts * utilization_fraction * hours) / 1000
```

For containers (K8s):
```
energy_kwh = (node_TDP * (container_cpu_request / node_allocatable_cpu) * hours) / 1000
```

For GPUs:
```
energy_kwh = (GPU_TDP * gpu_utilization * hours) / 1000
```

TDP values sourced from:
- Operator hardware inventory (BMC/IPMI data, METR-0005 collectors)
- Default TDP database (Intel ARK, AMD product specs, NVIDIA specs)
- Measured power (if PDU/BMC telemetry available via Redfish)

### 4.2 Storage Energy

```
energy_kwh = storage_tb * watts_per_tb * hours / 1000
```

Default values: HDD ~6W/TB, SSD ~2W/TB, NVMe ~3W/TB (configurable per storage class).

### 4.3 Network Energy

```
energy_kwh = data_transferred_gb * kwh_per_gb
```

Default: 0.06 kWh/GB (per Shift Project estimates, configurable).

### 4.4 PUE (Power Usage Effectiveness)

```
total_energy = IT_energy * PUE
```

PUE sources:
- Static per-datacenter configuration (e.g., PUE=1.2 for modern facility)
- Dynamic via BMS (Building Management System) API integration
- Time-varying (PUE higher in summer due to cooling load)

## 5. Carbon Intensity Data Sources

### 5.1 Electricity Maps Integration

```yaml
# Redpanda Connect carbon enrichment block
pipeline:
  processors:
    - http:
        url: "https://api.electricitymap.org/v3/carbon-intensity/latest?zone=${! this.region }"
        headers:
          auth-token: "${ELECTRICITY_MAPS_TOKEN}"
        result_map: |
          root.carbon_intensity_gco2_kwh = this.carbonIntensity
          root.energy_mix = this.fossilFuelPercentage
```

### 5.2 WattTime Integration

Alternative for US-focused deployments with marginal emissions data.

### 5.3 Operator-Defined (Air-Gapped)

For sovereign/air-gapped deployments:
```yaml
carbon_intensity:
  static_regions:
    - region: "eu-west-1"
      gco2_per_kwh: 250
      source: "operator_estimate"
    - region: "us-east-1"
      gco2_per_kwh: 380
      source: "epa_egrid_2024"
```

### 5.4 Cloud Provider Native Data

Ingest carbon data from:
- AWS: Customer Carbon Footprint Tool (CCFT) API
- Azure: Emissions Impact Dashboard API
- GCP: Carbon Footprint API

## 6. Embodied Carbon

### 6.1 Hardware Lifecycle Model

```go
type HardwareEmbodiedCarbon struct {
    HardwareType    string          `json:"hardware_type"`
    Manufacturer    string          `json:"manufacturer"`
    Model           string          `json:"model"`
    TotalEmbodied   decimal.Decimal `json:"total_embodied_kg_co2"`
    ExpectedLifeYrs int             `json:"expected_life_years"`
    AmortizedDaily  decimal.Decimal `json:"amortized_daily_g_co2"`
}
```

Amortization: `daily_embodied = total_embodied_kg * 1000 / (expected_life_years * 365)`

Allocated to tenants proportionally by resource consumption (same allocation method as METR-0005 chargeback).

### 6.2 Default Embodied Carbon Database

Based on manufacturer LCA (Life Cycle Assessment) reports:
- Dell PowerEdge R760: ~1,200 kgCO2e (manufacturing + transport)
- NVIDIA H100 SXM: ~150 kgCO2e (estimated from semiconductor fab data)
- Standard SSD 1TB: ~50 kgCO2e
- Network switch (Arista 7060X): ~300 kgCO2e

Operators can override with actual LCA data from their procurement.

## 7. Carbon-Aware Pricing

### 7.1 Carbon Surcharge Model

Operators can add carbon surcharges to rate plans:

```go
type CarbonPricingRule struct {
    ID              uuid.UUID       `json:"id"`
    RatePlanID      uuid.UUID       `json:"rate_plan_id"`
    PricingModel    CarbonPricing   `json:"pricing_model"`
    PricePerTonCO2  decimal.Decimal `json:"price_per_ton_co2,omitempty"`
    IncludedTons    decimal.Decimal `json:"included_tons,omitempty"`
}

type CarbonPricing string
const (
    CarbonPassThrough  CarbonPricing = "pass_through"
    CarbonInclusive    CarbonPricing = "inclusive"
    CarbonOffset       CarbonPricing = "offset_surcharge"
    CarbonBudget       CarbonPricing = "budget_enforcement"
)
```

Models:
- **Pass-through:** Carbon cost added as line item (e.g., EUR 80/ton CO2)
- **Inclusive:** Carbon cost baked into base pricing (transparent reporting only)
- **Offset surcharge:** Additional charge to fund carbon offset purchases
- **Budget enforcement:** Carbon budget per tenant with alerts/throttling at threshold (integrates with METR-0011 enforcement)

### 7.2 Carbon Budget Enforcement

Integration with METR-0011:
```yaml
enforcement_policy:
  trigger: carbon_budget_exceeded
  thresholds:
    - level: 80%
      action: notify
      signal: "carbon.budget.warning"
    - level: 100%
      action: throttle
      signal: "carbon.budget.exceeded"
    - level: 120%
      action: block_new_deployments
      signal: "carbon.budget.hard_limit"
```

## 8. Carbon-Aware Scheduling Signals

Meteridian can emit signals to orchestrators for carbon-aware workload placement:

```go
type CarbonSchedulingSignal struct {
    SignalType   string            `json:"signal_type"`
    Regions      []RegionCarbon    `json:"regions"`
    Recommendation string          `json:"recommendation"`
    ValidUntil   time.Time         `json:"valid_until"`
}

type RegionCarbon struct {
    Region          string          `json:"region"`
    CurrentIntensity decimal.Decimal `json:"current_intensity_gco2"`
    ForecastedLow   time.Time       `json:"forecasted_low"`
    Score           int             `json:"score"`
}
```

Signals published to NATS JetStream for consumption by:
- Kubernetes schedulers (carbon-aware pod placement)
- Batch job schedulers (shift deferrable workloads to low-carbon windows)
- VM placement engines

## 9. Reporting and Compliance

### 9.1 CSRD/ESRS E1 Report

Generate ESRS E1 (Climate Change) disclosures:
- Total GHG emissions (Scope 1, 2, 3) by category
- GHG intensity per net revenue
- Energy consumption and mix
- Targets and transition plans

### 9.2 Tenant Carbon Report

Per-tenant downloadable report including:
- Total emissions attributed (Scope 2 + 3)
- Breakdown by resource type
- Month-over-month trend
- Comparison to carbon budget
- Methodology notes (for auditor reference)

### 9.3 API Endpoints

```
GET  /api/v1/carbon/intensity/current?region=eu-west-1
GET  /api/v1/carbon/intensity/forecast?region=eu-west-1&hours=24
GET  /api/v1/carbon/footprint?tenant_id=X&period=2026-06
GET  /api/v1/carbon/footprint/breakdown?group_by=resource_type
GET  /api/v1/carbon/budget?tenant_id=X
POST /api/v1/carbon/budget                              # Set carbon budget
GET  /api/v1/carbon/reports/csrd?period=2026
GET  /api/v1/carbon/reports/tenant?tenant_id=X&period=2026-Q2
GET  /api/v1/carbon/scheduling/signals
```

## 10. Integration with Existing METRs

| METR | Integration |
|------|-------------|
| METR-0005 | Infrastructure collectors provide resource metrics; chargeback engine allocates carbon |
| METR-0010 | GPU energy model uses AI/ML metering data (utilization, MIG slices) |
| METR-0011 | Carbon budget enforcement via enforcement integration |
| METR-0012 | Cloud provider carbon data enriches multi-cloud metering |
| METR-0003 | Carbon surcharge as a pricing dimension in rate plans |

## 11. Non-Goals

- Carbon offset procurement (operators buy offsets externally; Meteridian tracks the surcharge)
- Scope 1 measurement (requires physical sensors, not software)
- Carbon credit trading or marketplace
- Full LCA of software supply chain (only infrastructure emissions)

## 12. Phased Delivery

| Phase | Scope | Target |
|-------|-------|--------|
| Phase 1 | Energy model for compute/storage/GPU, static carbon intensity, basic reporting | Q4 2026 |
| Phase 2 | Electricity Maps integration, PUE dynamic, embodied carbon, CSRD report | Q1 2027 |
| Phase 3 | Carbon-aware pricing models, budget enforcement, scheduling signals | Q2 2027 |
| Phase 4 | Cloud provider carbon data ingestion, tenant self-service carbon portal | Q3 2027 |
