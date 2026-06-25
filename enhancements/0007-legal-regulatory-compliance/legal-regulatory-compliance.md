# METR-0007: Legal and Regulatory Compliance Framework

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0001 (Platform Architecture), METR-0002 (Extensibility), METR-0003 (Product Catalog)
- **Related:** METR-0004 (Credit and Token Billing), METR-0005 (Internal Budget Units), METR-0006 (Developer Experience), METR-0008 (Compliance-as-Code), METR-0009 (E-Invoicing Engine), ADR-0014 (Cryptographic Audit Trail), ADR-0015 (Compliance Policy Engine), ADR-0016 (E-Invoice Canonical Model), ADR-0017 (FedRAMP Boundary Architecture)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Motivation](#2-motivation)
3. [Use Cases and Compliance Tiers](#3-use-cases-and-compliance-tiers)
4. [Tax and E-Invoicing](#4-tax-and-e-invoicing)
5. [Data Protection and Privacy](#5-data-protection-and-privacy)
6. [Financial Regulations](#6-financial-regulations)
7. [Contract Law and Consumer Protection](#7-contract-law-and-consumer-protection)
8. [Audit, Record Retention, and Reporting](#8-audit-record-retention-and-reporting)
9. [Industry-Specific Regulations](#9-industry-specific-regulations)
10. [Standards and Certifications](#10-standards-and-certifications)
11. [Sovereign Cloud Requirements](#11-sovereign-cloud-requirements)
12. [Open Source Licensing Considerations](#12-open-source-licensing-considerations)
13. [Competitive Landscape](#13-competitive-landscape)
14. [Jurisdiction-by-Jurisdiction Reference](#14-jurisdiction-by-jurisdiction-reference)
15. [Compliance Requirements Matrix](#15-compliance-requirements-matrix)
16. [Impact on Meteridian Architecture](#16-impact-on-meteridian-architecture)
17. [Roadmap Recommendations](#17-roadmap-recommendations)
18. [Open Questions](#18-open-questions)
19. [Related Documents](#19-related-documents)

---

## 1. Executive Summary

Meteridian operates at the intersection of metering, rating, and billing —
domains that are among the most heavily regulated in commercial software. A
billing platform that produces invoices, calculates taxes, manages multi-year
contracts, and handles financial data must comply with a complex web of
regulations that varies by jurisdiction, industry, and deployment model.

This METR provides a comprehensive analysis of the legal and regulatory
landscape across three primary use cases:

1. **Internal FinOps, Showback, and Chargeback** — enterprises using Meteridian
   internally to allocate cloud costs across departments and teams
2. **Managed Service Provider (MSP)** — B2B billing with multi-year contractual
   commitments
3. **Neocloud and Sovereign Cloud** — new cloud providers and government-regulated
   industry deployments

The analysis covers regulations across major global markets and categorizes
requirements into three tiers:

| Tier | Meaning | Consequence of non-compliance |
|------|---------|-------------------------------|
| **MUST** | Mandatory legal requirement | Fines, criminal liability, inability to operate |
| **SHOULD** | Industry standard or contractual expectation | Loss of enterprise deals, audit failures |
| **COULD** | Competitive differentiator | Builds trust, wins RFPs, opens new markets |

**Key finding:** The regulatory burden increases dramatically from Use Case 1
(internal) to Use Case 3 (sovereign cloud). Internal FinOps has minimal
mandatory requirements beyond data protection. MSP billing triggers tax,
invoicing, and contract law obligations. Sovereign cloud adds data residency,
government procurement, and security certification requirements on top of
everything else.

**Critical architectural implication:** Meteridian must be designed from the
ground up with regulatory extensibility — the ability to plug in jurisdiction-
specific tax engines, e-invoicing connectors, data residency controls, and
audit trail formatters. This is not a feature that can be bolted on later; it
must be a core architectural concern (see §16).

---

## 2. Motivation

### Why this matters now

1. **Market access depends on compliance.** An MSP in Brazil cannot issue a
   single invoice without NF-e integration. A sovereign cloud operator in Saudi
   Arabia must integrate with ZATCA's Fatoora platform. A SaaS billing platform
   that cannot support these requirements is excluded from these markets entirely.

2. **Compliance is a competitive moat.** Billing platforms that handle tax
   calculation, e-invoicing, and audit trails natively win enterprise deals
   against platforms that require customers to integrate third-party tax engines.

3. **Regulatory momentum is accelerating.** The EU's ViDA directive (adopted
   March 2025, mandatory from July 2030), India's expanding GST e-invoicing
   thresholds, Brazil's IBS and CBS tax reform (August 2026), and Saudi Arabia's
   ZATCA Phase 2 rollout are all creating new compliance obligations in the
   2026-2030 timeframe — exactly when Meteridian will be entering the market.

4. **Sovereign cloud is a first-class use case.** Government procurement
   requirements (FedRAMP in the US, EUCS in the EU, national schemes in France,
   Germany, India, and others) impose data residency, operational sovereignty,
   and security certification requirements that affect architectural decisions
   at every layer of the platform.

### What this document is NOT

- This is not legal advice. It is a technical analysis of the regulatory
  landscape to inform architectural decisions.
- Specific regulatory citations are provided where confidently known. Where the
  exact statute number or regulation name is uncertain, the regulatory category
  and jurisdiction are identified for follow-up with legal counsel.
- This document does not cover employment law, environmental regulations, or
  other domains that do not directly affect billing platform design.

---

## 3. Use Cases and Compliance Tiers

### Use Case 1: Internal FinOps, Showback, and Chargeback

An enterprise deploys Meteridian on-premises or in a private cloud to allocate
infrastructure costs across departments, teams, or projects. No external
invoices are generated. No money changes hands between legal entities.

**Regulatory exposure:** Low-to-moderate. Data protection applies to any
personal data processed (employee identifiers, usage patterns). Tax and
invoicing requirements generally do not apply because no commercial transaction
occurs. However, transfer pricing rules may apply in multi-national enterprises
where cost allocations cross legal entity boundaries.

### Use Case 2: MSP and B2B Billing

A managed service provider uses Meteridian to bill customers under multi-year
contracts. Invoices are generated, taxes are calculated, and payments are
collected. The MSP operates in one or more jurisdictions.

**Regulatory exposure:** High. Tax calculation, e-invoicing, contract law,
record retention, and potentially financial regulation (if the platform handles
payments or extends credit) all apply. Multi-jurisdictional operation multiplies
the compliance burden.

### Use Case 3: Neocloud and Sovereign Cloud

A new cloud provider or sovereign cloud operator uses Meteridian as its billing
platform. The operator may serve government customers, regulated industries
(healthcare, financial services, defense), or operate under data sovereignty
mandates.

**Regulatory exposure:** Very high. Everything from Use Case 2, plus government
procurement requirements, security certifications, data residency enforcement,
and potentially sector-specific regulations (HIPAA, PCI-DSS, telecom billing
rules).

---

## 4. Tax and E-Invoicing

Tax and invoicing regulations are the single largest area of mandatory
compliance for any billing platform operating outside a purely internal context.
The global trend is unmistakable: governments are mandating real-time or
near-real-time electronic invoicing with direct integration into tax authority
systems.

### 4.1 Global E-Invoicing Landscape

| Jurisdiction | Mandate | Model | Status (June 2026) | Use Case Impact |
|---|---|---|---|---|
| **EU (ViDA)** | Cross-border B2B e-invoicing | Post-audit → Real-time reporting | Adopted March 2025; mandatory July 2030; Member State transposition by Dec 2027 | UC2, UC3 |
| **Germany** | XRechnung (B2G), expanding to B2B | Peppol/EN 16931 | B2G mandatory; B2B mandate 2027-2028 | UC2, UC3 |
| **France** | Factur-X / Chorus Pro | Clearance (B2G), Y-model (B2B) | B2G mandatory; B2B phased 2026-2028 | UC2, UC3 |
| **Italy** | Sistema di Interscambio (SdI) | Clearance | Fully mandatory since 2019 (pioneer) | UC2, UC3 |
| **Spain** | SII and Veri*factu | Real-time reporting | Veri*factu mandatory for certain taxpayers 2026 | UC2, UC3 |
| **India (GST)** | E-invoicing via IRP | Clearance | Mandatory for turnover > ₹5 crore; 30-day upload for ≥₹10 crore; ₹2 crore threshold proposed | UC2, UC3 |
| **Brazil** | NF-e, NFS-e, CT-e | Clearance (SEFAZ) | IBS and CBS fields required from Aug 3, 2026; ICP-Brasil digital signature mandatory | UC2, UC3 |
| **Saudi Arabia (ZATCA)** | Fatoora | Clearance (B2B), Reporting (B2C) | Phase 2 Wave 24 deadline June 30, 2026 (≥SAR 375K); UBL 2.1 XML; cryptographic stamp | UC2, UC3 |
| **Mexico** | CFDI | Clearance (SAT and PAC) | Fully mandatory; CFDI 4.0 current version | UC2, UC3 |
| **Chile** | DTE | Clearance (SII) | Fully mandatory | UC2, UC3 |
| **Colombia** | Factura Electrónica | Clearance (DIAN) | Fully mandatory | UC2, UC3 |
| **Turkey** | e-Fatura and e-Arşiv | Clearance (GİB) | Mandatory for most businesses | UC2, UC3 |
| **Egypt** | e-Invoice | Clearance (ETA) | Mandatory for all taxpayers | UC2, UC3 |
| **Japan** | Qualified Invoice System | Post-audit | Mandatory since October 2023; Qualified Invoice Issuer registration required | UC2, UC3 |
| **South Korea** | e-Tax Invoice | Clearance (NTS) | Mandatory for most businesses | UC2, UC3 |
| **Australia** | Peppol e-Invoicing | Post-audit / Peppol network | B2G mandatory for Commonwealth agencies; B2B voluntary but growing | UC2, UC3 |
| **Malaysia** | MyInvois | Clearance (LHDN) | Phased rollout 2024-2025; mandatory for all by mid-2025 | UC2, UC3 |
| **Singapore** | InvoiceNow (Peppol) | Post-audit / Peppol | B2G mandatory; B2B voluntary | UC2, UC3 |
| **Canada** | No federal e-invoicing mandate | N/A | GST and HST invoicing rules apply; no electronic mandate | UC2, UC3 |
| **USA** | No federal e-invoicing mandate | N/A | Sales tax varies by state; no federal e-invoicing; state-level nexus rules apply | UC2, UC3 |
| **Russia** | Roaming EDI operators | Post-audit and EDI | e-Invoicing widespread via EDI operators (Kontur, SBIS, etc.); specific format requirements | UC2, UC3 |
| **China** | Fapiao system (Golden Tax) | Clearance | Fully digital VAT fapiao system; electronic fapiao mandatory for most transactions | UC2, UC3 |

### 4.2 Tax Calculation Complexity

Beyond e-invoicing format and submission, Meteridian must support:

- **Multi-rate VAT and GST**: Most jurisdictions have standard, reduced, zero, and
  exempt rates. The EU alone has 27 Member States with different standard rates
  (17-27%) and varying reduced rates for different product and service categories.
- **Reverse charge mechanisms**: B2B cross-border services within the EU, and
  various domestic reverse charge scenarios (construction services in Germany,
  IT services in India under certain conditions).
- **Withholding tax**: Common in India (TDS on payments to non-residents),
  Brazil, and many developing economies. The payer must deduct tax before
  payment.
- **Place of supply rules**: Digital services taxation depends on where the
  customer is located, not where the supplier is. The EU's ViDA OSS and IOSS
  framework, India's IGST for inter-state supplies, and OECD's Pillar One
  framework all address this.
- **Tax on digital services**: Several jurisdictions impose specific digital
  services taxes (DSTs) — the UK, France, Italy, India (equalisation levy,
  subsequently amended), Turkey, Kenya, and others.
- **Transfer pricing**: For Use Case 1 in multi-national enterprises, internal
  cost allocations that cross legal entity boundaries may trigger transfer
  pricing documentation requirements under OECD guidelines and local
  implementations.

### 4.3 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Pluggable tax engine interface | MUST | Core architecture must support external tax calculation services (Avalara, Vertex, TaxJar) and allow custom implementations |
| E-invoicing output format framework | MUST | Block-based output stage must generate UBL 2.1 XML, Factur-X PDF/A-3, Peppol BIS, CFDI XML, NF-e XML, and other formats |
| Tax authority API integration points | MUST | Clearance-model jurisdictions require real-time API integration with government systems before invoices are legally valid |
| Cryptographic signing capability | MUST | Multiple jurisdictions require digital signatures (ICP-Brasil, ZATCA CSID, qualified electronic signatures under eIDAS) |
| Multi-currency support with exchange rate tracking | MUST | Tax calculations must use the exchange rate specified by each jurisdiction's tax authority on the date of supply |
| Audit trail of tax calculations | MUST | Every tax determination must be logged with the inputs, rules applied, and result — this is required for audit defense |
| QR code generation | SHOULD | Required by ZATCA, India GST, and increasingly other jurisdictions |

---

## 5. Data Protection and Privacy

Data protection is mandatory in virtually every jurisdiction where Meteridian
operates. Even in the internal FinOps use case, billing data often contains
personal identifiers (employee names, email addresses, team assignments).

### 5.1 Major Data Protection Frameworks

#### European Union — GDPR (Regulation (EU) 2016/679)

The gold standard of data protection law. Applies to any processing of personal
data of EU residents, regardless of where the processor is located.

| Requirement | Tier | Use Cases |
|---|---|---|
| Lawful basis for processing (consent, legitimate interest, contract, legal obligation) | MUST | UC1, UC2, UC3 |
| Data minimization — collect only what is necessary | MUST | UC1, UC2, UC3 |
| Purpose limitation — use data only for stated purposes | MUST | UC1, UC2, UC3 |
| Right to erasure (Art. 17) with exceptions for legal obligations (tax records) | MUST | UC2, UC3 |
| Data Protection Impact Assessment (DPIA) for high-risk processing | MUST | UC2, UC3 |
| Data Processing Agreement (DPA) with all processors | MUST | UC2, UC3 |
| 72-hour breach notification to supervisory authority | MUST | UC1, UC2, UC3 |
| Cross-border transfer mechanisms (SCCs, adequacy decisions, BCRs) | MUST | UC2, UC3 |
| Data Protection Officer appointment (for large-scale processing) | SHOULD | UC2, UC3 |
| Records of processing activities (Art. 30) | MUST | UC1, UC2, UC3 |

**Fines:** Up to €20M or 4% of global annual turnover, whichever is higher.

#### United States — Patchwork of State and Federal Laws

The US has no single comprehensive federal data protection law as of June 2026.
Key frameworks:

| Law | Scope | Tier |
|---|---|---|
| **CCPA and CPRA** (California Consumer Privacy Act and California Privacy Rights Act) | California residents; businesses meeting revenue or data thresholds | MUST (if serving CA residents) |
| **State privacy laws** (Virginia VCDPA, Colorado CPA, Connecticut CTDPA, Utah UCPA, Texas TDPSA, Oregon, Montana, Iowa, Delaware, New Hampshire, New Jersey, Tennessee, Minnesota, Maryland, Nebraska, etc.) | Varies by state; expanding rapidly | MUST (per state) |
| **HIPAA** (Health Insurance Portability and Accountability Act) | Protected health information | MUST (UC3 healthcare) |
| **COPPA** (Children's Online Privacy Protection Act) | Children under 13 | MUST (if applicable) |
| **GLBA** (Gramm-Leach-Bliley Act) | Financial institutions | MUST (UC3 financial services) |
| **FTC Act Section 5** | Unfair or deceptive practices | MUST |
| **Proposed federal privacy legislation** (American Data Privacy and Protection Act and similar) | Federal-level comprehensive law | COULD (monitor) |

#### Brazil — LGPD (Lei Geral de Proteção de Dados, Law No. 13,709/2018)

Modeled heavily on GDPR. Enforced by the ANPD (Autoridade Nacional de Proteção
de Dados). Applies to processing of personal data in Brazil or of Brazilian
residents.

| Requirement | Tier |
|---|---|
| Lawful basis for processing (10 legal bases, including consent and legitimate interest) | MUST |
| Data Protection Officer (Encarregado) appointment | MUST |
| Cross-border transfer mechanisms (adequacy, SCCs, BCRs, or specific consent) | MUST |
| Data subject rights (access, correction, deletion, portability) | MUST |
| Breach notification to ANPD and data subjects within "reasonable time" | MUST |

**Fines:** Up to 2% of revenue in Brazil, capped at R$50M per infraction.

#### India — DPDP Act (Digital Personal Data Protection Act, 2023)

The DPDP Act was enacted in 2023 with Rules notified on November 13, 2025.
Implementation follows an 18-month phased schedule:

| Milestone | Date | What becomes enforceable |
|---|---|---|
| Board establishment + procedural rules | November 13, 2025 | Data Protection Board operational |
| Consent Manager framework | November 13, 2026 | Registration and obligations of Consent Managers |
| Full compliance obligations | May 13, 2027 | Notice, security safeguards, breach notification, SDF duties |

| Requirement | Tier |
|---|---|
| Consent-based processing with clear notice | MUST (from May 2027) |
| Data localization for Significant Data Fiduciaries (specific categories TBD) | MUST (once notified) |
| Cross-border transfers restricted to notified "allowed" jurisdictions | MUST (once notified) |
| Data breach notification to Board and data principals | MUST (from May 2027) |
| Independent data audits for Significant Data Fiduciaries | MUST (from 2027) |

**Fines:** Up to ₹250 crore (~US$30M) per violation.

#### China — PIPL (Personal Information Protection Law, effective Nov 2021)

The PIPL imposes strict requirements on cross-border data transfers, with three
lawful transfer mechanisms as of January 2026:

1. **CAC Security Assessment** — mandatory for Critical Information
   Infrastructure Operators, processors of >1M individuals' data, or cumulative
   transfers of >100K individuals' PI (or >10K sensitive PI)
2. **Standard Contractual Clauses (SCC) filing** — file with provincial
   cyberspace administration
3. **Personal Information Protection Certification** — new route effective
   January 1, 2026; 3-year certificate validity; suitable for mid-scale
   exporters (100K-1M individuals' non-sensitive PI)

| Requirement | Tier |
|---|---|
| Personal Information Protection Impact Assessment (PIPIA) before any cross-border transfer | MUST |
| Separate consent for cross-border transfers | MUST |
| Data localization for CIIOs and "important data" | MUST |
| Appointment of a dedicated person or representative for PI protection | MUST |
| Personal information handling rules for automated decision-making | MUST |

**Fines:** Up to ¥50M or 5% of prior-year revenue.

#### Japan — APPI (Act on Protection of Personal Information)

Amended significantly in 2022. Enforced by the Personal Information Protection
Commission (PPC).

| Requirement | Tier |
|---|---|
| Cross-border transfer requires consent or equivalent protection assessment | MUST |
| Breach notification to PPC and data subjects for qualifying incidents | MUST |
| Records of provision of personal data to third parties | MUST |
| Restrictions on processing of "special care-required" personal information | MUST |

#### Other Key Jurisdictions

| Jurisdiction | Framework | Key Requirements |
|---|---|---|
| **UK** | UK GDPR + Data Protection Act 2018 | Post-Brexit divergence; adequacy decision from EU until June 2025 (extended); independent framework |
| **Canada** | PIPEDA + provincial laws (Quebec Law 25, Alberta PIPA, BC PIPA) | Quebec Law 25 full enforcement 2024; PIPEDA reform pending |
| **Australia** | Privacy Act 1988 (reformed 2024) | Strengthened breach notification; cross-border disclosure rules; proposed Children's Privacy Code |
| **South Korea** | PIPA (Personal Information Protection Act) | One of the strictest; EU adequacy decision; strict consent requirements |
| **Russia** | Federal Law No. 152-FZ on Personal Data | Strict data localization (databases containing Russian citizens' PI must be in Russia); Roskomnadzor enforcement |
| **Middle East** | UAE PDPL, Saudi PDPL, Bahrain PDPL, Qatar PDPL | Rapidly evolving; varying maturity; data localization requirements common |
| **Africa** | South Africa POPIA, Nigeria NDPR and NDPA, Kenya DPA, Egypt (pending) | Fragmented; POPIA fully enforced; Nigeria transitioning from NDPR to NDPA |
| **Southeast Asia** | Singapore PDPA, Thailand PDPA, Philippines DPA, Indonesia PDP Law (2022), Vietnam PDPD | Indonesia PDP Law enforcement from Oct 2024; Vietnam decree-based framework |

### 5.2 Data Residency Requirements Summary

Data residency — the legal requirement that certain data must be stored and/or
processed within specific geographic boundaries — directly affects Meteridian's
deployment architecture.

| Jurisdiction | Requirement | Strictness |
|---|---|---|
| **Russia** | PI databases must be stored in Russia | Strict localization |
| **China** | CIIOs and "important data" must stay in-country; PI transfers require assessment | Conditional localization |
| **India** | Specific categories for SDFs (pending notification); previously mirroring localization for payment data (RBI) | Conditional localization |
| **Indonesia** | Strategic electronic systems must keep data in-country (Government Regulation 71/2019 and subsequent) | Conditional localization |
| **Vietnam** | Local storage copy requirement for certain data categories | Mirroring requirement |
| **Turkey** | Explicit consent required for cross-border transfers; limited exceptions | Consent-based |
| **EU** | No intra-EU restriction; cross-border requires adequacy, SCCs, or BCRs | Transfer mechanism required |
| **Saudi Arabia** | Government data must remain in-kingdom; private sector varies | Sector-dependent |
| **Nigeria** | Strategic data must be stored locally | Conditional localization |
| **Australia** | No strict localization but cross-border disclosure rules apply | Disclosure-based |

### 5.3 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Configurable data residency per tenant | MUST (UC3), SHOULD (UC2) | TimescaleDB and event store must support geographic partitioning; deployment topology must allow per-tenant data region selection |
| Data subject rights API and workflows | MUST (UC2, UC3) | Erasure, access, portability endpoints; conflict resolution with tax record retention obligations |
| Encryption at rest and in transit | MUST | AES-256 at rest, TLS 1.3 in transit as baseline |
| Audit log of all personal data access | MUST (UC2, UC3) | Immutable access logs for compliance audits |
| Data Processing Agreement templates | SHOULD | Provide standard DPA templates for Meteridian-as-processor scenarios |
| Privacy by design documentation | SHOULD | DPIA templates and records of processing activities |
| Cross-border transfer mechanism support | MUST (UC2, UC3) | SCCs, adequacy assessments, certification support as configurable metadata |

---

## 6. Financial Regulations

When Meteridian facilitates payments, extends credit, or manages prepaid
balances, it may trigger financial regulation obligations.

### 6.1 Payment Services

#### EU — PSD2/PSD3 and PSR

The PSD3 package (Third Payment Services Directive + Payment Services
Regulation) reached political agreement in November 2025. COREPER endorsed the
final texts on April 22, 2026. The ECON Committee approved on May 5, 2026.
Plenary adoption expected June 2026, with application ~21 months after
Official Journal publication (targeting early 2028).

**Impact on Meteridian:**

| Scenario | Regulatory trigger | Tier |
|---|---|---|
| Meteridian holds customer funds (prepaid balances, credit grants) | May constitute e-money issuance; PSD3 merges e-money and payment institution licensing | MUST assess |
| Meteridian routes payments between parties (marketplace model) | Payment service activity; requires licensing or exemption | MUST assess |
| Meteridian only calculates amounts and generates invoices (no fund flows) | Generally exempt — pure invoicing and billing software | No licensing needed |
| Commercial agent exemption (billing on behalf of a single principal) | PSD3/PSR tightens the commercial agent exemption | MUST assess |

**Key rule:** If Meteridian's credit and prepaid balance system (METR-0004) allows
customers to hold monetary value that can be used to pay for services, this
could constitute e-money under EU law. The architectural decision about whether
balances are "monetary" or "unit-based" (METR-0005) has direct regulatory
consequences.

#### USA — State Money Transmitter Licensing

Each US state has its own money transmitter licensing regime. If Meteridian
holds, controls, or transmits funds, it may need licenses in each state where
it operates. The Uniform Money Services Act provides a framework, but
adoption varies.

#### Other Jurisdictions

- **UK:** PSRs 2017 (implementing PSD2); FCA-regulated
- **India:** RBI payment aggregator guidelines (PA-PG framework)
- **Brazil:** Central Bank payment institution regulations (PIX, instant
  payments)
- **Singapore:** Payment Services Act 2019 (MAS licensing)
- **Japan:** Banking Act, Payment Services Act
- **Australia:** AFSL requirements for certain payment services

### 6.2 Anti-Money Laundering (AML) and Know Your Customer (KYC)

If Meteridian processes payments or manages financial accounts, AML and KYC
obligations may apply:

| Regulation | Jurisdiction | Tier |
|---|---|---|
| **AMLD6** (6th Anti-Money Laundering Directive) + AMLR (AML Regulation) | EU | MUST (if payment services) |
| **Bank Secrecy Act (BSA)** + FinCEN regulations | USA | MUST (if money transmission) |
| **PMLA** (Prevention of Money Laundering Act) | India | MUST (if payment services) |
| **FATF Recommendations** | Global | SHOULD (international standard) |
| **UK MLRs** (Money Laundering Regulations 2017) | UK | MUST (if payment services) |

**Meteridian's safest position:** Design the platform so that Meteridian
calculates and records billing events but does not hold, transmit, or control
customer funds. Payment collection should be handled by licensed payment
processors (Stripe, Adyen, etc.) through the pluggable payment provider
architecture (ADR-0010, METR-0002 §10). This avoids triggering financial
regulation in most jurisdictions.

### 6.3 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Clear architectural boundary between billing calculation and payment processing | MUST | Pluggable payment providers (ADR-0010) must handle all fund flows |
| Credit and prepaid balance design must avoid creating "e-money" | MUST assess | METR-0004 balance model needs legal review per target jurisdiction |
| KYC data collection capability (if needed) | SHOULD (UC2, UC3) | Customer onboarding workflows may need identity verification |
| Transaction monitoring hooks | SHOULD (UC2, UC3) | Event stream should support AML monitoring integration |
| Sanctions screening integration point | SHOULD (UC2, UC3) | Block architecture should allow sanctions-list checks |

---

## 7. Contract Law and Consumer Protection

Multi-year billing contracts are subject to contract law in each jurisdiction
where the contract is formed or performed.

### 7.1 Multi-Year Contract Enforceability

| Jurisdiction | Key Rules | Tier |
|---|---|---|
| **USA** | UCC Article 2 (goods); common law (services); statute of frauds for contracts >1 year (must be in writing); varying state auto-renewal laws (NY GBL §527, CA Business & Professions Code §17600 et seq., IL 815 ILCS 601/) | MUST |
| **EU** | Unfair Contract Terms Directive (93/13/EEC) for B2C; national implementations vary for B2B; EU Digital Content Directive 2019/770 | MUST |
| **Germany** | BGB §§305-310 (Allgemeine Geschäftsbedingungen — standard terms control); strict fairness review even in B2B; auto-renewal tacit extension rules (BGB §309 Nr. 9) | MUST |
| **France** | Code de commerce and Code civil; Loi Châtel (auto-renewal notification requirements); unfair terms control (Code de la consommation) | MUST |
| **UK** | Consumer Rights Act 2015 (fairness); Unfair Contract Terms Act 1977 (B2B); common law freedom of contract | MUST |
| **India** | Indian Contract Act 1872; Information Technology Act 2000 (electronic contracts); Consumer Protection Act 2019 (extends to B2B in some interpretations) | MUST |
| **Brazil** | Civil Code; Consumer Protection Code (CDC) applies broadly, even to some B2B relationships where there is a "vulnerability" imbalance | MUST |
| **Japan** | Civil Code; Consumer Contract Act; Act on Specified Commercial Transactions | MUST |
| **Australia** | Australian Consumer Law (ACL); unfair contract terms protections extended to small business B2B contracts (threshold raised in 2023) | MUST |
| **Canada** | Provincial consumer protection acts; Internet agreements regulations (Ontario, BC, Alberta) | MUST |

### 7.2 Auto-Renewal and Cancellation Rights

Many jurisdictions have specific rules about automatic contract renewal that
directly affect subscription and multi-year billing:

| Jurisdiction | Key Requirements |
|---|---|
| **EU** | Cooling-off period for distance/online contracts (14 days for B2C under Consumer Rights Directive 2011/83/EU); notification before auto-renewal |
| **USA (varies by state)** | California (automatic renewal must be clearly disclosed, easy cancellation; ARA — Automatic Renewal Act); New York (GBL §527 — 15-day advance notice for service contracts); Illinois (ARSA — similar to CA); FTC "click-to-cancel" rule (effective 2025) requiring cancellation to be as easy as sign-up |
| **Germany** | BGB §309 Nr. 9: auto-renewal clauses in standard terms limited to 1 year initial term, 1 month renewal extension, 3 months' notice; reform effective March 2022 allows termination via "Kündigungsbutton" (cancellation button) |
| **France** | Loi Châtel: written notice to consumer at earliest 3 months, latest 1 month before auto-renewal deadline; failure to notify allows immediate cancellation |
| **UK** | Consumer Contracts Regulations 2013 (14-day cooling-off for distance sales); CMA guidance on subscription traps |
| **Australia** | ACCC enforcement against unfair auto-renewal; no specific statute but ACL unfair terms applies |

### 7.3 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Contract metadata model (term, auto-renewal, notice period, cooling-off) | MUST (UC2) | Product catalog (METR-0003) must capture jurisdiction-specific contract parameters |
| Auto-renewal notification engine | MUST (UC2) | Configurable advance notification per jurisdiction |
| Cancellation workflow with easy-cancel compliance | MUST (UC2 in B2C/prosumer markets) | FTC click-to-cancel, German Kündigungsbutton |
| Cooling-off period enforcement | MUST (UC2, B2C) | Automatic refund/cancellation within statutory period |
| Contract template library with jurisdiction-specific terms | SHOULD (UC2) | Pre-built contract templates for major jurisdictions |

---

## 8. Audit, Record Retention, and Reporting

### 8.1 Record Retention Requirements

Billing records, tax documents, and financial transaction logs must be retained
for minimum periods that vary by jurisdiction:

| Jurisdiction | Minimum Retention | What Must Be Retained | Regulation |
|---|---|---|---|
| **USA (SOX)** | 7 years | Financial records, audit workpapers, communications | Sarbanes-Oxley Act §802 |
| **USA (IRS)** | 3-7 years | Tax returns, supporting documentation | IRC §6501 et seq. |
| **EU (ViDA/VAT)** | Typically 10 years (varies by Member State) | VAT invoices, supporting documents | VAT Directive Art. 247 |
| **Germany** | 10 years (invoices, financial records); 6 years (business correspondence) | All tax-relevant documents | AO §147 (Abgabenordnung) |
| **France** | 10 years (accounting records); 6 years (tax documents) | All commercial and tax documents | Code de commerce L.123-22; LPF L.102B |
| **UK** | 6 years (general); 20 years (HMRC in certain cases) | VAT records, invoices | VAT Regulations 1995 |
| **India** | 8 years (GST); 8 years (Income Tax) | GST returns, invoices, books of account | CGST Act §36; Income Tax Act §149 |
| **Brazil** | 5 years (tax documents); 10 years (civil claims) | NF-e XML, SEFAZ-authorized documents | CTN Art. 173-174 |
| **Saudi Arabia** | 6 years | VAT invoices, records | ZATCA VAT regulations |
| **China** | 10 years (accounting records); 30 years (permanent records) | Accounting vouchers, books, statements | PRC Accounting Law; Archives Law |
| **Japan** | 7 years (tax); 10 years (corporate records) | Tax invoices, accounting books | Corporation Tax Act; Companies Act |
| **Australia** | 5 years (tax records); 7 years (corporate records) | Tax records, financial statements | Income Tax Assessment Act; Corporations Act |
| **Russia** | 5 years (accounting); 4 years (tax) | Primary accounting documents, tax declarations | Federal Law on Accounting; Tax Code |
| **Canada** | 6 years (from end of tax year) | Tax records, supporting documents | Income Tax Act §230 |

### 8.2 Audit Trail Requirements

Beyond retention, many frameworks require specific audit trail capabilities:

| Requirement | Jurisdictions | Tier |
|---|---|---|
| Immutable audit trail of all billing events | All (implicit in tax law) | MUST |
| Timestamp with authoritative time source (NTP or PTP) | EU (eIDAS for qualified timestamps), India (GST) | MUST |
| Sequential invoice numbering (no gaps) | EU (VAT Directive), India (GST), Brazil (NF-e), Saudi Arabia (ZATCA) | MUST |
| Tamper-evident storage of tax documents | Germany (GoBD — Grundsätze ordnungsmäßiger Buchführung und Dokumentation), France, Italy | MUST |
| Change tracking with user attribution | SOX (USA), SOC 2 (global standard) | MUST (UC3), SHOULD (UC2) |
| Export capability in machine-readable format | EU Data Act, Germany (GDPdU and GoBD), France (FEC — Fichier des Écritures Comptables) | MUST |

### 8.3 Germany's GoBD: A Particularly Stringent Example

Germany's GoBD (Grundsätze zur ordnungsmäßigen Führung und Aufbewahrung von
Büchern, Aufzeichnungen und Unterlagen in elektronischer Form sowie zum
Datenzugriff) imposes detailed requirements on electronic bookkeeping systems:

- All booking entries must be traceable, complete, correct, timely, and orderly
- Electronic documents must be stored in their original format (no conversion)
- A procedural documentation (Verfahrensdokumentation) must describe how the
  system works
- Data must be made available to tax auditors in machine-readable format
  (Z1 — direct data access, Z2 — indirect access via queries, Z3 — data export)

This is relevant because cloud billing platforms that serve German customers
must be able to produce GoBD-compliant exports.

### 8.4 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Immutable event store with configurable retention periods | MUST | TimescaleDB event store (ADR-0001) must support per-tenant retention policies, with minimum retention meeting the strictest applicable jurisdiction |
| Sequential invoice numbering with gap detection | MUST | Numbering service must be centralized, tenant-isolated, and gap-free |
| Machine-readable export in standard formats | MUST | FEC (France), GoBD/GDPdU (Germany), SAF-T (OECD standard, used by Portugal, Norway, Poland, Lithuania, etc.), AuditFile (Netherlands) |
| Tamper-evident storage | MUST | Cryptographic hash chains or similar for billing records |
| Procedural documentation generator | SHOULD | GoBD Verfahrensdokumentation, SOC 2 system descriptions |
| Data retention policy engine | MUST | Per-tenant, per-jurisdiction configurable retention with automated hold and purge |
| Conflict resolution: retention vs. erasure | MUST | When GDPR right-to-erasure conflicts with tax retention, tax retention wins (GDPR Art. 17(3)(b)) — must document this logic |

---

## 9. Industry-Specific Regulations

Certain Meteridian deployment scenarios trigger sector-specific regulations
beyond the general frameworks above.

### 9.1 Government and Public Sector

| Regulation | Jurisdiction | Requirements | Tier (UC3) |
|---|---|---|---|
| **FedRAMP** | USA | Security certification for cloud services used by federal agencies; now uses "FedRAMP Certification" terminology (2026); Class A-D; Rev5 or 20x pathway; machine-readable JSON artifacts; independent assessment | MUST (US gov) |
| **StateRAMP and GovRAMP** | USA (states) | State-level equivalent of FedRAMP; accepted as external framework for FedRAMP 20x Level 1 | SHOULD (US state gov) |
| **EUCS** (EU Cybersecurity Certification Scheme) | EU | Basic, Substantial, and High assurance levels; sovereignty requirements still evolving; CADA (Cloud and AI Development Act, proposed June 2026) may reintroduce sovereignty tiers for public procurement | SHOULD (EU gov) |
| **SecNumCloud** | France | ANSSI certification; strict sovereignty (EU HQ, EU data, immunity from extraterritorial laws); Level 4 of proposed CADA framework mirrors this | MUST (French gov) |
| **C5** (Cloud Computing Compliance Criteria Catalogue) | Germany | BSI certification; required for government cloud in Germany | MUST (German gov) |
| **IRAP** | Australia | Information Security Registered Assessors Program; required for government cloud | MUST (Australian gov) |
| **CCCS and ITSG-33** | Canada | Canadian Centre for Cyber Security guidance; Protected B requirement | MUST (Canadian gov) |
| **MeitY empanelment** | India | Government cloud empanelment for government workloads | MUST (Indian gov) |
| **GovCloud (various)** | Multiple | Government procurement often requires dedicated infrastructure, clearance for personnel, and specific data handling | MUST (government) |

### 9.2 Telecommunications

Telecom billing has its own regulatory framework in most jurisdictions:

| Requirement | Jurisdictions | Tier |
|---|---|---|
| CDR (Call Detail Record) retention and format | ITU standards, national telecom regulators | MUST (telecom UC3) |
| Billing accuracy requirements (6-sigma, per ETSI/3GPP) | ETSI standards, national regulators | MUST (telecom UC3) |
| Interconnect settlement standards | ITU-T, national regulators | MUST (telecom UC3) |
| Number portability support | Most jurisdictions | MUST (telecom UC3) |
| Emergency services funding surcharge calculation | USA (FCC), EU (BEREC) | MUST (telecom UC3) |

### 9.3 Healthcare

| Regulation | Jurisdiction | Requirements | Tier |
|---|---|---|---|
| **HIPAA** | USA | PHI (Protected Health Information) in billing data; minimum necessary standard; Business Associate Agreements; breach notification | MUST (healthcare UC3) |
| **HITECH Act** | USA | Strengthened HIPAA enforcement; EHR incentive programs | MUST (healthcare UC3) |
| **EU MDR and IVDR** | EU | Medical device regulation — less directly relevant but may apply if Meteridian processes clinical data for billing | Assess |
| **NHS DTAC** | UK | Digital Technology Assessment Criteria for NHS digital services | MUST (UK healthcare UC3) |

### 9.4 Financial Services

| Regulation | Jurisdiction | Requirements | Tier |
|---|---|---|---|
| **DORA** (Digital Operational Resilience Act) | EU | ICT risk management for financial entities and their critical third-party ICT providers; effective January 2025 | MUST (if critical provider to EU financial institutions) |
| **SOX** (Sarbanes-Oxley) | USA | Internal controls over financial reporting; audit trail requirements | MUST (publicly traded US companies or their billing platforms) |
| **MiFID II** | EU | Transaction reporting for financial instruments | Assess (if applicable) |
| **RBI IT Framework** | India | IT governance for regulated financial institutions | MUST (Indian financial services UC3) |

### 9.5 EU Digital Markets Act (DMA)

The DMA (Regulation (EU) 2022/1925, effective May 2023, obligations from March
2024) designates "gatekeeper" platforms and imposes interoperability and
non-discrimination obligations. While Meteridian itself is unlikely to be
designated as a gatekeeper, DMA obligations on designated gatekeepers (Apple,
Google, Amazon, Meta, Microsoft, ByteDance) may affect how Meteridian integrates
with their platforms for billing data access.

### 9.6 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Healthcare data isolation capability (HIPAA) | MUST (UC3 healthcare) | Separate encryption keys, access controls, BAA support |
| Telecom CDR processing blocks | COULD (UC3 telecom) | Pre-built blocks for CDR ingestion, rating, and billing |
| Financial reporting export (SOX) | SHOULD (UC2, UC3) | Export formats for financial audit tools |
| DORA ICT risk management documentation | SHOULD (UC3 financial) | Incident reporting, resilience testing support |
| Government procurement documentation | MUST (UC3 government) | Security plans, authorization packages, certification artifacts |

---

## 10. Standards and Certifications

Standards and certifications are not laws — they are voluntary frameworks that
become de facto mandatory through contractual requirements (enterprise
procurement), regulatory references (FedRAMP references NIST 800-53), or
industry expectations.

### 10.1 Security and Operations Standards

| Standard | What It Covers | Tier | Relevance |
|---|---|---|---|
| **SOC 2 Type II** | Trust service criteria: security, availability, processing integrity, confidentiality, privacy | MUST (UC2 enterprise, UC3) | Most requested certification in enterprise B2B SaaS procurement |
| **ISO 27001** | Information security management system (ISMS) | MUST (UC2 enterprise, UC3) | International baseline; required by many government procurement frameworks |
| **ISO 27017** | Cloud-specific security controls (extends 27001) | SHOULD (UC3) | Cloud-specific extension; increasingly requested |
| **ISO 27018** | PII protection in public clouds | SHOULD (UC2, UC3) | Privacy-focused extension of 27001 |
| **ISO 27701** | Privacy information management system (extends 27001) | SHOULD (UC2, UC3) | GDPR-aligned privacy management |
| **ISAE 3402 / SSAE 18** | Service organization controls (predecessor to SOC reports) | SHOULD (UC2, UC3) | Used internationally where SOC 2 is less established |
| **CSA STAR** (Cloud Security Alliance) | Cloud security maturity | COULD (UC3) | Self-assessment (Level 1) is free; Level 2 involves third-party audit |
| **NIST SP 800-53** | Security and privacy controls for federal information systems | MUST (UC3 US gov) | Baseline for FedRAMP; Rev5 is current |
| **NIST Cybersecurity Framework (CSF)** | Identify, Protect, Detect, Respond, Recover | SHOULD (all) | Widely adopted voluntary framework |

### 10.2 Payment and Financial Standards

| Standard | What It Covers | Tier | Relevance |
|---|---|---|---|
| **PCI DSS** (Payment Card Industry Data Security Standard) | Cardholder data protection | MUST (if storing/processing payment card data) | Typically avoided by using tokenization via payment processors |
| **PA-DSS and PCI SSF** | Payment application security | Assess | If Meteridian's payment integration touches card data |
| **ISO 20022** | Financial messaging standard | COULD | Increasingly mandated for payment messages (SWIFT, SEPA) |

### 10.3 Quality and Process Standards

| Standard | What It Covers | Tier | Relevance |
|---|---|---|---|
| **ISO 9001** | Quality management system | COULD | General quality; less common in SaaS |
| **CMMI** | Process maturity | COULD | Government contractors, defense |
| **ITIL 4** | IT service management | COULD | Enterprise operations |

### 10.4 Billing-Specific Standards

| Standard | What It Covers | Tier | Relevance |
|---|---|---|---|
| **TM Forum Open APIs** | Telecommunications management APIs (TMF620 Product Catalog, TMF622 Product Ordering, TMF632 Party Management, TMF678 Customer Bill Management) | SHOULD (UC3 telecom) | Standard API contracts for telecom billing |
| **ETSI ES 201 915** | OSA and Parlay — service-independent API standards | COULD (UC3 telecom) | Legacy but still referenced |
| **3GPP TS 32.2xx series** | Charging and billing for mobile networks | MUST (UC3 mobile telecom) | Defines CDR formats, online and offline charging |
| **OASIS UBL 2.1** | Universal Business Language for e-invoicing | MUST (UC2, UC3) | Required by ZATCA, Peppol, and many e-invoicing mandates |
| **EN 16931** | European standard for electronic invoicing | MUST (EU UC2, UC3) | Core invoice model for EU e-invoicing |
| **Peppol BIS Billing 3.0** | Cross-border e-invoicing profile | SHOULD (EU UC2, UC3) | Based on EN 16931; used in EU, Australia, Singapore, New Zealand |

### 10.5 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| SOC 2 Type II readiness from day one | MUST (UC2, UC3) | Design controls, logging, access management with SOC 2 criteria in mind |
| ISO 27001 certifiable architecture | SHOULD | Document ISMS, risk assessments, control implementations |
| PCI DSS avoidance through payment tokenization | MUST | Never store raw card data; use payment processor tokens |
| TM Forum API compatibility (TMF620, TMF678) | SHOULD (UC3 telecom) | Product catalog and billing APIs should map to TM Forum Open APIs |
| UBL 2.1 and EN 16931 as first-class output formats | MUST | Core invoice data model must be EN 16931-compatible |

---

## 11. Sovereign Cloud Requirements

Sovereign cloud is not a single regulatory framework but a convergence of data
residency, operational sovereignty, security certification, and government
procurement requirements that varies significantly by jurisdiction.

### 11.1 Sovereignty Dimensions

| Dimension | Definition | Examples |
|---|---|---|
| **Data sovereignty** | Data is stored and processed within national/regional borders | Russia (strict), China (CIIO data), EU (GDPR + national requirements) |
| **Operational sovereignty** | Operations (support, administration, monitoring) are performed by nationals/residents with security clearances | France (SecNumCloud), Germany (C5 + operational requirements), US (FedRAMP + ITAR) |
| **Legal sovereignty / Immunity from extraterritorial laws** | The service is not subject to foreign government data access demands | EU vs. US CLOUD Act conflict; France SecNumCloud Level 4; proposed CADA Level 4 |
| **Supply chain sovereignty** | Hardware, software, and firmware are sourced from trusted (typically domestic) suppliers | China (信创/Xinchuang — innovation and creation initiative), Russia (import substitution) |

### 11.2 Government Procurement Rules

| Jurisdiction | Key Requirements | Tier (UC3) |
|---|---|---|
| **USA** | FAR/DFARS clauses; FedRAMP certification; ITAR for defense; CUI handling (NIST 800-171 / CMMC); Buy American Act | MUST |
| **EU** | Public procurement directives (2014/24/EU); proposed CADA sovereignty levels; EUCS certification; Member State-specific rules | MUST |
| **France** | SecNumCloud for sensitive public sector data; ANSSI qualification; French-law entity requirement | MUST |
| **Germany** | BSI C5 attestation; IT-Grundschutz; national security classification handling | MUST |
| **UK** | NCSC Cloud Security Principles; Crown Commercial Service G-Cloud framework; Cyber Essentials Plus | MUST |
| **India** | MeitY guidelines; Data localization for government data; empanelment requirements; STQC certification | MUST |
| **Australia** | ASD/IRAP assessment; Hosting Certification Framework (HCF); Certified, Strategic, or Assured levels | MUST |
| **Japan** | ISMAP (Information System Security Management and Assessment Program) | MUST |
| **South Korea** | CSAP (Cloud Security Assurance Program) administered by KISA | MUST |
| **Saudi Arabia** | NCA (National Cybersecurity Authority) cloud security requirements; data localization for government | MUST |
| **UAE** | IAS (Information Assurance Standards) from NESA/TDRA | MUST |
| **Brazil** | LGPD compliance; ICP-Brasil digital certificates; specific procurement rules for federal government IT | MUST |
| **Russia** | FSTEC certification; FSB requirements; import substitution mandates; domestic hosting | MUST |

### 11.3 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Multi-region, multi-tenant deployment with per-tenant region pinning | MUST (UC3) | Kubernetes-native deployment must support geographic constraints |
| Operational access controls with nationality/clearance filtering | MUST (UC3 government) | RBAC must support personnel clearance level attributes |
| Air-gapped deployment capability | MUST (UC3 defense/classified) | Must function without internet connectivity |
| Supply chain documentation (SBOM) | MUST (UC3) | Software Bill of Materials for all dependencies; SLSA provenance (already planned in ADR-0009) |
| Encryption key management with customer-controlled keys (BYOK/HYOK) | MUST (UC3) | Key management must support external HSMs and customer-owned keys |
| Isolated control plane per sovereignty boundary | SHOULD (UC3) | Management APIs should be deployable within the sovereignty boundary |

---

## 12. Open Source Licensing Considerations

Meteridian is licensed under Apache License 2.0. This interacts with regulated
billing in several ways.

### 12.1 Warranty Disclaimers and Liability

The Apache 2.0 license includes standard open-source warranty disclaimers:

> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND...

In a billing context, this creates tension:

| Scenario | Risk | Mitigation |
|---|---|---|
| Tax calculation error due to a bug | Financial liability for incorrect tax amounts; tax authority penalties | The platform operator (not the open-source project) is responsible; commercial support agreements should address SLAs |
| Invoice formatting error causes rejection by tax authority | Business disruption; inability to collect payment | Comprehensive validation against e-invoicing schemas before submission |
| Billing accuracy dispute in multi-year contract | Contractual liability | The MSP or neocloud operator bears liability, not the open-source project |

### 12.2 Regulatory Certification and Open Source

Some regulatory certifications (FedRAMP, SOC 2, ISO 27001) require a
responsible entity that can be audited. Open-source projects cannot hold
certifications — only organizations deploying the software can. This means:

- Meteridian the open-source project provides the _certifiable_ architecture
- Each deploying organization obtains its own certifications
- Commercial distributions (if any) could offer pre-certified packages

### 12.3 Export Control

Meteridian includes cryptographic functionality (TLS, encryption at rest,
digital signatures). Under US export control regulations (EAR — Export
Administration Regulations), open-source software that uses encryption is
generally eligible for the publicly available encryption exclusion (§742.15(b)),
but a TSU (Technology and Software Unrestricted) exception notification to BIS
may be required.

### 12.4 License Compatibility

Meteridian's block marketplace (METR-0002) will include third-party blocks
that may use different licenses. The marketplace must enforce:

- License compatibility with Apache 2.0 for core platform integration
- Clear attribution for blocks with attribution requirements
- GPL-compatibility assessment (GPL blocks cannot be tightly coupled with
  Apache-licensed code without the combined work being GPL)
- Commercial/proprietary blocks must use well-defined API boundaries

### 12.5 Meteridian Implications

| Requirement | Tier | Impact |
|---|---|---|
| Clear liability boundaries in documentation and DPA templates | MUST | Operator vs. project responsibility must be unambiguous |
| EAR TSU notification for encryption | SHOULD | File notification with BIS for the open-source distribution |
| License compatibility checker for marketplace blocks | MUST | Automated license scanning in the marketplace submission pipeline |
| SBOM generation | MUST (UC3) | CycloneDX or SPDX format; required by US Executive Order 14028 and EU CRA |

---

## 13. Competitive Landscape

This section summarizes how existing billing, metering, and FinOps platforms
approach legal and regulatory compliance. A detailed competitive analysis is
available at [`docs/competitive/legal-regulatory-compliance.md`](../../docs/competitive/legal-regulatory-compliance.md).

### 13.1 Market Compliance Hierarchy

The billing platform market exhibits a clear compliance maturity hierarchy that
reveals both the table stakes and the whitespace opportunity for Meteridian:

| Tier | Vendors | Typical Certifications |
|---|---|---|
| **Enterprise incumbents** | SAP BRIM, Oracle RMAB, Salesforce Revenue Cloud | FedRAMP, HIPAA, ISO 27001, SOC 1/2, PCI-DSS; sovereign cloud deployments |
| **Established SaaS billing** | Zuora, Chargebee, Stripe, Recurly | SOC 2 Type II, PCI-DSS Level 1; Zuora leads with ISO 27001/27701/27018 and HIPAA |
| **Usage-based billing startups** | Orb, Metronome, Amberflo, Togai, m3ter | SOC 2 Type II as baseline; no HIPAA, FedRAMP, or ISO 27001 |
| **Open-source billing** | Lago (AGPLv3), OpenMeter (Apache 2.0), Kill Bill (Apache 2.0) | Compliance is the deployer's responsibility; no vendor certifications for self-hosted |
| **Tax compliance engines** | Avalara, Vertex, Anrok | SOC 2, ISO 27001 (Avalara); 100+ country coverage, e-invoicing, nexus detection |
| **FinOps platforms** | IBM Apptio (FedRAMP), Vantage (SOC 1/2), OpenCost (none) | Apptio is the only FinOps platform with FedRAMP authorization |

### 13.2 Key Market Gaps Meteridian Can Address

1. **No open-source platform offers compliance-as-code.** All open-source billing
   platforms provide compliance *tooling* (audit logs, RBAC) but leave the
   compliance *program* entirely to the deployer. None provide automated
   compliance report generation, audit-ready data exports in regulatory-specific
   formats, or built-in GDPR data lifecycle workflows.

2. **FedRAMP is absent from all pure-play billing/metering platforms.** Only ERP
   vendors (SAP, Oracle, Salesforce) and IBM Apptio hold FedRAMP. This blocks
   the entire billing startup category from US government procurement.
   Meteridian's self-hostable architecture with NIST SP 800-53 controls mapped
   from day one could provide a path to FedRAMP-ready deployments.

3. **E-invoicing is universally outsourced.** Every billing platform delegates
   e-invoicing to Avalara or Vertex. No metering/rating engine natively generates
   jurisdiction-compliant e-invoices. Meteridian's block architecture (METR-0002)
   enables first-class e-invoicing output blocks without requiring a monolithic
   tax engine.

4. **Data sovereignty through self-hosting lacks enterprise tooling.** Self-hosted
   open-source options (Lago, Kill Bill, OpenMeter) give full data residency
   control but provide no automated compliance scanning, SOC 2/ISO 27001
   readiness tooling, or compliance dashboards.

### 13.3 Certification Targets for Meteridian

Based on competitor positioning and market requirements, Meteridian should
target the following certifications on its roadmap:

| Certification | Target | Rationale |
|---|---|---|
| **SOC 2 Type II readiness** | v1.0 (design controls) | Table stakes — every competitor from Orb to Zuora holds this. Design the architecture with Trust Services Criteria from day one. |
| **ISO 27001 certifiable architecture** | v1.x | Zuora holds ISO 27001/27701/27018; Chargebee holds 27001:2022. Having a certifiable architecture (even if each deployer obtains their own certification) differentiates vs. Orb/Metronome/m3ter. |
| **ISO 27701 (privacy management)** | v1.x | Only Zuora and IBM Turbonomic hold this among billing/FinOps competitors. GDPR-aligned privacy management is a strong differentiator. |
| **NIST SP 800-53 control mapping** | v1.x | Prerequisite for FedRAMP. No billing startup has done this. |
| **FedRAMP-ready architecture** | v2.0 | The deploying organization obtains the ATO, but Meteridian provides pre-built security documentation, continuous monitoring hooks, and machine-readable artifacts. |

### 13.4 Competitive Implications for Architecture

| Competitor Practice | Meteridian Response |
|---|---|
| Zuora and Chargebee outsource tax to Avalara and Vertex | Design pluggable tax engine interface (§4.3) with pre-built connectors; allow operators to also use native tax calculation blocks |
| Paddle handles compliance as Merchant of Record | Meteridian supports the operator-as-MoR pattern by providing the compliance infrastructure without assuming the MoR role |
| Kill Bill provides immutable append-only audit logs | Meteridian goes further with cryptographic hash chains (§8.4) for tamper-evident, verifiable audit trails |
| No OSS billing platform generates FOCUS-conformant output | Meteridian should produce FOCUS 1.4-conformant billing data natively for FinOps interoperability |
| SAP BRIM supports 1,000+ local versions | Meteridian's block marketplace (§16.5 / METR-0002) enables community-driven jurisdiction support without monolithic localization |

---

## 14. Jurisdiction-by-Jurisdiction Reference

### 14.1 United States

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | No federal e-invoicing; state sales tax nexus (South Dakota v. Wayfair); varying rates and rules across 50 states + territories; use tax obligations | — | MUST | MUST |
| **Data protection** | CCPA/CPRA + state patchwork; HIPAA (healthcare); GLBA (financial); FTC Act | SHOULD | MUST | MUST |
| **Financial** | State money transmitter licensing; BSA/AML; CFPB oversight | — | Assess | Assess |
| **Audit** | SOX (7 years); IRS (3-7 years); state requirements vary | — | MUST | MUST |
| **Government** | FedRAMP (now "FedRAMP Certification"); NIST 800-53/800-171; CMMC; ITAR; FAR/DFARS | — | — | MUST |
| **Standards** | SOC 2 Type II; PCI DSS (if handling card data) | COULD | MUST | MUST |

### 14.2 European Union

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | ViDA (mandatory e-invoicing from July 2030 for cross-border B2B); Member State domestic mandates accelerating; VAT reverse charge; EN 16931 format | — | MUST | MUST |
| **Data protection** | GDPR; cross-border transfers via SCCs/adequacy/BCRs | MUST | MUST | MUST |
| **Financial** | PSD3/PSR (adoption 2026, application ~early 2028); AMLD6/AMLR | — | Assess | Assess |
| **Cloud switching** | EU Data Act (applicable from Sep 2025); max 2-month notice; 30-day data porting; switching fees prohibited from Jan 2027 | — | MUST | MUST |
| **Government** | EUCS; proposed CADA (sovereignty tiers); NIS2 (critical entity obligations); CRA (Cyber Resilience Act — product security) | — | SHOULD | MUST |
| **Standards** | ISO 27001; ISAE 3402; SOC 2 (increasingly accepted) | COULD | SHOULD | MUST |

#### Germany (Additional)

| Area | Additional Requirements |
|---|---|
| **Tax** | XRechnung (B2G mandatory); B2B e-invoicing mandate 2027-2028; GoBD compliance for electronic bookkeeping; 10-year retention |
| **Government** | BSI C5; IT-Grundschutz; operational sovereignty requirements |
| **Audit** | GoBD Verfahrensdokumentation; FEC equivalent (GDPdU export) |

#### France (Additional)

| Area | Additional Requirements |
|---|---|
| **Tax** | Factur-X / Chorus Pro; B2B e-invoicing phased 2026-2028; FEC (Fichier des Écritures Comptables) export mandatory |
| **Government** | SecNumCloud (ANSSI); proposed CADA Level 4 mirrors SecNumCloud |
| **Contract** | Loi Châtel (auto-renewal notification) |

### 14.3 United Kingdom

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | Making Tax Digital (MTD) for VAT; no B2B e-invoicing mandate yet but under review; Peppol adoption growing | — | MUST | MUST |
| **Data protection** | UK GDPR + Data Protection Act 2018; ICO enforcement | MUST | MUST | MUST |
| **Government** | NCSC Cloud Security Principles; Cyber Essentials Plus; G-Cloud | — | — | MUST |
| **Contract** | Consumer Rights Act 2015; Unfair Contract Terms Act 1977 | — | MUST | MUST |

### 14.4 India

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | GST e-invoicing (IRP) mandatory for >₹5 crore turnover; 30-day upload for >₹10 crore; ₹2 crore threshold proposed; TDS/TCS; IGST for inter-state | — | MUST | MUST |
| **Data protection** | DPDP Act 2023 + Rules 2025; phased enforcement through May 2027; data localization for SDFs (pending notification) | SHOULD | MUST | MUST |
| **Financial** | RBI payment aggregator guidelines; FEMA for cross-border payments | — | Assess | Assess |
| **Government** | MeitY empanelment; STQC certification; data localization for government data | — | — | MUST |

### 14.5 China

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | Golden Tax System; electronic fapiao; VAT calculation | — | MUST | MUST |
| **Data protection** | PIPL; Data Security Law; Cybersecurity Law; strict cross-border transfer requirements (3 routes); data localization for CIIOs | MUST | MUST | MUST |
| **Government** | MLPS (Multi-Level Protection Scheme) for information systems; Xinchuang (domestic IT) preference | — | — | MUST |
| **Supply chain** | Domestic hardware/software preference for government and critical infrastructure | — | — | MUST |

### 14.6 Japan

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | Qualified Invoice System (October 2023); Qualified Invoice Issuer registration; consumption tax calculation | — | MUST | MUST |
| **Data protection** | APPI; cross-border transfer restrictions; PPC enforcement | MUST | MUST | MUST |
| **Government** | ISMAP certification for government cloud | — | — | MUST |
| **Contract** | Civil Code; Consumer Contract Act; Act on Specified Commercial Transactions | — | MUST | MUST |

### 14.7 Brazil

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | NF-e/NFS-e/CT-e clearance model (SEFAZ); IBS/CBS fields mandatory from Aug 2026; ICP-Brasil digital certificate; 5-year retention of SEFAZ-authorized XML | — | MUST | MUST |
| **Data protection** | LGPD; ANPD enforcement; DPO (Encarregado) appointment mandatory | SHOULD | MUST | MUST |
| **Contract** | Civil Code; Consumer Protection Code (CDC) — applies broadly including some B2B | — | MUST | MUST |
| **Government** | ICP-Brasil digital certificates; specific federal procurement rules | — | — | MUST |

### 14.8 Latin America (Other)

| Country | Key Requirements |
|---|---|
| **Mexico** | CFDI 4.0 (clearance via SAT/PAC); mandatory for all commercial transactions; RFC (tax ID) required; Carta Porte complement for transport |
| **Colombia** | Factura Electrónica (clearance via DIAN); mandatory since 2019; CUFE unique invoice code |
| **Chile** | DTE (Documento Tributario Electrónico) via SII; fully mandatory; electronic boleta |
| **Argentina** | Factura Electrónica via AFIP; mandatory for most; CAE/CAEA authorization codes |
| **Peru** | CPE (Comprobante de Pago Electrónico) via SUNAT; mandatory for most businesses |
| **Ecuador** | Comprobantes electrónicos via SRI; mandatory |
| **Costa Rica** | Factura electrónica via Ministerio de Hacienda; mandatory |

Latin America is the most advanced region globally for mandatory e-invoicing
clearance models. Any MSP or neocloud operating in this region must have robust
e-invoicing integration from day one.

### 14.9 Middle East

| Country | Key Requirements |
|---|---|
| **Saudi Arabia** | ZATCA Fatoora Phase 2 (clearance/reporting); UBL 2.1 XML; cryptographic stamp; Wave 24 deadline June 30, 2026 |
| **UAE** | No e-invoicing mandate yet (as of June 2026) but FTA has signaled plans; VAT at 5% |
| **Bahrain** | VAT at 10% (increased from 5% in 2022); no e-invoicing mandate yet |
| **Oman** | VAT at 5%; e-invoicing under consideration |
| **Qatar** | No VAT currently |
| **Egypt** | e-Invoice mandatory via ETA; phased rollout completed |
| **Jordan** | JoFotara e-invoicing system; phased implementation |

### 14.10 Southeast Asia

| Country | Key Requirements |
|---|---|
| **Singapore** | InvoiceNow (Peppol) for B2G; GST at 9%; PDPA data protection; Payment Services Act (MAS) |
| **Malaysia** | MyInvois e-invoicing via LHDN; fully mandatory mid-2025; SST (Sales and Service Tax) |
| **Indonesia** | e-Faktur for VAT invoices; PDP Law (2022) data protection; Government Regulation 71/2019 data localization |
| **Thailand** | e-Tax Invoice system; PDPA data protection; Revenue Department RD system |
| **Philippines** | CAS/EIS (Computerized Accounting System / Electronic Invoicing System) via BIR; DPA data protection |
| **Vietnam** | e-Invoice mandatory since July 2022; Cybersecurity Law data localization |

### 14.11 Africa

| Country | Key Requirements |
|---|---|
| **South Africa** | No e-invoicing mandate; VAT at 15%; POPIA data protection (fully enforced) |
| **Nigeria** | e-Invoicing pilot; NDPA data protection (transitioning from NDPR); TIN requirements |
| **Kenya** | TIMS (Tax Invoice Management System) via eTIMS; mandatory for VAT-registered; Data Protection Act 2019 |
| **Egypt** | e-Invoice mandatory (see Middle East section); Data Protection Law 2020 (limited enforcement) |
| **Rwanda** | EBM (Electronic Billing Machine) mandatory for VAT |
| **Tanzania** | EFD (Electronic Fiscal Device) / VFD (Virtual Fiscal Device) mandatory |

### 14.12 Australia / New Zealand

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | Peppol e-invoicing (B2G mandatory for Commonwealth; B2B voluntary); GST at 10% (AU), 15% (NZ) | — | MUST | MUST |
| **Data protection** | Privacy Act 1988 (AU, reformed 2024); Privacy Act 2020 (NZ); APP and IPP frameworks | MUST | MUST | MUST |
| **Government** | IRAP assessment (AU); HCF hosting certification (AU); NZISM (NZ) | — | — | MUST |
| **Contract** | ACL unfair contract terms (AU, extended to small business B2B 2023); Fair Trading Act (NZ) | — | MUST | MUST |

### 14.13 Canada

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | GST/HST; provincial sales tax (PST) in some provinces; no federal e-invoicing mandate; Peppol pilot | — | MUST | MUST |
| **Data protection** | PIPEDA (federal); Quebec Law 25 (full enforcement 2024); Alberta/BC PIPA; PIPEDA reform pending | MUST | MUST | MUST |
| **Government** | CCCS/ITSG-33; Protected B cloud requirements; ISED procurement | — | — | MUST |
| **Contract** | Provincial consumer protection acts; Internet agreements regulations | — | MUST | MUST |

### 14.14 Russia

| Area | Requirements | UC1 | UC2 | UC3 |
|---|---|---|---|---|
| **Tax** | Electronic document interchange via EDI operators; VAT invoicing (scheta-faktury); tax reporting to FNS | — | MUST | MUST |
| **Data protection** | Federal Law 152-FZ; strict data localization (PI databases in Russia); Roskomnadzor enforcement | MUST | MUST | MUST |
| **Government** | FSTEC/FSB certification; import substitution (domestic IT preference); domestic hosting mandatory | — | — | MUST |

---

## 15. Compliance Requirements Matrix

This matrix maps compliance areas to use cases and priority tiers across all
jurisdictions.

### 15.1 By Compliance Area

| # | Area | UC1 (Internal) | UC2 (MSP) | UC3 (Sovereign) | Priority |
|---|---|---|---|---|---|
| 1 | Tax calculation engine | — | MUST | MUST | P0 (v1.0) |
| 2 | E-invoicing output formats | — | MUST | MUST | P0 (v1.0) |
| 3 | Tax authority API integration | — | MUST | MUST | P1 (v1.x) |
| 4 | Data protection (GDPR/equivalent) | MUST | MUST | MUST | P0 (v1.0) |
| 5 | Data residency controls | COULD | SHOULD | MUST | P0 (v1.0) |
| 6 | Immutable audit trail | SHOULD | MUST | MUST | P0 (v1.0) |
| 7 | Record retention engine | COULD | MUST | MUST | P0 (v1.0) |
| 8 | Sequential invoice numbering | — | MUST | MUST | P0 (v1.0) |
| 9 | Cryptographic document signing | — | MUST | MUST | P1 (v1.x) |
| 10 | Multi-currency with official rates | — | MUST | MUST | P0 (v1.0) |
| 11 | Contract lifecycle management | — | MUST | MUST | P1 (v1.x) |
| 12 | Auto-renewal compliance | — | MUST | MUST | P1 (v1.x) |
| 13 | SOC 2 / ISO 27001 readiness | COULD | SHOULD | MUST | P0 (design) |
| 14 | FedRAMP / EUCS readiness | — | — | MUST | P2 (v2.0) |
| 15 | SBOM generation | COULD | SHOULD | MUST | P0 (v1.0) |
| 16 | PCI DSS avoidance (tokenization) | — | MUST | MUST | P0 (v1.0) |
| 17 | AML/KYC integration points | — | SHOULD | SHOULD | P1 (v1.x) |
| 18 | Machine-readable audit export | COULD | MUST | MUST | P1 (v1.x) |
| 19 | Telecom CDR support | — | — | COULD | P2 (v2.0) |
| 20 | Healthcare (HIPAA) isolation | — | — | COULD | P2 (v2.0) |
| 21 | Air-gapped deployment | — | — | MUST (defense) | P1 (v1.x) |
| 22 | BYOK/HYOK key management | — | SHOULD | MUST | P1 (v1.x) |
| 23 | TM Forum API compatibility | — | COULD | SHOULD (telecom) | P2 (v2.0) |
| 24 | Transfer pricing documentation | SHOULD (MNC) | — | — | P2 (v2.0) |

### 15.2 By Jurisdiction Priority

Based on market size, regulatory complexity, and Meteridian's target markets,
recommended jurisdiction prioritization:

| Priority | Jurisdictions | Rationale |
|---|---|---|
| **P0** (v1.0) | USA, EU (Germany, France), UK | Largest enterprise markets; GDPR as global baseline |
| **P1** (v1.x) | India, Brazil, Saudi Arabia, Australia, Canada, Japan | Large or fast-growing markets with active e-invoicing mandates |
| **P2** (v2.0) | Mexico, Colombia, Chile, Southeast Asia, South Korea, UAE, South Africa | Significant markets with established or emerging e-invoicing |
| **P3** (future) | Russia, China, smaller African and Asian markets | Complex and restricted markets; lower initial priority |

---

## 16. Impact on Meteridian Architecture

The regulatory landscape imposes concrete architectural requirements on
Meteridian. These must be addressed in the platform design, not treated as
afterthoughts.

### 16.1 Data Layer

| Architectural Decision | Regulatory Driver | Affected Components |
|---|---|---|
| **Per-tenant data region pinning** | Data residency laws (Russia, China, India, EU) | TimescaleDB deployment topology; event routing; Redpanda Connect configuration |
| **Configurable retention policies per tenant per jurisdiction** | Varying retention requirements (5-30 years) | TimescaleDB continuous aggregation and retention policies; archival storage |
| **Immutable event log with cryptographic hash chains** | Audit trail requirements (GoBD, SOX, all tax authorities) | Event store design; hash chain computation as part of event ingestion |
| **Separation of PII from billing data** | GDPR erasure vs. tax retention conflict | Data model must support pseudonymization while maintaining tax record integrity |
| **Encryption with customer-managed keys** | Sovereign cloud requirements; HIPAA | Key management service; integration with external HSMs and KMS |

### 16.2 Processing Layer

| Architectural Decision | Regulatory Driver | Affected Components |
|---|---|---|
| **Pluggable tax engine interface** | Varying tax calculation rules worldwide | Block architecture (METR-0002); tax calculation block as a swappable component |
| **E-invoicing output block framework** | 20+ e-invoicing formats globally | Output blocks for UBL 2.1, Factur-X, CFDI, NF-e, etc.; schema validation before submission |
| **Tax authority API integration blocks** | Clearance models (Brazil, India, Saudi Arabia, Mexico, etc.) | Blocks that call tax authority APIs for invoice clearance/reporting |
| **Cryptographic signing blocks** | Digital signature requirements (ICP-Brasil, ZATCA CSID, eIDAS QES) | Block that applies jurisdiction-specific digital signatures |
| **Sequential numbering service** | Gap-free invoice numbering (most jurisdictions) | Centralized, tenant-isolated, persistent counter service |

### 16.3 API Layer

| Architectural Decision | Regulatory Driver | Affected Components |
|---|---|---|
| **Data subject rights endpoints** | GDPR Art. 15-22, CCPA, LGPD, DPDP | API endpoints for access, deletion, portability requests |
| **Audit export endpoints** | GoBD, FEC, SAF-T, SOX | API for exporting billing data in jurisdiction-specific audit formats |
| **Contract lifecycle API** | Auto-renewal laws, cooling-off periods | Contract metadata CRUD with jurisdiction-aware validation |
| **Consent management** | GDPR, DPDP, PIPL | Consent recording and withdrawal API |

### 16.4 Deployment Layer

| Architectural Decision | Regulatory Driver | Affected Components |
|---|---|---|
| **Air-gapped deployment mode** | Government and defense sovereign cloud | Helm charts, operator, all components must function without internet |
| **Multi-region Kubernetes deployment** | Data residency; operational sovereignty | Kubernetes federation; per-region control planes |
| **RBAC with clearance-level attributes** | Government personnel security requirements | Authorization model (planned METR-TBD) |
| **SBOM and provenance for all artifacts** | US EO 14028, EU CRA, FedRAMP | SLSA/Sigstore (ADR-0009); CycloneDX/SPDX SBOM in CI/CD |

### 16.5 Compliance as a Block Category

Meteridian's block architecture (METR-0002) naturally supports compliance as a
block category. Recommended compliance blocks for the marketplace:

| Block | Function | Priority |
|---|---|---|
| `tax-calc-avalara` | Avalara tax calculation integration | P0 |
| `tax-calc-vertex` | Vertex tax calculation integration | P1 |
| `tax-calc-taxjar` | TaxJar tax calculation integration | P1 |
| `einvoice-ubl21` | UBL 2.1 XML e-invoice generation | P0 |
| `einvoice-facturx` | Factur-X PDF/A-3 e-invoice generation | P1 |
| `einvoice-cfdi40` | CFDI 4.0 XML generation for Mexico | P1 |
| `einvoice-nfe` | NF-e XML generation for Brazil | P1 |
| `einvoice-peppol` | Peppol BIS 3.0 e-invoice generation | P1 |
| `clearance-zatca` | ZATCA Fatoora API integration (Saudi Arabia) | P1 |
| `clearance-irp-india` | India GST IRP integration | P1 |
| `clearance-sefaz` | Brazil SEFAZ NF-e clearance | P1 |
| `clearance-sat-mexico` | Mexico SAT/PAC clearance | P2 |
| `signing-icp-brasil` | ICP-Brasil digital signature | P1 |
| `signing-eidas` | eIDAS qualified electronic signature | P1 |
| `signing-zatca-csid` | ZATCA cryptographic stamp | P1 |
| `audit-export-gobd` | GoBD/GDPdU export for Germany | P1 |
| `audit-export-fec` | FEC export for France | P1 |
| `audit-export-saft` | SAF-T export (OECD standard) | P2 |
| `data-residency-enforcer` | Validates and enforces data residency rules | P0 |
| `gdpr-rights-handler` | Data subject rights workflow automation | P1 |
| `pci-tokenizer` | Payment card tokenization (PCI DSS avoidance) | P0 |
| `aml-screening` | Sanctions list and AML screening integration | P2 |

---

## 17. Roadmap Recommendations

### 17.1 v1.0 — Foundation (Compliance-Critical)

The following compliance capabilities must be present in v1.0:

1. **Pluggable tax engine interface** — define the block interface for tax
   calculation; ship with a basic multi-rate VAT/GST engine and integration
   points for Avalara, Vertex
2. **E-invoicing output framework** — UBL 2.1 and EN 16931 as first-class
   output formats; extensible for jurisdiction-specific formats
3. **Immutable audit trail** — event store with cryptographic hash chains,
   configurable per-tenant retention
4. **Data residency metadata** — per-tenant region configuration; enforcement
   at the data layer
5. **Sequential invoice numbering** — gap-free, tenant-isolated numbering
   service
6. **PCI DSS avoidance** — payment tokenization through pluggable payment
   providers (ADR-0010)
7. **SBOM generation** — CycloneDX SBOM for every release
8. **GDPR-compatible data model** — PII separation, erasure capability with
   tax record exemption
9. **SOC 2 controls by design** — access logging, change management, encryption

### 17.2 v1.x — Market Expansion

1. **Tax authority API integration blocks** — ZATCA, India IRP, Brazil SEFAZ
   clearance blocks
2. **Cryptographic signing** — ICP-Brasil, ZATCA CSID, eIDAS integration
3. **Contract lifecycle management** — auto-renewal compliance engine
4. **Machine-readable audit exports** — GoBD, FEC, SAF-T formatters
5. **BYOK/HYOK key management** — customer-controlled encryption keys
6. **Air-gapped deployment mode** — full functionality without internet
7. **AML/KYC integration points** — hooks for sanctions screening and identity
   verification

### 17.3 v2.0 — Enterprise and Sovereign

1. **FedRAMP/EUCS certification support** — security documentation, continuous
   monitoring, machine-readable artifacts
2. **TM Forum API compatibility** — TMF620, TMF678 for telecom markets
3. **Telecom CDR blocks** — 3GPP-compatible CDR ingestion and rating
4. **HIPAA isolation mode** — PHI-specific access controls and encryption
5. **Multi-region federation** — catalog sync, event routing, data residency
   enforcement across regions
6. **Advanced compliance reporting** — dashboards for compliance posture across
   jurisdictions

### 17.4 Recommended ADRs

| ADR | Title | Trigger |
|---|---|---|
| ADR-TBD | Tax engine interface design | Before v1.0 — defines how tax calculation blocks interact with the rating engine |
| ADR-TBD | E-invoicing output architecture | Before v1.0 — defines the output block interface for jurisdiction-specific invoice formats |
| ADR-TBD | Data residency enforcement model | Before v1.0 — how per-tenant geographic constraints are enforced at the data layer |
| ADR-TBD | PII separation and erasure strategy | Before v1.0 — how to support GDPR erasure while preserving tax records |
| ADR-TBD | Cryptographic audit trail design | Before v1.0 — hash chain or Merkle tree approach for immutable billing records |
| ADR-TBD | BYOK/HYOK key management architecture | Before v1.x — integration with external KMS/HSM systems |

---

## 18. Open Questions

### 18.1 Requiring Legal Counsel

These questions require jurisdiction-specific legal review and cannot be
resolved through technical analysis alone:

1. **Credit and prepaid balance classification**: Does Meteridian's credit grant
   system (METR-0004) constitute e-money under PSD3/PSR? Under state money
   transmitter laws? The answer determines whether the operator needs financial
   licensing.

2. **Budget unit classification**: Are internal budget units (METR-0005)
   subject to transfer pricing rules when used across legal entities within a
   multinational enterprise?

3. **Tax nexus creation**: Does providing a billing SaaS platform to customers
   in a jurisdiction create tax nexus for Meteridian (the company) in that
   jurisdiction?

4. **Data processor vs. controller**: In each deployment model, is Meteridian
   the data processor (acting on the operator's instructions) or a joint
   controller? This affects GDPR, LGPD, PIPL obligations.

5. **Export control for cryptographic blocks**: Do Meteridian's cryptographic
   signing blocks require export licenses in any target jurisdiction?

6. **Open source warranty interaction**: How do Apache 2.0 warranty disclaimers
   interact with strict liability regimes in some jurisdictions (e.g., EU
   Product Liability Directive, EU Cyber Resilience Act requirements for
   open-source stewards)?

### 18.2 Requiring Technical Investigation

1. **Tax engine performance**: Can external tax calculation APIs (Avalara,
   Vertex) meet Meteridian's throughput requirements for high-volume metering?
   What is the latency impact of synchronous tax calculation in the rating
   pipeline?

2. **E-invoicing format proliferation**: With 20+ e-invoicing formats globally,
   should Meteridian maintain a core canonical invoice model (EN 16931-based)
   with format-specific output blocks, or should each format be independently
   generated?

3. **Data residency enforcement at scale**: How does per-tenant geographic
   pinning interact with TimescaleDB's continuous aggregation and Redpanda
   Connect's event routing?

4. **Cryptographic hash chain overhead**: What is the performance impact of
   computing and verifying hash chains for every billing event at high
   throughput?

5. **Air-gapped marketplace**: How do operators in air-gapped environments
   install and update blocks from the marketplace?

### 18.3 Requiring Community and Market Input

1. **Jurisdiction prioritization**: The recommended priority order (§15.2) is
   based on market size. Customer demand may differ. Which jurisdictions should
   be prioritized based on actual deployment interest?

2. **Compliance block ownership**: Should compliance blocks (tax engines,
   e-invoicing connectors) be maintained by the Meteridian core team, by
   community contributors, or by commercial partners?

3. **Certification strategy**: Should the Meteridian project pursue any
   certifications (SOC 2, ISO 27001) as an organization, or should this be
   left entirely to deploying organizations?

---

## 19. Related Documents

### Internal

- [METR-0001: Platform Architecture](../0001-architecture/architecture.md) —
  architectural decisions affected by regulatory requirements
- [METR-0002: Platform Extensibility](../0002-extensibility/extensibility.md) —
  block architecture enables compliance as pluggable blocks
- [METR-0003: Product and Service Catalog](../0003-product-catalog/product-catalog.md) —
  catalog must support jurisdiction-specific tax rules and multi-currency
- [METR-0004: Credit, Prepaid, and Token-Based Billing](../0004-credit-token-billing/credit-token-billing.md) —
  credit and prepaid balance design has financial regulation implications
- [METR-0005: Internal Budget Units and Chargeback](../0005-internal-token-economy/internal-token-economy.md) —
  transfer pricing and internal allocation rules
- [METR-0006: Developer Experience and Tooling](../0006-developer-experience/developer-experience.md) —
  compliance block development workflow and marketplace submission pipeline
- [ADR-0010: Pluggable Payment Providers](../../docs/adr/0010-pluggable-payment-providers.md) —
  payment processing architecture that avoids financial licensing
- [ADR-0009: SLSA/Sigstore for Marketplace Provenance](../../docs/adr/0009-slsa-sigstore-provenance.md) —
  supply chain security and SBOM
- [Competitive Analysis: Legal & Regulatory Compliance](../../docs/competitive/legal-regulatory-compliance.md) —
  detailed competitive research on how billing, metering, and FinOps platforms handle compliance

### External References

- [EU ViDA Package — Official EU page](https://taxation-customs.ec.europa.eu/taxation/vat/vat-digital-age-vida_en)
- [OECD BEPS — Base Erosion and Profit Shifting](https://www.oecd.org/tax/beps/)
- [ZATCA Fatoora Platform](https://zatca.gov.sa/en/E-Invoicing/)
- [India GST e-Invoice Portal](https://einvoice1.gst.gov.in/)
- [Brazil NF-e Portal Nacional](https://www.nfe.fazenda.gov.br/)
- [FedRAMP Consolidated Rules 2026](https://www.fedramp.gov/preview/2026/)
- [EU Data Act](https://eur-lex.europa.eu/eli/reg/2023/2854/oj)
- [TM Forum Open APIs](https://www.tmforum.org/open-apis/)
- [EN 16931 — European e-invoicing standard](https://ec.europa.eu/digital-building-blocks/sites/display/DIGITAL/EN+16931)
- [Peppol e-Delivery Network](https://peppol.org/)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [PSD3/PSR — European Parliament legislative train](https://www.europarl.europa.eu/legislative-train/)
- [EU Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act)
