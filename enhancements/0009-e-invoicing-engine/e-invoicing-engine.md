# METR-0009: Native E-Invoicing Engine

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0001 (Platform Architecture), METR-0002 (Extensibility), METR-0003 (Product Catalog)
- **Related:** METR-0004 (Credit and Token Billing), METR-0007 (Legal and Regulatory Compliance), METR-0008 (Compliance-as-Code), ADR-0016 (E-Invoice Canonical Model)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Motivation](#2-motivation)
3. [Competitive Landscape](#3-competitive-landscape)
4. [Architecture Overview](#4-architecture-overview)
5. [Canonical Invoice Model](#5-canonical-invoice-model)
6. [Format Generation Framework](#6-format-generation-framework)
7. [Digital Signature Framework](#7-digital-signature-framework)
8. [Tax Authority Integration](#8-tax-authority-integration)
9. [Validation Engine](#9-validation-engine)
10. [Sequential Invoice Numbering](#10-sequential-invoice-numbering)
11. [Multi-Currency and Exchange Rates](#11-multi-currency-and-exchange-rates)
12. [QR Code Generation](#12-qr-code-generation)
13. [Integration with Block Architecture](#13-integration-with-block-architecture)
14. [Integration with Rating Pipeline](#14-integration-with-rating-pipeline)
15. [Tax Engine Integration](#15-tax-engine-integration)
16. [Archival and Long-Term Storage](#16-archival-and-long-term-storage)
17. [API Design](#17-api-design)
18. [Roadmap and Phasing](#18-roadmap-and-phasing)
19. [Open Questions](#19-open-questions)
20. [Related Documents](#20-related-documents)

---

## 1. Executive Summary

E-invoicing — the generation, validation, digital signing, and submission of
legally compliant electronic invoices to tax authorities — is universally
outsourced in the billing platform market. Every billing platform (Stripe,
Zuora, Chargebee, Lago, Kill Bill) either lacks e-invoicing entirely or
delegates it to third-party tax engines like Avalara or Vertex (see
[competitive analysis](../../docs/competitive/legal-regulatory-compliance.md),
Gap 3).

This is a significant market gap because e-invoicing mandates are proliferating
globally. The EU ViDA directive (mandatory from July 2030), India's expanding
GST thresholds, Brazil's IBS and CBS reform (August 2026), and Saudi Arabia's
ZATCA Phase 2 are creating new compliance obligations in the 2026-2030
timeframe — exactly when Meteridian enters the market
([METR-0007 §4.1](../0007-legal-regulatory-compliance/legal-regulatory-compliance.md#41-global-e-invoicing-landscape)).

Meteridian's native e-invoicing engine addresses this gap through:

1. **Canonical invoice model** — an internal data model based on EN 16931 that
   captures all fields required by any jurisdiction's e-invoicing mandate
   (ADR-0016)

2. **Format generation framework** — a block-based architecture that transforms
   the canonical model into jurisdiction-specific output formats (UBL 2.1 XML,
   Factur-X PDF/A-3, CFDI 4.0, NF-e XML, Peppol BIS 3.0, etc.)

3. **Digital signature framework** — pluggable cryptographic signing that
   supports jurisdiction-specific requirements (ICP-Brasil certificates,
   ZATCA CSID, eIDAS qualified electronic signatures, XML-DSig)

4. **Tax authority integration** — blocks that submit invoices to government
   clearance systems (Brazil SEFAZ, India IRP, Saudi Arabia ZATCA Fatoora,
   Mexico SAT and PAC) and handle the async approval or rejection lifecycle

5. **Validation engine** — pre-submission validation against jurisdiction-
   specific schemas and business rules, preventing rejection by tax authorities

**Why native, not just integration?** Third-party tax engines (Avalara, Vertex)
are excellent at tax _calculation_ — determining the correct tax rate and
amount. But e-invoicing is more than tax calculation: it is format generation,
digital signing, government API integration, and lifecycle management. By
handling e-invoicing natively, Meteridian:

- Eliminates per-invoice API costs from third-party services
- Enables air-gapped and sovereign deployments where external API calls are
  prohibited
- Owns the full invoice lifecycle (generation → validation → signing →
  submission → approval or rejection → archival)
- Can optimize the format generation for Meteridian's specific data model
  rather than adapting to a generic tax engine's input format

Meteridian still integrates with external tax engines for tax _calculation_
(see §15). The e-invoicing engine handles everything that happens _after_ tax
amounts are determined.

---

## 2. Motivation

### 2.1 The Scale of the Problem

METR-0007 §4.1 catalogues e-invoicing mandates across 30+ jurisdictions. The
global trend is unmistakable:

| Model | Jurisdictions | How It Works |
|-------|--------------|-------------|
| **Clearance** | Brazil, India, Saudi Arabia, Mexico, Chile, Colombia, Turkey, Egypt, Malaysia | Invoice must be submitted to and approved by tax authority _before_ it is legally valid |
| **Real-time reporting** | Spain (SII and Veri*factu), EU ViDA (cross-border, from 2030) | Invoice data is reported to tax authority in real-time or near-real-time, but invoice is valid upon issuance |
| **Post-audit** | Germany (B2G via XRechnung), Japan (Qualified Invoice), Australia (Peppol) | Invoice format is mandated but submission is not required at time of issuance |

For **clearance-model** jurisdictions, the e-invoicing engine is on the
critical path: an invoice that has not been cleared by the tax authority is
not legally valid. This means the billing pipeline cannot complete without
e-invoicing integration.

### 2.2 Why This Cannot Be a Plugin

E-invoicing is not a post-processing decoration on an already-complete invoice.
It affects the billing pipeline at multiple points:

1. **Data collection** — the canonical model requires fields (buyer and seller tax
   IDs, place of supply, tax category codes) that must be captured during
   customer onboarding and product catalog configuration
2. **Rating** — tax calculation must happen during rating, not after invoice
   generation, because the rated output feeds into the canonical model
3. **Invoice generation** — the canonical model is populated from rated usage
   data, contract terms, and customer data
4. **Validation** — the invoice is validated against jurisdiction-specific
   schemas before any output is generated
5. **Format generation** — the canonical model is transformed into one or more
   output formats
6. **Signing** — digital signatures are applied using jurisdiction-specific
   certificates and algorithms
7. **Submission** — the signed invoice is submitted to the tax authority (for
   clearance models)
8. **Lifecycle management** — the tax authority may approve, reject, or request
   corrections; this status must flow back into the billing system
9. **Archival** — the signed, approved invoice must be stored in its original
   format for the jurisdiction-mandated retention period

This is a pipeline, not a plugin.

---

## 3. Competitive Landscape

### 3.1 Current State

| Platform | E-Invoicing Approach | Formats | Clearance | Limitation |
|----------|---------------------|---------|-----------|------------|
| **Avalara** | Native (core product) | Peppol, DBNA, many national | Yes (many) | Not a billing platform; integration-only |
| **Vertex** | Native (core product) | Many national formats | Yes | Not a billing platform; on-prem option |
| **SAP BRIM** | Native (via Document Compliance) | All major formats | Yes (global) | Enormous cost; 12-24 month implementation |
| **Stripe** | Via Stripe Invoicing | Limited (Thailand) | Limited | Not a full e-invoicing solution |
| **Zuora** | Via integrations (Avalara and Vertex) | None native | None native | Per-invoice API costs |
| **Chargebee** | Via integrations | None native | None native | Per-invoice API costs |
| **Lago** (OSS) | None | None | None | Completely absent |
| **Kill Bill** (OSS) | Via plugins | None native | None native | Plugin ecosystem limited |
| **OpenMeter** (OSS) | None | None | None | Not a billing platform |

### 3.2 Meteridian's Differentiation

Meteridian will be the first open-source metering and rating platform with native
e-invoicing. The combination of:

- **Open-source canonical model** (auditable, extensible)
- **Block-based format generators** (community-extensible for new jurisdictions)
- **Air-gapped capability** (no external API dependency for format generation
  and signing)
- **Native pipeline integration** (e-invoicing is part of the billing pipeline,
  not a post-processing step)

...creates a unique position: enterprise-grade e-invoicing without enterprise
pricing or vendor lock-in.

---

## 4. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Billing Pipeline                                 │
│                                                                          │
│  Usage Events → Rating Engine → Tax Calculation → Invoice Generation     │
│                                        │                                 │
│                                        ▼                                 │
│                            ┌──────────────────────┐                      │
│                            │  Canonical Invoice    │                      │
│                            │  Model (EN 16931)     │                      │
│                            └──────────┬───────────┘                      │
│                                       │                                  │
│                          ┌────────────┼────────────┐                     │
│                          ▼            ▼            ▼                     │
│                    ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│                    │Validation│ │Validation│ │Validation│               │
│                    │  (UBL)   │ │(Factur-X)│ │  (NF-e)  │  ...         │
│                    └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│                         ▼            ▼            ▼                     │
│                    ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│                    │ Format   │ │ Format   │ │ Format   │               │
│                    │Generator │ │Generator │ │Generator │  ...         │
│                    │ (UBL 2.1)│ │(Factur-X)│ │(NF-e XML)│               │
│                    └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│                         ▼            ▼            ▼                     │
│                    ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│                    │ Signing  │ │ Signing  │ │ Signing  │               │
│                    │(XML-DSig)│ │(CAdES/XL)│ │(ICP-Bras)│  ...         │
│                    └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│                         │            │            │                     │
│                         ▼            ▼            ▼                     │
│                    ┌─────────────────────────────────────┐               │
│                    │      Tax Authority Submission       │               │
│                    │  (Clearance / Reporting / Archive)  │               │
│                    └─────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

Each stage (validation, format generation, signing, submission) is implemented
as a block in Meteridian's block architecture (METR-0002), enabling:

- **Per-jurisdiction customization** without modifying the core pipeline
- **Community extension** for new jurisdictions through marketplace blocks
- **Independent versioning** of format generators (e.g., CFDI 4.0 → 5.0)
- **Air-gapped deployment** by including all necessary blocks locally

---

## 5. Canonical Invoice Model

### 5.1 Design Decision

ADR-0016 documents the decision to use **EN 16931** (the European electronic
invoicing standard) as the basis for Meteridian's canonical invoice model.
EN 16931 was chosen because:

- It is the most comprehensive e-invoicing data model, designed to be
  jurisdiction-agnostic while supporting jurisdiction-specific extensions
- It is mandated by the EU (700M+ population, largest billing market by
  regulatory complexity)
- UBL 2.1 (the most common e-invoicing XML format globally) is a syntactic
  binding of EN 16931
- Peppol BIS 3.0, XRechnung, and Factur-X are all profiles of EN 16931
- Non-European formats (CFDI, NF-e, ZATCA) can be mapped from EN 16931
  with jurisdiction-specific extensions

### 5.2 Core Data Groups

The canonical model follows EN 16931's information groups:

| ID | Group | Description | Key Fields |
|----|-------|-------------|------------|
| BG-1 | Invoice note | Free-text notes | Subject code, note text |
| BG-2 | Process control | Business process identification | Profile ID, customization ID |
| BG-4 | Seller | Seller party information | Name, address, tax ID, registration |
| BG-7 | Buyer | Buyer party information | Name, address, tax ID |
| BG-13 | Delivery information | Where and when delivered | Delivery date, location, party |
| BG-16 | Payment instructions | How to pay | Payment means, account, reference |
| BG-20 | Document allowances | Invoice-level discounts | Amount, reason, tax category |
| BG-21 | Document charges | Invoice-level charges and surcharges | Amount, reason, tax category |
| BG-22 | Document totals | Monetary totals | Net, tax, gross, prepaid, payable |
| BG-23 | VAT breakdown | Tax breakdown by category and rate | Category, rate, base, amount |
| BG-25 | Invoice line | Line items | Description, quantity, net price, tax |
| BG-30 | Line allowances and charges | Line-level discounts and surcharges | Amount, reason, percentage |

### 5.3 Meteridian Extensions

EN 16931 covers standard invoicing. Meteridian extends it for metering-specific
scenarios:

| Extension Group | Fields | Purpose |
|----------------|--------|---------|
| MG-1 (Metering metadata) | `metering_period_start`, `metering_period_end`, `usage_summary` | Links invoice lines to underlying usage data |
| MG-2 (Credit and prepaid) | `credit_grant_id`, `balance_before`, `balance_after`, `drawdown_amount` | Links to credit system (METR-0004) |
| MG-3 (Rating details) | `rate_plan_id`, `tier_applied`, `unit_rate`, `quantity_rated` | Rating transparency for dispute resolution |
| MG-4 (Multi-party) | `reseller_party`, `end_user_party`, `platform_party` | MSP/marketplace billing chains |

### 5.4 Data Model (Go)

```go
type CanonicalInvoice struct {
    InvoiceNumber       string              `json:"invoice_number"`
    IssueDate           time.Time           `json:"issue_date"`
    DueDate             time.Time           `json:"due_date"`
    InvoiceTypeCode     string              `json:"invoice_type_code"` // EN 16931 code list
    CurrencyCode        string              `json:"currency_code"`    // ISO 4217
    TaxCurrencyCode     string              `json:"tax_currency_code,omitempty"`
    TaxPointDate        *time.Time          `json:"tax_point_date,omitempty"`
    Seller              Party               `json:"seller"`
    Buyer               Party               `json:"buyer"`
    Delivery            *DeliveryInfo       `json:"delivery,omitempty"`
    PaymentInstructions PaymentInstructions `json:"payment_instructions"`
    Lines               []InvoiceLine       `json:"lines"`
    Allowances          []AllowanceCharge   `json:"allowances,omitempty"`
    Charges             []AllowanceCharge   `json:"charges,omitempty"`
    TaxBreakdown        []TaxBreakdown      `json:"tax_breakdown"`
    Totals              DocumentTotals      `json:"totals"`
    Notes               []Note              `json:"notes,omitempty"`
    // Meteridian extensions
    MeteringMetadata    *MeteringMetadata   `json:"metering_metadata,omitempty"`
    CreditInfo          *CreditInfo         `json:"credit_info,omitempty"`
    RatingDetails       []RatingDetail      `json:"rating_details,omitempty"`
}
```

---

## 6. Format Generation Framework

### 6.1 Format Generator Interface

Each format generator is a block (METR-0002) that implements:

```go
type FormatGenerator interface {
    block.Block
    SupportedFormat() string           // e.g., "UBL-2.1", "Factur-X", "NF-e"
    SupportedJurisdictions() []string  // e.g., ["EU", "SA", "AU"]
    Generate(invoice *CanonicalInvoice) ([]byte, string, error) // output, content-type, error
    Validate(invoice *CanonicalInvoice) []ValidationError       // pre-generation validation
}
```

### 6.2 Planned Format Generators

| Block | Format | Standard | Jurisdictions | Priority |
|-------|--------|----------|--------------|----------|
| `einvoice-gen-ubl21` | UBL 2.1 XML | OASIS UBL 2.1 | EU, Saudi Arabia, Australia, Singapore | P0 |
| `einvoice-gen-facturx` | Factur-X / ZUGFeRD | EN 16931 + PDF/A-3 | France, Germany, EU | P0 |
| `einvoice-gen-peppol` | Peppol BIS 3.0 | EN 16931 profile | EU, Australia, Singapore, NZ | P1 |
| `einvoice-gen-xrechnung` | XRechnung | EN 16931 CIUS | Germany (B2G) | P1 |
| `einvoice-gen-cfdi40` | CFDI 4.0 XML | SAT schema | Mexico | P1 |
| `einvoice-gen-nfe` | NF-e XML | SEFAZ schema | Brazil | P1 |
| `einvoice-gen-nfse` | NFS-e XML | ABRASF schema | Brazil (services) | P2 |
| `einvoice-gen-zatca` | ZATCA UBL 2.1 | ZATCA profile of UBL | Saudi Arabia | P1 |
| `einvoice-gen-gst-india` | GST e-Invoice JSON | IRP schema | India | P1 |
| `einvoice-gen-fapiao` | VAT Fapiao | Golden Tax schema | China | P2 |
| `einvoice-gen-qualified-jp` | Qualified Invoice | Japanese QI format | Japan | P2 |
| `einvoice-gen-etax-kr` | e-Tax Invoice | NTS schema | South Korea | P2 |
| `einvoice-gen-myinvois` | MyInvois | LHDN schema | Malaysia | P2 |

### 6.3 Format Generator Block Contract

Format generator blocks receive a `CanonicalInvoice` as an Arrow RecordBatch
and return the formatted output:

- **Input schema:** Arrow schema matching the canonical invoice model
- **Output schema:** Arrow schema with `format` (string), `content_type`
  (string), `content` (binary), `validation_errors` (list<string>)
- **Metadata:** Jurisdiction, format version, schema version

---

## 7. Digital Signature Framework

### 7.1 Jurisdiction Requirements

| Jurisdiction | Signature Type | Certificate Standard | Algorithm | Reference |
|-------------|---------------|---------------------|-----------|-----------|
| Brazil | XML-DSig (enveloped) | ICP-Brasil (A1/A3) | RSA-SHA256 | NF-e Manual de Orientação |
| Saudi Arabia | ZATCA Cryptographic Stamp | ZATCA CSID (Compliance + Production) | ECDSA P-256 | ZATCA Technical Specs |
| EU (eIDAS) | XAdES-BES/T/XL/A | Qualified Electronic Signature (QES) | RSA or ECDSA | eIDAS Regulation (EU) 910/2014 |
| Mexico | XML-DSig | CSD (Certificado de Sello Digital) | RSA-SHA256 | SAT CFDI specs |
| India | Not required for e-invoice | N/A (IRP signs) | N/A | GST e-Invoice API |
| Italy | CAdES or XAdES | Qualified certificate | RSA or ECDSA | SdI technical rules |
| France | XAdES (for some formats) | RGS-qualified | RSA or ECDSA | ANSSI RGS |

### 7.2 Signing Block Interface

```go
type SigningBlock interface {
    block.Block
    SupportedSignatureType() string        // e.g., "XML-DSig", "CAdES", "ZATCA-CSID"
    SupportedJurisdictions() []string
    Sign(document []byte, config SigningConfig) ([]byte, error)
    Verify(signedDocument []byte) (SignatureVerification, error)
}

type SigningConfig struct {
    CertificatePath  string            // Path to signing certificate (PEM/PKCS12)
    PrivateKeyPath   string            // Path to private key (PEM/PKCS8)
    HSMConfig        *HSMConfig        // Optional: external HSM/KMS configuration
    TimestampURL     string            // RFC 3161 timestamping service URL
    JurisdictionOpts map[string]string // Jurisdiction-specific options
}
```

### 7.3 Planned Signing Blocks

| Block | Signature Type | Jurisdictions | Priority |
|-------|---------------|--------------|----------|
| `einvoice-sign-xmldsig` | XML-DSig (enveloped) | Brazil (NF-e), Mexico (CFDI) | P1 |
| `einvoice-sign-xades` | XAdES-BES/T/XL | EU (eIDAS), France, Italy | P1 |
| `einvoice-sign-cades` | CAdES-BES/T | Italy (SdI alternative) | P2 |
| `einvoice-sign-zatca` | ZATCA Cryptographic Stamp | Saudi Arabia | P1 |
| `einvoice-sign-icpbrasil` | ICP-Brasil A1/A3 | Brazil | P1 |
| `einvoice-sign-csd-mexico` | CSD signing | Mexico | P2 |

### 7.4 HSM and KMS Integration

For production deployments, private keys should not reside on disk. The signing
framework supports external key management:

- **PKCS#11** — standard interface for hardware security modules (Thales Luna,
  nCipher, AWS CloudHSM, Azure Dedicated HSM)
- **AWS KMS** / **Azure Key Vault** / **GCP Cloud KMS** — cloud KMS integration
  for cloud deployments
- **HashiCorp Vault** — for operators using Vault as their secrets manager

This aligns with METR-0007 §11.3 (BYOK/HYOK key management for sovereign
cloud).

---

## 8. Tax Authority Integration

### 8.1 Clearance Model Lifecycle

For jurisdictions using the clearance model, the invoice lifecycle involves
interaction with the tax authority:

```
Generate → Validate → Sign → Submit → Wait → Approved/Rejected
                                                    │
                                         ┌──────────┴──────────┐
                                         ▼                      ▼
                                    Approved:              Rejected:
                                    - Store approved        - Parse errors
                                    - Deliver to buyer      - Correct invoice
                                    - Record in audit       - Resubmit
```

### 8.2 Tax Authority Connector Interface

```go
type TaxAuthorityConnector interface {
    block.Block
    Authority() string                      // e.g., "SEFAZ", "IRP", "ZATCA", "SAT"
    SupportedJurisdictions() []string
    Submit(signedInvoice []byte) (SubmissionResult, error)
    CheckStatus(submissionID string) (SubmissionStatus, error)
    Cancel(invoiceID string, reason string) (CancellationResult, error)
}

type SubmissionResult struct {
    SubmissionID   string
    Status         string    // "accepted", "pending", "rejected"
    AuthorizationCode string // e.g., SEFAZ protocol number, IRP IRN
    Errors         []AuthorityError
    Timestamp      time.Time
}
```

### 8.3 Planned Tax Authority Connectors

| Block | Tax Authority | Model | Jurisdictions | Priority |
|-------|--------------|-------|--------------|----------|
| `einvoice-submit-sefaz` | SEFAZ (Secretaria da Fazenda) | Clearance | Brazil | P1 |
| `einvoice-submit-irp` | IRP (Invoice Registration Portal) | Clearance | India | P1 |
| `einvoice-submit-zatca` | ZATCA Fatoora Platform | Clearance (B2B) / Reporting (B2C) | Saudi Arabia | P1 |
| `einvoice-submit-sat` | SAT via PAC (Proveedor Autorizado de Certificación) | Clearance | Mexico | P2 |
| `einvoice-submit-sdi` | SdI (Sistema di Interscambio) | Clearance | Italy | P2 |
| `einvoice-submit-gib` | GİB (Gelir İdaresi Başkanlığı) | Clearance | Turkey | P2 |
| `einvoice-submit-sii-spain` | SII (Suministro Inmediato de Información) | Real-time reporting | Spain | P2 |
| `einvoice-submit-eta` | ETA (Egyptian Tax Authority) | Clearance | Egypt | P3 |
| `einvoice-submit-peppol` | Peppol Access Point | Post-audit | EU, AU, SG, NZ | P1 |
| `einvoice-submit-chorus` | Chorus Pro | Clearance (B2G) | France | P2 |

---

## 9. Validation Engine

### 9.1 Validation Layers

Validation happens before format generation to catch errors early and avoid
rejection by tax authorities:

| Layer | What It Validates | Examples |
|-------|------------------|---------|
| **Schema validation** | Canonical model completeness and type correctness | Required fields present, dates valid, amounts non-negative |
| **Business rule validation** | Jurisdiction-specific business rules | Tax category matches VAT rate, payment means code valid for jurisdiction |
| **Cross-field validation** | Relationships between fields | Line totals sum to document total, tax breakdown matches line taxes |
| **Authority-specific validation** | Rules specific to a tax authority's implementation | ZATCA required fields (CSID, PIH hash), SEFAZ municipality codes |

### 9.2 Validation Rule Distribution

Validation rules are packaged within format generator blocks and as standalone
validator blocks. This ensures that:

- Format generators can validate before generating (preventing invalid output)
- Standalone validators can be run independently (e.g., during invoice preview)
- Validation rules are versioned alongside format generators

---

## 10. Sequential Invoice Numbering

METR-0007 §8.2 identifies sequential, gap-free invoice numbering as a MUST
requirement across virtually all jurisdictions. METR-0008 §5.4 defines the
general sequential numbering service. This section specifies the e-invoicing-
specific requirements.

### 10.1 Numbering Scheme

| Requirement | Implementation |
|------------|---------------|
| **Gap-free** | PostgreSQL sequence with `NO CYCLE`; gaps trigger compliance alerts |
| **Tenant-isolated** | Separate sequence per tenant |
| **Jurisdiction-aware** | Some jurisdictions require specific numbering formats (e.g., CFDI with series prefix, NF-e with series + sequential) |
| **Chronologically ordered** | Invoice numbers must be strictly increasing with time |
| **Format configurable** | Operators can configure prefix, suffix, padding, series per jurisdiction |

### 10.2 Multi-Series Support

Some jurisdictions and business models require multiple invoice series:

- **Brazil NF-e**: Series number (1-999) + sequential within series
- **Mexico CFDI**: Series prefix (alphabetic) + folio (numeric)
- **Spain**: Different series for different business lines or locations

Meteridian supports per-tenant, per-jurisdiction, per-series numbering:

```
{tenant_id}:{jurisdiction}:{series} → atomic_counter
```

---

## 11. Multi-Currency and Exchange Rates

### 11.1 Currency in E-Invoicing

E-invoices must specify the invoice currency and, if different from the tax
reporting currency, the exchange rate used. Many jurisdictions mandate specific
exchange rate sources:

| Jurisdiction | Tax Reporting Currency | Exchange Rate Source |
|-------------|----------------------|-------------------|
| EU | Member State currency (EUR for eurozone) | ECB reference rates on date of supply |
| India | INR | RBI reference rate on date of invoice |
| Brazil | BRL | BCB PTAX rate on business day prior to event |
| Saudi Arabia | SAR | SAMA rate |
| UK | GBP | HMRC rates or market rate |

### 11.2 Exchange Rate Service

Meteridian provides a pluggable exchange rate service:

- **Built-in**: ECB, RBI, BCB PTAX rate fetchers (scheduled daily)
- **Pluggable**: Operators can provide custom exchange rate sources via blocks
- **Cached**: Rates are cached in Valkey with the rate date as key
- **Audited**: Every exchange rate used in an invoice is recorded in the audit
  trail with its source, date, and rate value

---

## 12. QR Code Generation

Several jurisdictions require QR codes on invoices:

| Jurisdiction | QR Code Contents | Encoding |
|-------------|-----------------|---------|
| Saudi Arabia (ZATCA) | Invoice hash, seller VAT, timestamp, amounts | Base64-encoded TLV |
| India (GST) | IRN, signed QR per IRP | Generated by IRP; embedded by Meteridian |
| Brazil (NF-e) | Access key URL | URL encoding |
| Mexico (CFDI) | UUID, amounts, digital stamp | URL encoding |

QR code generation is a lightweight utility within the format generator blocks,
using standard QR encoding libraries. Output supports both embedded (in PDF
invoices) and standalone (for print/display) modes.

---

## 13. Integration with Block Architecture

### 13.1 E-Invoicing as a Pipeline Stage

E-invoicing blocks integrate into Meteridian's billing pipeline as a terminal
stage:

```yaml
pipeline:
  name: "billing-pipeline-eu"
  stages:
    - block: "usage-aggregator"
    - block: "rating-engine"
    - block: "tax-calc-avalara"          # Tax calculation (external)
    - block: "invoice-generator"          # Canonical model population
    - block: "einvoice-gen-facturx"       # Format generation
    - block: "einvoice-sign-xades"        # Digital signing
    - block: "einvoice-submit-peppol"     # Submission
```

### 13.2 Multi-Format Output

A single invoice can generate multiple output formats:

```yaml
# For a German customer: generate both Factur-X (for the buyer) and
# XRechnung (for government reporting)
einvoice_output:
  formats:
    - block: "einvoice-gen-facturx"
      purpose: "buyer_delivery"
    - block: "einvoice-gen-xrechnung"
      purpose: "government_reporting"
```

### 13.3 Block Categories

E-invoicing blocks are organized in the marketplace under the `einvoice`
category, complementing the `compliance` category defined in METR-0008:

| Category | Block Types | Examples |
|----------|-----------|---------|
| `einvoice/generator` | Format generators | `einvoice-gen-ubl21`, `einvoice-gen-nfe` |
| `einvoice/signer` | Digital signing | `einvoice-sign-xmldsig`, `einvoice-sign-zatca` |
| `einvoice/validator` | Pre-submission validation | `einvoice-validate-zatca`, `einvoice-validate-sefaz` |
| `einvoice/connector` | Tax authority submission | `einvoice-submit-sefaz`, `einvoice-submit-irp` |
| `einvoice/exchange-rate` | Exchange rate fetchers | `rate-ecb`, `rate-rbi`, `rate-bcb` |

---

## 14. Integration with Rating Pipeline

The e-invoicing engine requires specific data from the rating pipeline that
must be propagated through the billing process:

### 14.1 Tax-Relevant Data in Rating Output

| Field | Source | E-Invoice Use |
|-------|--------|-------------|
| `tax_category_code` | Product catalog (METR-0003) | EN 16931 BT-151 (Invoice line VAT category code) |
| `tax_rate_percentage` | Tax engine or catalog | EN 16931 BT-152 (Invoiced item VAT rate) |
| `place_of_supply` | Customer data + supply rules | Determines applicable tax jurisdiction |
| `tax_exemption_reason` | Tax engine | EN 16931 BT-120 (VAT exemption reason text) |
| `item_classification` | Product catalog | Tax category classification (UNSPSC, CPV, HS codes) |

### 14.2 Product Catalog Requirements

METR-0003 (Product and Service Catalog) must support:

- **Tax classification codes** per product/service (UNSPSC, CPV, HS)
- **Tax category** per price book entry (standard, reduced, zero-rated, exempt)
- **Place of supply rules** per product type (goods vs. services, digital vs.
  physical)
- **Withholding tax applicability** per jurisdiction

---

## 15. Tax Engine Integration

Meteridian does **not** replace external tax engines for tax _calculation_. Tax
calculation — determining the correct tax rate, exemptions, and amounts for a
given transaction — is handled by integrations with specialized providers:

| Integration | Provider | Approach |
|------------|---------|----------|
| `tax-calc-avalara` | Avalara AvaTax | Block that calls Avalara API for rate determination |
| `tax-calc-vertex` | Vertex O Series | Block that calls Vertex API |
| `tax-calc-anrok` | Anrok | Block that calls Anrok API |
| `tax-calc-stripe` | Stripe Tax | Block that calls Stripe Tax API |
| `tax-calc-internal` | Meteridian built-in | Simple multi-rate VAT/GST engine for basic scenarios |

The **boundary** between tax calculation and e-invoicing is:

- Tax engine: "This transaction is taxed at 19% standard VAT in Germany"
- E-invoicing engine: "Generate a Factur-X invoice with this tax breakdown,
  sign it with a qualified certificate, and deliver via Peppol"

The built-in `tax-calc-internal` block handles straightforward cases (single
jurisdiction, known rates) to enable air-gapped deployments and reduce
dependency on external services for simple scenarios. It is not a substitute
for Avalara or Vertex in complex multi-jurisdiction environments.

---

## 16. Archival and Long-Term Storage

### 16.1 Retention Requirements

Signed e-invoices must be archived in their original format for jurisdiction-
mandated retention periods (catalogued in METR-0007 §8.1):

| Jurisdiction | Retention Period | Format Requirement |
|-------------|-----------------|-------------------|
| Germany | 10 years | Original format (GoBD) |
| France | 10 years | Original format |
| Brazil | 5 years | Original NF-e XML with SEFAZ authorization |
| India | 8 years | Original IRP-signed JSON/QR |
| Saudi Arabia | 6 years | Original ZATCA-stamped XML |
| EU general | 10 years (varies by Member State) | VAT Directive Art. 247 |

### 16.2 Archival Architecture

- **Hot storage (0-2 years):** TimescaleDB with original signed documents
  stored as BYTEA or linked to S3-compatible object storage
- **Warm storage (2-7 years):** Compressed and moved to lower-cost
  S3-compatible storage; retrievable within minutes
- **Cold storage (7+ years):** Glacier-class storage for jurisdictions
  requiring 10+ year retention
- **Integrity:** Hash of archived document is recorded in the audit trail
  (METR-0008); periodic integrity verification against the hash

---

## 17. API Design

### 17.1 Invoice Generation API

```
POST   /api/v1/invoices/generate               # Generate invoice from billing data
POST   /api/v1/invoices/{id}/format             # Generate e-invoice format(s) from existing invoice
POST   /api/v1/invoices/{id}/sign               # Sign a formatted invoice
POST   /api/v1/invoices/{id}/submit             # Submit to tax authority
GET    /api/v1/invoices/{id}/status             # Check submission status
POST   /api/v1/invoices/{id}/cancel             # Cancel/void an invoice
```

### 17.2 E-Invoice Format API

```
GET    /api/v1/einvoice/formats                 # List available format generators
GET    /api/v1/einvoice/formats/{format}/schema  # Get format schema/requirements
POST   /api/v1/einvoice/validate                # Validate invoice against format rules
```

### 17.3 Exchange Rate API

```
GET    /api/v1/exchange-rates/{from}/{to}       # Get exchange rate
GET    /api/v1/exchange-rates/{from}/{to}/history # Historical rates
POST   /api/v1/exchange-rates/sources           # Configure rate sources
```

---

## 18. Roadmap and Phasing

### Phase 1: v1.0 — Foundation

| Capability | Description |
|-----------|------------|
| Canonical invoice model | EN 16931-based data model with Meteridian extensions |
| UBL 2.1 generator | Core format used by ZATCA, Peppol, and many jurisdictions |
| Factur-X generator | EU-focused format combining structured XML with PDF |
| Basic validation engine | Schema + business rule validation for supported formats |
| Sequential numbering | Gap-free, tenant-isolated, jurisdiction-aware numbering |
| Built-in tax calculator | Simple multi-rate VAT/GST engine for air-gapped deployment |
| Avalara integration block | Tax calculation via Avalara AvaTax |
| Multi-currency support | ISO 4217 currencies, ECB exchange rate fetcher |

### Phase 2: v1.x — Clearance and Signing

| Capability | Description |
|-----------|------------|
| XML-DSig signing | For Brazil (NF-e) and Mexico (CFDI) |
| XAdES signing | For EU (eIDAS) |
| ZATCA cryptographic stamp | For Saudi Arabia |
| ICP-Brasil signing block | For Brazil |
| SEFAZ connector | Brazil NF-e clearance |
| IRP connector | India GST e-invoice clearance |
| ZATCA connector | Saudi Arabia Fatoora clearance |
| Peppol Access Point | EU/AU/SG invoice delivery |
| NF-e format generator | Brazil-specific |
| CFDI 4.0 generator | Mexico-specific |
| India GST JSON generator | India-specific |
| HSM/KMS integration | PKCS#11, AWS KMS, Azure Key Vault, GCP KMS |
| Vertex integration block | Tax calculation via Vertex |

### Phase 3: v2.0 — Global Expansion

| Capability | Description |
|-----------|------------|
| XRechnung generator | Germany B2G |
| Chorus Pro connector | France B2G clearance |
| SdI connector | Italy clearance |
| SII connector | Spain real-time reporting |
| Golden Tax fapiao | China |
| MyInvois connector | Malaysia |
| e-Tax Invoice generator | South Korea |
| Advanced validation | Jurisdiction-specific business rule libraries |
| QR code enhancements | ZATCA TLV, India IRN QR, Brazil access key |

---

## 19. Open Questions

### 19.1 Architectural

1. **Peppol Access Point certification**: Should Meteridian include a certified
   Peppol Access Point, or should operators use third-party Access Points?
   Certification requires organizational audit, not just technical compliance.

2. **PAC relationship for Mexico**: CFDI clearance requires a PAC (Proveedor
   Autorizado de Certificación). Should Meteridian partner with a PAC or
   provide the integration interface for operators to connect their own PAC?

3. **IRP sandbox vs. production**: India's GST IRP provides sandbox
   environments for testing. Should the IRP connector block support automatic
   environment switching based on deployment configuration?

### 19.2 Format Strategy

4. **Canonical model completeness**: Can EN 16931 truly serve as a universal
   superset, or will some jurisdictions (China Golden Tax, Japan Qualified
   Invoice) require a fundamentally different data model? Initial analysis
   suggests mapping is feasible but needs validation with production data.

5. **PDF/A-3 generation**: Factur-X embeds XML in a PDF/A-3 document. PDF
   generation is a significant engineering effort. Should this use a Go PDF
   library, an external rendering service, or a headless browser?

### 19.3 Operational

6. **Certificate management UX**: Operators must provision signing certificates
   (ICP-Brasil, ZATCA CSID, eIDAS QES). How should Meteridian manage the
   certificate lifecycle (procurement, installation, renewal, revocation)?

7. **Fallback behavior**: If a tax authority API is down (clearance model),
   should the invoice be queued for retry or should the billing pipeline block?
   Different jurisdictions may have different rules about how long an invoice
   can remain uncleared.

---

## 20. Related Documents

### Internal

- [METR-0001: Platform Architecture](../0001-architecture/architecture.md) —
  event store and data layer that the e-invoicing engine builds upon
- [METR-0002: Platform Extensibility](../0002-extensibility/extensibility.md) —
  block architecture used for format generators, signers, and connectors
- [METR-0003: Product and Service Catalog](../0003-product-catalog/product-catalog.md) —
  tax classification and product metadata required for e-invoicing
- [METR-0004: Credit, Prepaid, and Token-Based Billing](../0004-credit-token-billing/credit-token-billing.md) —
  credit grants and prepaid balances appear as invoice line items
- [METR-0007: Legal and Regulatory Compliance](../0007-legal-regulatory-compliance/legal-regulatory-compliance.md) —
  the regulatory landscape driving e-invoicing requirements (§4, §8, §13)
- [METR-0008: Compliance-as-Code](../0008-compliance-as-code/compliance-as-code.md) —
  audit trail and sequential numbering infrastructure shared with e-invoicing
- [ADR-0010: Pluggable Payment Providers](../../docs/adr/0010-pluggable-payment-providers.md) —
  payment collection is separate from invoice generation
- [ADR-0016: E-Invoice Canonical Model and Format Selection](../../docs/adr/0016-einvoice-canonical-model.md) —
  decision to use EN 16931 as canonical model

### External References

- [EN 16931 — European e-invoicing standard](https://ec.europa.eu/digital-building-blocks/sites/display/DIGITAL/EN+16931) — canonical model basis
- [OASIS UBL 2.1](http://docs.oasis-open.org/ubl/UBL-2.1.html) — primary XML format
- [Peppol BIS Billing 3.0](https://docs.peppol.eu/poacc/billing/3.0/) — EU/global e-invoicing profile
- [Factur-X / ZUGFeRD](https://fnfe-mpe.org/factur-x/) — hybrid PDF/XML format
- [ZATCA E-Invoicing Technical Specification](https://zatca.gov.sa/en/E-Invoicing/) — Saudi Arabia
- [India GST e-Invoice API](https://einvoice1.gst.gov.in/) — India clearance
- [Brazil NF-e Technical Manual](https://www.nfe.fazenda.gov.br/) — Brazil clearance
- [Mexico CFDI 4.0](https://www.sat.gob.mx/consultas/35025/formato-de-factura-electronica-(anexo-20)) — Mexico clearance
