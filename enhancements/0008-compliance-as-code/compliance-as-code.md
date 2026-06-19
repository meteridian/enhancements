# METR-0008: Compliance-as-Code and Audit Trail Guarantees

- **Status:** draft
- **Authors:** @pgarciaq
- **Created:** 2026-06-19
- **Last Updated:** 2026-06-19
- **Depends on:** METR-0001 (Platform Architecture), METR-0002 (Extensibility)
- **Related:** METR-0007 (Legal and Regulatory Compliance), METR-0004 (Credit and Token Billing), METR-0009 (E-Invoicing Engine), ADR-0014 (Cryptographic Audit Trail), ADR-0015 (Compliance Policy Engine)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Motivation](#2-motivation)
3. [Competitive Landscape](#3-competitive-landscape)
4. [Architecture Overview](#4-architecture-overview)
5. [Immutable Audit Trail](#5-immutable-audit-trail)
6. [Compliance Policy Engine](#6-compliance-policy-engine)
7. [Regulatory Rule Sets](#7-regulatory-rule-sets)
8. [Automated Compliance Reporting](#8-automated-compliance-reporting)
9. [Data Lifecycle Management](#9-data-lifecycle-management)
10. [SOC 2 and ISO 27001 Evidence Collection](#10-soc-2-and-iso-27001-evidence-collection)
11. [Integration with Block Architecture](#11-integration-with-block-architecture)
12. [API Design](#12-api-design)
13. [Performance Impact Analysis](#13-performance-impact-analysis)
14. [Deployment Considerations](#14-deployment-considerations)
15. [Roadmap and Phasing](#15-roadmap-and-phasing)
16. [Open Questions](#16-open-questions)
17. [Related Documents](#17-related-documents)

---

## 1. Executive Summary

Meteridian occupies a unique position in the billing and metering landscape: an
open-source platform that can be **compliance-rich by design**. Today, no
open-source metering or rating platform offers compliance-as-code or built-in
audit trail guarantees (see [competitive analysis](../../docs/competitive/legal-regulatory-compliance.md),
Gap 1). The compliance-rich quadrant is occupied exclusively by proprietary
vendors (Zuora, SAP, Salesforce), while open-source alternatives (Lago,
OpenMeter, Kill Bill) leave compliance entirely to the deployer.

This METR defines Meteridian's approach to closing that gap through four
interconnected capabilities:

1. **Immutable Audit Trail** — every billing-significant event is captured in a
   cryptographically verifiable, append-only log that satisfies audit
   requirements across jurisdictions (GoBD, SOX, GST, ZATCA, and others
   catalogued in [METR-0007 §8](../0007-legal-regulatory-compliance/legal-regulatory-compliance.md#8-audit-record-retention-and-reporting)).

2. **Compliance Policy Engine** — a declarative, version-controlled system for
   defining compliance rules as code. Operators can express requirements like
   "invoices in Germany must be retained for 10 years" or "PII must not leave
   the EU region" as executable policies that are continuously evaluated against
   the live system.

3. **Regulatory Rule Sets** — pre-built, community-maintained policy packages
   for specific jurisdictions and standards (GDPR, SOX, GoBD, India GST, etc.),
   distributed through the block marketplace (METR-0002).

4. **Automated Compliance Reporting** — the platform generates audit evidence
   artifacts (SOC 2 control evidence, ISO 27001 ISMS artifacts, jurisdiction-
   specific audit exports in GoBD, FEC, SAF-T formats) from its own operational
   data, reducing the manual burden of compliance programs.

**Why this matters:** An enterprise deploying Meteridian today would need to
build all of this from scratch — audit logging, retention policies, compliance
checks, evidence collection. By embedding these capabilities in the platform
itself, Meteridian makes compliance a configuration decision rather than an
engineering project. This is the single strongest differentiator against both
proprietary competitors (who charge premium prices for compliance features)
and open-source alternatives (who offer none).

---

## 2. Motivation

### 2.1 The Market Gap

The [competitive research](../../docs/competitive/legal-regulatory-compliance.md)
identified a 2×2 matrix with an empty quadrant:

|  | Proprietary | Open-Source |
|--|-------------|-------------|
| **Compliance-rich** | Zuora, SAP, Salesforce | **(empty — Meteridian's target)** |
| **Compliance-light** | Orb, Metronome, Amberflo | Lago, OpenMeter, Kill Bill |

Every open-source billing platform provides compliance _tooling_ (audit logs,
RBAC) but leaves the compliance _program_ entirely to the deployer. No OSS
platform provides:

- Pre-built compliance policies as code
- Automated compliance report generation
- Audit-ready data export in regulatory-specific formats
- Built-in data retention and purge workflows for GDPR Article 17

### 2.2 Why Compliance Must Be a Core Capability

Compliance cannot be an afterthought because it affects fundamental architectural
decisions:

1. **Every write path must be auditable.** If the audit trail is a plugin, it can
   be bypassed. If it is woven into the event ingestion pipeline, it cannot.

2. **Retention policies must be enforced at the data layer.** A "retention block"
   running alongside the pipeline cannot guarantee that data is actually purged
   from TimescaleDB continuous aggregates, Redpanda topics, and archived storage.
   The platform itself must enforce retention.

3. **Compliance policies must be evaluated continuously.** A monthly compliance
   check finds violations after they have occurred. A policy engine that runs
   in-line with operations prevents violations before they are committed.

4. **Evidence must be generated from operational data.** If compliance evidence
   is collected manually (screenshots, spreadsheets), it is expensive, error-
   prone, and not repeatable. If the platform generates evidence from its own
   logs and metrics, compliance programs become automated.

### 2.3 Customer Segments That Require This

| Segment | Why They Need Compliance-as-Code | Reference |
|---------|----------------------------------|-----------|
| Sovereign cloud operators | Government procurement mandates audit trails, data residency, security certification artifacts | METR-0007 §11 |
| MSPs serving regulated industries | Healthcare (HIPAA), financial services (SOX), telecom — each requires domain-specific audit evidence | METR-0007 §9 |
| Enterprises with internal FinOps | SOC 2 Type II is now table stakes for B2B SaaS; generating evidence is the #1 cost of compliance programs | METR-0007 §10 |
| Multi-jurisdictional operators | Different retention periods, different audit export formats, different data residency rules per country | METR-0007 §13 |

---

## 3. Competitive Landscape

### 3.1 What Exists Today

| Platform | Audit Logs | Immutability | Compliance Policies | Auto Reports | Audit Exports |
|----------|-----------|-------------|-------------------|-------------|--------------|
| **SAP BRIM** | Full enterprise audit | Yes (ERP-grade) | Via GRC modules | Yes | Yes (GoBD, FEC, SAF-T) |
| **Zuora** | Yes | Configurable | Limited | Via Rev Pro | Via integrations |
| **Stripe** | API-accessible | Yes (append-only) | No | No | No |
| **Lago** (OSS) | Event and config changes | No guarantees | No | No | No |
| **Kill Bill** (OSS) | Append-only | Yes (architecture) | No | No | No |
| **OpenMeter** (OSS) | Cloud only | No | No | No | No |
| **Meteridian** (target) | **Cryptographic hash chains** | **Yes (verifiable)** | **Yes (policy engine)** | **Yes (automated)** | **Yes (GoBD, FEC, SAF-T, SOX)** |

### 3.2 What Makes Meteridian Different

Kill Bill has append-only audit logs — but no way to verify they haven't been
tampered with after the fact. Lago has event logs — but no retention enforcement
or compliance policy evaluation. SAP has everything — but costs millions and
takes 12-24 months to implement.

Meteridian's approach is unique:

1. **Cryptographic verifiability** — every audit entry is hash-chained, so
   tampering is detectable without trusting the platform operator (ADR-0014)
2. **Policy-as-code** — compliance rules are version-controlled, testable, and
   distributable through the marketplace (ADR-0015)
3. **Open source** — the compliance engine itself is auditable, unlike
   proprietary black-box implementations

---

## 4. Architecture Overview

Compliance-as-code integrates with Meteridian at three architectural layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                       API / Control Plane                        │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Compliance    │  │ Report       │  │ Data Lifecycle          │ │
│  │ Policy API    │  │ Generation   │  │ Management API          │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────────┘ │
├─────────┼──────────────────┼───────────────────┼────────────────┤
│         │        Processing / Block Runtime     │                │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────────▼───────────────┐ │
│  │ Policy       │  │ Evidence     │  │ Retention              │ │
│  │ Evaluator    │  │ Collector    │  │ Enforcer               │ │
│  │ (OPA/Rego)   │  │              │  │                        │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────────┘ │
├─────────┼──────────────────┼───────────────────┼────────────────┤
│         │           Data Layer                  │                │
│  ┌──────▼───────────────────────────────────────▼──────────────┐ │
│  │                  Immutable Audit Log                         │ │
│  │        (TimescaleDB + Cryptographic Hash Chains)            │ │
│  │                       (ADR-0014)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Key design principle:** The audit trail and retention enforcer operate at the
platform level, not as blocks. They cannot be bypassed, misconfigured by a
pipeline author, or disabled by a block. Compliance policies and reporting,
however, are extensible through the block architecture — operators can add
jurisdiction-specific rules and export formats as blocks.

---

## 5. Immutable Audit Trail

The immutable audit trail is the foundation of Meteridian's compliance
architecture. Every billing-significant action is captured as an audit entry
that is append-only, cryptographically chained, and independently verifiable.

### 5.1 What Gets Audited

| Category | Events | Justification |
|----------|--------|---------------|
| **Billing events** | Usage ingestion, rating calculations, invoice generation, credit grants and drawdowns, payment recording | Tax law (all jurisdictions), SOX §302/§404, GoBD |
| **Configuration changes** | Product catalog updates, price changes, rate card modifications, pipeline changes | SOC 2 CC6.1 (change management), GoBD Verfahrensdokumentation |
| **Access events** | API authentication, data access, report generation, export requests | GDPR Art. 30, SOC 2 CC6.1-CC6.3, HIPAA §164.312 |
| **Policy changes** | Compliance policy create/update/delete, rule set activation/deactivation | SOX (control environment), ISO 27001 A.5 |
| **Data lifecycle** | Retention holds, purge requests, erasure executions, archival operations | GDPR Art. 17, jurisdiction-specific retention laws (METR-0007 §8.1) |
| **System events** | Schema migrations, block installations, certificate rotations, key rotations | SOC 2 CC7, FedRAMP CA-7 (continuous monitoring) |

### 5.2 Audit Entry Schema

Each audit entry contains:

```json
{
  "entry_id": "uuid-v7",
  "tenant_id": "uuid",
  "timestamp": "2026-06-19T10:30:00.000000Z",
  "timestamp_source": "NTP",
  "event_type": "billing.invoice.generated",
  "actor": {
    "type": "user|system|block|api_key",
    "id": "uuid",
    "ip_address": "192.168.1.1",
    "user_agent": "..."
  },
  "resource": {
    "type": "invoice",
    "id": "uuid",
    "version": 1
  },
  "action": "create",
  "details": { },
  "previous_hash": "sha256:abc123...",
  "entry_hash": "sha256:def456...",
  "sequence_number": 12345678
}
```

### 5.3 Cryptographic Integrity

The audit trail uses hash chains for tamper detection (see ADR-0014 for the
architectural decision):

- Each audit entry includes the hash of the previous entry (`previous_hash`)
- The `entry_hash` is computed as `SHA-256(entry_id || timestamp || event_type
  || actor || resource || action || details || previous_hash)`
- Hash chains are per-tenant to allow independent verification
- A periodic checkpoint mechanism anchors the chain to an external timestamping
  service (RFC 3161 or blockchain-based) for non-repudiation

### 5.4 Sequential Numbering

Audit entries receive a strictly monotonic `sequence_number` within each tenant.
This satisfies the gap-free sequential numbering requirement for invoices
(METR-0007 §8.2) and extends it to all audit entries:

- The sequence is maintained by an atomic counter in PostgreSQL (not Valkey,
  because counter persistence across restarts is critical for audit integrity)
- Gap detection is built in: any gap in the sequence indicates a potential
  integrity violation and triggers an alert
- The counter is tenant-isolated: each tenant has its own sequence space

### 5.5 Storage and Retention

- **Primary store:** TimescaleDB hypertable with time-based partitioning
  (monthly partitions)
- **Retention:** Per-tenant, per-jurisdiction configurable retention policies
  (see §9)
- **Archival:** Partitions older than the hot retention window are compressed
  and optionally exported to cold storage (S3-compatible) while maintaining
  hash chain continuity
- **Export:** Audit entries can be exported in standard formats (JSON Lines,
  CSV, Parquet) for external audit tools

### 5.6 Verification API

Any party (auditor, tenant, regulator) can independently verify the audit trail:

```
GET /api/v1/compliance/audit-trail/verify
  ?tenant_id={uuid}
  &start_sequence={n}
  &end_sequence={m}

Response:
{
  "verified": true,
  "entries_checked": 1000,
  "chain_valid": true,
  "first_entry": { "sequence": 1000, "hash": "..." },
  "last_entry": { "sequence": 2000, "hash": "..." },
  "checkpoint_anchors": [
    { "sequence": 1500, "anchor_type": "rfc3161", "timestamp": "...", "proof": "..." }
  ]
}
```

---

## 6. Compliance Policy Engine

The compliance policy engine enables operators to express regulatory
requirements as executable code — policies that are continuously evaluated
against the live system state.

### 6.1 Policy Language

Policies are written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/)
(Open Policy Agent's policy language). This choice is made in ADR-0015 and is
based on:

- **Industry adoption:** OPA/Rego is the de facto standard for policy-as-code
  in cloud-native systems (Kubernetes admission control, Envoy authorization,
  Terraform Sentinel alternative)
- **Declarative:** Policies express _what_ must be true, not _how_ to check it
- **Testable:** Rego policies can be unit-tested with `opa test`
- **Version-controllable:** Policies are text files that can be stored in Git,
  reviewed in PRs, and deployed through CI/CD
- **Embeddable:** The OPA engine can be embedded as a Go library (Meteridian is
  Go-based), avoiding external service dependencies

### 6.2 Policy Categories

| Category | Examples | Evaluation Trigger |
|----------|----------|-------------------|
| **Data residency** | "Tenant X's data must not leave EU region" | Every data write, replication event |
| **Retention** | "German tax invoices must be retained for 10 years" | Purge operations, archival decisions |
| **Access control** | "Only users with security clearance can access tenant Y" | Every API request (middleware) |
| **Billing integrity** | "Invoices must have sequential numbering with no gaps" | Invoice generation |
| **Data protection** | "PII fields must be encrypted at rest" | Schema changes, data writes |
| **Configuration** | "Production pipelines must have at least 2 replicas" | Pipeline deployment |
| **Operational** | "Audit log export must complete within 24 hours of request" | Scheduled evaluation |

### 6.3 Policy Structure

```rego
# package: meteridian.compliance.gdpr.retention
package meteridian.compliance.gdpr.retention

import rego.v1

default compliant := false

# Tax invoices in Germany must be retained for 10 years (AO §147)
compliant if {
    input.resource.type == "invoice"
    input.tenant.jurisdiction == "DE"
    input.retention_policy.duration_years >= 10
}

# GDPR right to erasure — PII can be erased unless tax retention applies
erasure_allowed if {
    input.resource.type == "personal_data"
    not tax_retention_applies
}

tax_retention_applies if {
    input.resource.linked_invoices[_].jurisdiction == "DE"
    input.resource.linked_invoices[_].age_years < 10
}

# Violation details for reporting
violations contains msg if {
    not compliant
    msg := sprintf(
        "Retention policy for tenant %s in jurisdiction %s is %d years, minimum required is 10",
        [input.tenant.id, input.tenant.jurisdiction, input.retention_policy.duration_years]
    )
}
```

### 6.4 Policy Evaluation Modes

| Mode | When | Latency Budget | Failure Action |
|------|------|---------------|----------------|
| **Inline (synchronous)** | Before committing a billing event, data write, or configuration change | <5ms per evaluation | Reject the operation with policy violation details |
| **Continuous (async)** | Periodic background sweep of system state | N/A (background) | Generate compliance violation alerts |
| **On-demand** | Auditor or operator requests compliance check | Seconds to minutes | Generate compliance report |

### 6.5 Policy Bundles

Policies are packaged as **OPA bundles** — versioned tarballs containing Rego
files, data files, and metadata. They are distributed through Meteridian's
block marketplace:

```yaml
# policy-bundle.yaml
name: "meteridian-gdpr-eu"
version: "1.2.0"
description: "GDPR compliance policies for EU jurisdictions"
jurisdictions: ["EU", "DE", "FR", "IT", "ES", "NL", "BE", "AT"]
policies:
  - path: "retention/invoice_retention.rego"
  - path: "data_residency/eu_boundary.rego"
  - path: "erasure/right_to_erasure.rego"
  - path: "access/consent_verification.rego"
data:
  - path: "data/retention_periods.json"
  - path: "data/eu_member_states.json"
```

---

## 7. Regulatory Rule Sets

Regulatory rule sets are pre-built policy bundles that encode jurisdiction-
specific requirements. They transform the compliance requirements catalogued
in METR-0007 into executable policies.

### 7.1 Planned Rule Sets

| Rule Set | Jurisdictions | Key Policies | Priority |
|----------|--------------|-------------|----------|
| `meteridian-gdpr-eu` | EU (all 27 Member States) | Data residency, right to erasure, consent, retention (METR-0007 §5.1) | P0 |
| `meteridian-sox-us` | USA | Audit trail integrity (7-year retention), change management, access logging (METR-0007 §8.1) | P0 |
| `meteridian-gobd-de` | Germany | 10-year invoice retention, tamper-evident storage, procedural documentation, GDPdU export format (METR-0007 §8.3) | P1 |
| `meteridian-gst-in` | India | GST e-invoicing compliance checks, 8-year retention, sequential numbering, IRP integration verification (METR-0007 §13.4) | P1 |
| `meteridian-zatca-sa` | Saudi Arabia | ZATCA Fatoora compliance, UBL 2.1 validation, cryptographic stamp verification, 6-year retention (METR-0007 §13.9) | P1 |
| `meteridian-lgpd-br` | Brazil | LGPD data protection, NF-e retention (5 years), ICP-Brasil signature verification (METR-0007 §13.7) | P1 |
| `meteridian-soc2` | Global | SOC 2 Trust Services Criteria mapping, control evidence collection, continuous monitoring (METR-0007 §10.1) | P0 |
| `meteridian-iso27001` | Global | ISO 27001:2022 Annex A controls mapping, ISMS artifact generation (METR-0007 §10.1) | P1 |
| `meteridian-hipaa-us` | USA | PHI data isolation, BAA support, access logging, minimum necessary standard (METR-0007 §9.3) | P2 |
| `meteridian-fedramp` | USA | NIST SP 800-53 Rev5 control mapping, continuous monitoring, POA&M tracking (ADR-0017) | P2 |
| `meteridian-fec-fr` | France | FEC export format, 10-year accounting retention, Factur-X validation (METR-0007 §13.2) | P1 |
| `meteridian-ccpa-us` | USA (California) | Consumer data rights, opt-out mechanisms, data sale restrictions | P1 |

### 7.2 Community Contribution Model

Rule sets follow the same tiered trust model as blocks (METR-0002 §9):

| Tier | Source | Review | Trust Level |
|------|--------|--------|-------------|
| **Official** | Meteridian core team | Full legal and technical review | Runs with full platform access |
| **Verified partner** | Compliance consultancies, law firms, Big 4 accounting firms | Code review + legal attestation | Runs with scoped access |
| **Community** | Anyone | Automated testing only | Sandboxed; operator must explicitly enable |

### 7.3 Rule Set Versioning

Regulations change. Rule sets must be version-controlled with clear changelogs
that map to regulatory changes:

```
CHANGELOG.md:
## v1.3.0 (2027-01-15)
- Updated EU ViDA cross-border e-invoicing rules for July 2030 deadline
- Added Slovenia domestic e-invoicing rules (effective January 2027)
- Fixed: German B2B e-invoicing threshold (lowered to match 2027 mandate)

## v1.2.0 (2026-09-01)
- Added Spain Veri*factu rules for 2026 mandate
- Updated India GST e-invoicing threshold to ₹2 crore
```

---

## 8. Automated Compliance Reporting

### 8.1 Report Types

| Report | Standard | Output Format | Audience | Frequency |
|--------|----------|--------------|----------|-----------|
| **SOC 2 Evidence Package** | AICPA Trust Services Criteria | ZIP (JSON + screenshots + logs) | External auditors | Continuous collection, periodic export |
| **ISO 27001 ISMS Artifacts** | ISO 27001:2022 | DOCX/PDF + structured JSON | Certification body | Annual, with continuous updates |
| **GoBD Export** | GoBD/GDPdU (Germany) | CSV/XML per GDPdU schema | German tax auditor (Betriebsprüfer) | On-demand |
| **FEC Export** | FEC (France) | Tab-delimited text per FEC format | French tax administration | On-demand |
| **SAF-T Export** | OECD SAF-T v2.0 | XML per SAF-T schema | Tax authorities (Portugal, Norway, Poland, Lithuania, etc.) | On-demand |
| **GDPR Records of Processing** | GDPR Art. 30 | JSON or PDF | Data Protection Authority | On-demand |
| **Compliance Posture Dashboard** | Internal | JSON (API) or HTML | Compliance team | Real-time |
| **Audit Trail Integrity Report** | Internal | JSON or PDF | Internal and external auditors | On-demand or scheduled |

### 8.2 Evidence Collection

The evidence collector continuously gathers compliance-relevant data from
Meteridian's operational systems:

- **Access logs** → SOC 2 CC6.1 (logical access security)
- **Configuration change history** → SOC 2 CC8.1 (change management)
- **Encryption status** → SOC 2 CC6.7 (encryption), ISO 27001 A.8.24
- **Incident response logs** → SOC 2 CC7.3-CC7.5 (incident management)
- **Uptime and availability metrics** → SOC 2 A1.1-A1.3 (availability)
- **Data residency verification** → GDPR, national data residency laws
- **Retention policy compliance** → Jurisdiction-specific requirements

### 8.3 Export Blocks

Audit export formats are implemented as output blocks in the block architecture
(METR-0002), enabling the community to add jurisdiction-specific formats:

| Block | Description | Priority |
|-------|------------|----------|
| `compliance-export-gobd` | GoBD/GDPdU export for German tax audits | P1 |
| `compliance-export-fec` | FEC export for French tax administration | P1 |
| `compliance-export-saft` | SAF-T export (OECD standard, multiple countries) | P1 |
| `compliance-export-soc2` | SOC 2 evidence package generator | P0 |
| `compliance-export-iso27001` | ISO 27001 ISMS artifact generator | P1 |
| `compliance-export-gdpr-ropa` | GDPR Records of Processing Activities | P1 |

---

## 9. Data Lifecycle Management

METR-0007 §8.1 catalogs retention requirements ranging from 3 years (US IRS)
to 30 years (China permanent records). METR-0007 §5 identifies the fundamental
tension between data protection laws (right to erasure) and tax retention
obligations. This section defines how Meteridian manages that lifecycle.

### 9.1 Retention Policy Model

```yaml
retention_policy:
  tenant_id: "uuid"
  jurisdiction: "DE"
  rules:
    - resource_type: "invoice"
      min_retention_years: 10
      regulation: "AO §147"
      action_after_expiry: "archive_then_purge"
    - resource_type: "business_correspondence"
      min_retention_years: 6
      regulation: "AO §147"
      action_after_expiry: "purge"
    - resource_type: "personal_data"
      max_retention: "until_purpose_fulfilled"
      regulation: "GDPR Art. 5(1)(e)"
      action_after_expiry: "anonymize"
      exception: "tax_retention_override"
```

### 9.2 Erasure vs. Retention Conflict Resolution

When GDPR right to erasure (Art. 17) conflicts with tax retention requirements,
tax retention wins per GDPR Art. 17(3)(b) ("compliance with a legal obligation").
Meteridian implements this as:

1. **PII separation** — personal data is stored separately from billing data,
   linked by a pseudonymous identifier
2. **Selective erasure** — when an erasure request arrives, PII is purged but
   billing records are retained with the pseudonymous identifier
3. **Audit trail of erasure** — the erasure action itself is recorded in the
   audit trail with the legal basis for any data that was retained
4. **Automated conflict detection** — the policy engine detects when an erasure
   request would conflict with a retention obligation and returns a structured
   response explaining which data is retained and why

### 9.3 Data Subject Rights Workflows

| Right | GDPR Article | Implementation |
|-------|-------------|---------------|
| Right of access | Art. 15 | API endpoint returns all data associated with a data subject identifier |
| Right to rectification | Art. 16 | API endpoint to correct personal data; change recorded in audit trail |
| Right to erasure | Art. 17 | Workflow that checks retention policies, executes selective erasure, returns report |
| Right to data portability | Art. 20 | Export in machine-readable format (JSON, CSV) |
| Right to restriction | Art. 18 | Marks data as restricted; blocks processing while maintaining storage |
| Right to object | Art. 21 | Records objection; blocks specified processing activities |

---

## 10. SOC 2 and ISO 27001 Evidence Collection

### 10.1 SOC 2 Trust Services Criteria Mapping

Meteridian maps its built-in controls to SOC 2 Trust Services Criteria:

| TSC | Criteria | Meteridian Control | Evidence Source |
|-----|---------|-------------------|----------------|
| CC1.1-CC1.5 | Control environment | Policy engine configuration, role assignments | Policy API, RBAC configuration |
| CC2.1-CC2.3 | Communication and information | Compliance dashboard, alert configuration | Dashboard API, alert logs |
| CC3.1-CC3.4 | Risk assessment | Policy evaluation results, violation trends | Policy evaluation logs |
| CC5.1-CC5.3 | Control activities | Pipeline configuration, block security tiers | Pipeline definitions, marketplace metadata |
| CC6.1-CC6.8 | Logical and physical access | Authentication logs, API key management, RBAC | Audit trail (access events) |
| CC7.1-CC7.5 | System operations | Monitoring alerts, incident response logs | Observability stack, audit trail |
| CC8.1 | Change management | Configuration change audit trail | Audit trail (configuration events) |
| CC9.1-CC9.2 | Risk mitigation | Compliance policy evaluations, remediation tracking | Policy engine, compliance reports |
| A1.1-A1.3 | Availability | Uptime metrics, failover logs, capacity planning | Observability stack |
| PI1.1-PI1.5 | Processing integrity | Hash chain verification, rating accuracy validation | Audit trail, verification API |
| C1.1-C1.2 | Confidentiality | Encryption status, access controls, data classification | Encryption audit, RBAC configuration |
| P1.0-P8.1 | Privacy | Data lifecycle management, consent records, DPA compliance | Data lifecycle API, consent records |

### 10.2 Continuous Compliance Monitoring

Rather than point-in-time audits, Meteridian supports continuous compliance
monitoring:

- **Control effectiveness** — each SOC 2 control is tested automatically at
  configurable intervals
- **Drift detection** — if a control drifts from its expected state (e.g.,
  encryption is disabled, RBAC bypassed), an alert is generated
- **Evidence freshness** — evidence artifacts are timestamped and stale evidence
  triggers re-collection

---

## 11. Integration with Block Architecture

Compliance-as-code extends Meteridian's block architecture (METR-0002) in
two ways:

### 11.1 Compliance as a Block Category

The block marketplace (METR-0002 §9) includes a `compliance` category:

| Block Type | Description | Examples |
|-----------|------------|---------|
| `compliance/policy` | Regulatory policy bundles (OPA bundles) | `meteridian-gdpr-eu`, `meteridian-sox-us` |
| `compliance/export` | Audit export format generators | `compliance-export-gobd`, `compliance-export-fec` |
| `compliance/validator` | Pre-submission validators for external submissions | `validator-zatca-ubl`, `validator-gst-irp` |
| `compliance/connector` | Connectors to external compliance tools | `connector-vanta`, `connector-drata`, `connector-secureframe` |

### 11.2 Compliance Hooks in the Block Runtime

The block runtime (METR-0002 §6, ADR-0007) provides compliance hooks at
pipeline execution boundaries:

```
Event ingestion → [Pre-processing compliance check] → Block pipeline
→ [Post-processing compliance check] → Output
```

- **Pre-processing:** Validates that incoming data complies with data residency
  and data protection policies before it enters the pipeline
- **Post-processing:** Validates that output (invoices, reports, exports)
  complies with format, numbering, and content requirements before delivery
- **Block audit:** Every block invocation is recorded in the audit trail with
  input/output hashes for non-repudiation

---

## 12. API Design

### 12.1 Compliance Policy API

```
POST   /api/v1/compliance/policies              # Create policy
GET    /api/v1/compliance/policies               # List policies
GET    /api/v1/compliance/policies/{id}          # Get policy details
PUT    /api/v1/compliance/policies/{id}          # Update policy
DELETE /api/v1/compliance/policies/{id}          # Deactivate policy

POST   /api/v1/compliance/policies/{id}/evaluate # Evaluate policy against current state
GET    /api/v1/compliance/policies/{id}/results  # Get evaluation history
```

### 12.2 Audit Trail API

```
GET    /api/v1/compliance/audit-trail            # Query audit entries (filtered)
GET    /api/v1/compliance/audit-trail/verify     # Verify hash chain integrity
POST   /api/v1/compliance/audit-trail/export     # Export audit entries in specified format
GET    /api/v1/compliance/audit-trail/checkpoints # List external anchor checkpoints
```

### 12.3 Data Lifecycle API

```
POST   /api/v1/compliance/retention-policies     # Create retention policy
GET    /api/v1/compliance/retention-policies      # List retention policies
PUT    /api/v1/compliance/retention-policies/{id} # Update retention policy

POST   /api/v1/compliance/data-subject/access    # Data subject access request
POST   /api/v1/compliance/data-subject/erasure   # Data subject erasure request
GET    /api/v1/compliance/data-subject/erasure/{id} # Erasure request status
POST   /api/v1/compliance/data-subject/export    # Data subject portability export
```

### 12.4 Compliance Reporting API

```
POST   /api/v1/compliance/reports/generate       # Generate compliance report
GET    /api/v1/compliance/reports                 # List generated reports
GET    /api/v1/compliance/reports/{id}            # Download report
GET    /api/v1/compliance/posture                 # Real-time compliance posture summary
```

---

## 13. Performance Impact Analysis

Compliance-as-code adds overhead to every write path. This overhead must be
bounded and measurable.

### 13.1 Overhead Budget

| Operation | Overhead Source | Target Latency Impact | Mitigation |
|-----------|----------------|----------------------|------------|
| Audit entry creation | Hash computation + DB write | <1ms per event | Batch audit writes; compute hash inline but persist async |
| Inline policy evaluation | OPA evaluation | <5ms per evaluation | Pre-compile policies; cache evaluation results for identical inputs |
| Sequence number allocation | Atomic counter increment | <0.5ms per event | PostgreSQL `nextval()` on per-tenant sequence |
| Hash chain maintenance | SHA-256 of previous + current | <0.1ms per entry | Single hash computation, negligible |

### 13.2 Throughput Considerations

At Meteridian's target throughput (100K+ events/second), the audit trail must
sustain high write rates:

- **Batching:** Audit entries are written in batches (configurable batch size,
  default 1000 entries or 100ms flush interval)
- **Asynchronous persistence:** The hash chain is computed synchronously (for
  integrity), but persistence to TimescaleDB is asynchronous with a write-
  ahead buffer
- **Policy evaluation caching:** Identical policy inputs (same tenant, same
  policy version, same resource type) produce identical results; cache results
  with TTL

### 13.3 Benchmark Requirements

Before v1 GA, the following must be measured (not estimated):

| Metric | Target | Measured? |
|--------|--------|-----------|
| Audit entry write throughput | ≥100K entries/second | No — need benchmark |
| Inline policy evaluation p99 latency | <5ms | No — need benchmark |
| Hash chain verification speed | ≥1M entries/minute | No — need benchmark |
| GoBD export generation (1M invoices) | <10 minutes | No — need benchmark |

---

## 14. Deployment Considerations

### 14.1 Air-Gapped Environments

Compliance-as-code must function fully in air-gapped deployments (METR-0007
§11.3):

- Policy bundles are distributed as OPA bundles (tarballs), loadable from
  local storage
- External timestamping service (RFC 3161) is optional; air-gapped deployments
  can use internal trusted time sources
- Compliance reports are generated locally without external API calls

### 14.2 Multi-Region Deployments

In multi-region deployments, audit trails are region-local (data residency
compliance) with optional cross-region checkpoint synchronization for global
compliance posture views.

### 14.3 Operator Responsibility

Meteridian provides the compliance _engine_. The operator is responsible for:

- Selecting and activating the appropriate rule sets for their jurisdictions
- Configuring retention policies per tenant
- Responding to data subject requests (Meteridian provides the API; the
  operator provides the customer-facing workflow)
- Obtaining certifications (SOC 2, ISO 27001, FedRAMP) for their specific
  deployment — Meteridian generates the evidence, but the certification is
  issued to the operator, not the open-source project

---

## 15. Roadmap and Phasing

### Phase 1: v1.0 — Foundation

| Capability | Description |
|-----------|------------|
| Immutable audit trail | Cryptographic hash chains, per-tenant isolation, TimescaleDB storage |
| Sequential numbering | Gap-free, tenant-isolated sequence service |
| Basic retention policies | Per-tenant configurable retention with automated purge |
| PII separation | Pseudonymous identifiers, selective erasure |
| Verification API | Hash chain verification endpoint |
| SOC 2-aligned logging | Access, configuration, and billing event logging |

### Phase 2: v1.x — Policy Engine and Reporting

| Capability | Description |
|-----------|------------|
| OPA-based policy engine | Inline and continuous evaluation modes |
| GDPR rule set | Data residency, erasure, consent policies |
| SOX rule set | Audit trail integrity, change management policies |
| Data subject rights API | Access, erasure, portability, restriction endpoints |
| SOC 2 evidence collector | Automated evidence gathering |
| GoBD and FEC export blocks | German and French audit export formats |

### Phase 3: v2.0 — Ecosystem and Advanced

| Capability | Description |
|-----------|------------|
| Rule set marketplace | Community-contributed regulatory rule sets |
| SAF-T export | OECD standard audit export |
| ISO 27001 artifact generator | ISMS documentation automation |
| Compliance posture dashboard | Real-time multi-jurisdiction compliance view |
| External tool connectors | Vanta, Drata, Secureframe integration |
| FedRAMP continuous monitoring | NIST 800-53 control monitoring (see ADR-0017) |

---

## 16. Open Questions

### 16.1 Architectural

1. **OPA embedding vs. external OPA server**: Should the OPA engine be embedded
   as a Go library (lower latency, simpler deployment) or run as a sidecar
   (independent scaling, standard OPA tooling)? See ADR-0015.

2. **Hash chain algorithm**: SHA-256 is the proposed default. Should we support
   configurable algorithms for jurisdictions that require specific cryptographic
   standards (e.g., GOST for Russia)?

3. **External timestamping cost**: RFC 3161 timestamping services charge per
   timestamp. What is the optimal checkpoint frequency to balance cost against
   non-repudiation strength?

### 16.2 Operational

4. **Policy bundle testing**: How should operators test policy changes before
   deploying to production? A staging environment? A dry-run mode?

5. **Compliance dashboard access control**: Who should have access to the
   compliance posture dashboard — only compliance officers, or also platform
   engineers?

### 16.3 Legal

6. **Liability for rule set accuracy**: If a community-contributed rule set
   contains an error (e.g., wrong retention period), who is liable? The
   contributor? The operator who activated it? This needs legal review.

7. **Cross-border audit trail access**: If a regulator in Country A requests
   audit trail data that includes entries from Country B tenants, how does
   Meteridian handle the conflict?

---

## 17. Related Documents

### Internal

- [METR-0001: Platform Architecture](../0001-architecture/architecture.md) —
  TimescaleDB event store provides the foundation for the audit trail
- [METR-0002: Platform Extensibility](../0002-extensibility/extensibility.md) —
  compliance blocks extend the block marketplace; policy bundles are distributed
  as marketplace artifacts
- [METR-0004: Credit, Prepaid, and Token-Based Billing](../0004-credit-token-billing/credit-token-billing.md) —
  credit ledger operations must be fully audited
- [METR-0007: Legal and Regulatory Compliance](../0007-legal-regulatory-compliance/legal-regulatory-compliance.md) —
  the regulatory requirements that this METR implements as platform capabilities
- [METR-0009: Native E-Invoicing Engine](../0009-e-invoicing-engine/e-invoicing-engine.md) —
  e-invoicing output must comply with audit trail and sequential numbering requirements
- [ADR-0014: Cryptographic Audit Trail Architecture](../../docs/adr/0014-cryptographic-audit-trail.md) —
  hash chain vs. Merkle tree decision for audit trail immutability
- [ADR-0015: Compliance Policy Engine Architecture](../../docs/adr/0015-compliance-policy-engine.md) —
  OPA embedding strategy and policy evaluation architecture

### External References

- [Open Policy Agent](https://www.openpolicyagent.org/) — Policy-as-code engine
- [RFC 3161: Internet X.509 PKI Timestamp Protocol](https://datatracker.ietf.org/doc/html/rfc3161) — Trusted timestamping
- [AICPA SOC 2 Trust Services Criteria](https://us.aicpa.org/interestareas/frc/assuranceadvisoryservices/trustdataintegritytaskforce) — SOC 2 framework
- [ISO 27001:2022](https://www.iso.org/standard/27001) — Information security management
- [GoBD (Germany)](https://www.bundesfinanzministerium.de/Content/DE/Downloads/BMF_Schreiben/Weitere_Steuerthemen/Abgabenordnung/2019-11-28-GoBD.html) — German electronic bookkeeping requirements
- [FEC Format (France)](https://bofip.impots.gouv.fr/bofip/9028-PGP.html/identifiant%3DBOI-CF-IOR-60-40-20-20140801) — French audit file format
- [OECD SAF-T](https://www.oecd.org/tax/forum-on-tax-administration/publications-and-products/technologies/34910394.pdf) — Standard Audit File for Tax
