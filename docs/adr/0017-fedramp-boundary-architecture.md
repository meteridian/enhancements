# ADR-0017: FedRAMP Boundary Architecture and Sovereign Cloud Deployment

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq
- **Related:** METR-0007 (Legal and Regulatory Compliance §9.1, §11), METR-0008 (Compliance-as-Code)

## Context

The [competitive analysis](../../docs/competitive/legal-regulatory-compliance.md)
identified that no pure-play billing or metering platform holds FedRAMP
authorization (Gap 2). Only ERP vendors (SAP, Oracle, Salesforce) and IBM's
FinOps products (Apptio, Turbonomic) have achieved it. Meteridian, being
deployable on-premises and on sovereign cloud infrastructure, is uniquely
positioned to pursue FedRAMP readiness — but this requires specific
architectural decisions about system boundaries, deployment topology, and
continuous monitoring.

This ADR addresses the deployment architecture for three overlapping but
distinct certification frameworks:

1. **FedRAMP** (USA) — NIST SP 800-53 Rev5 controls, continuous monitoring,
   machine-readable authorization artifacts
2. **EUCS** (EU) — Basic/Substantial/High assurance levels, with evolving
   sovereignty requirements (proposed CADA framework)
3. **National sovereign cloud** — SecNumCloud (France), C5 (Germany), IRAP
   (Australia), ISMAP (Japan), and others catalogued in METR-0007 §11.2

The core architectural question is: how should Meteridian's components be
organized within a **certification boundary** so that the deploying
organization can achieve certification with minimal custom engineering?

Three approaches were considered:

1. **Monolithic boundary** — all Meteridian components (API, workers, database,
   cache, event store, blocks) are within a single authorization boundary
2. **Tiered boundary** — the control plane (API, policy engine, audit trail)
   is within the boundary; the data plane (block runtime, pipelines) can
   optionally extend outside
3. **Component-level boundaries** — each Meteridian component has its own
   security posture documentation, and the deployer composes them into their
   authorization boundary

## Decision

**Use a tiered boundary model (Option 2) with a well-defined "FedRAMP-ready
core" and optional "extended data plane."**

### Tier 1: FedRAMP-Ready Core (Authorization Boundary)

All components that handle control, authentication, audit, and compliance:

| Component | Role | NIST 800-53 Relevance |
|-----------|------|----------------------|
| API Gateway | Authentication, authorization, rate limiting | AC-*, IA-*, SC-* |
| Compliance Policy Engine | Policy evaluation, violation detection | CA-*, RA-*, SI-* |
| Audit Trail Service | Immutable event logging, hash chain | AU-*, SI-7 |
| Identity and Access Management | RBAC, tenant isolation | AC-*, IA-* |
| Configuration Management | Pipeline and block configuration | CM-* |
| PostgreSQL / TimescaleDB | Persistent data store (audit, config, billing data) | SC-28, MP-* |
| Valkey (Redis-compatible) | Session cache, real-time balance | SC-28 |
| Certificate and Key Management | Signing keys, TLS certificates, BYOK | SC-12, SC-13 |

### Tier 2: Extended Data Plane (Optionally Outside Boundary)

Components that process data but do not make authorization or compliance
decisions:

| Component | Role | Boundary Position |
|-----------|------|-------------------|
| Block Runtime | Pipeline execution | Inside boundary for FedRAMP; optionally outside for non-government |
| Redpanda Connect | Data collection/routing | Inside boundary for FedRAMP |
| External Tax Engines | Avalara, Vertex API calls | Outside boundary (FedRAMP CA-9 interconnection) |
| External Payment Providers | Stripe, Adyen, Paddle | Outside boundary (FedRAMP CA-9) |
| Marketplace (block distribution) | Block download/update | Outside boundary; air-gapped alternative for FedRAMP |

### Air-Gapped Mode

For FedRAMP High, DoD IL4/IL5, and classified environments, Meteridian supports
a fully air-gapped deployment:

- No outbound network connections from any component
- Blocks are loaded from local storage (no marketplace API calls)
- Exchange rates are loaded from local files (no ECB/RBI API calls)
- Time synchronization via local NTP/PTP (no public NTP)
- Certificate management via local PKI (no ACME/Let's Encrypt)
- The built-in tax calculator replaces external tax engine calls

## Rationale

### Why Not Monolithic Boundary (Option 1)?

Including the entire system (including all blocks, Redpanda, external
integrations) in a single authorization boundary:

- **Increases certification scope dramatically** — every block, every external
  API connection, and every data path must be documented and assessed
- **Makes the boundary fragile** — a single non-compliant block or integration
  invalidates the entire authorization
- **Is unnecessary** — FedRAMP allows "leveraged authorizations" and
  "interconnections" (CA-9) for components outside the boundary

### Why Not Component-Level Boundaries (Option 3)?

Per-component boundaries are technically sound but:

- **Shift too much burden to the deployer** — the deployer must compose 8+
  component-level authorizations into a coherent system authorization
- **Miss the "ready-to-certify" value proposition** — Meteridian's
  differentiator is that the compliance architecture is pre-built, not
  that components are individually certifiable
- **FedRAMP expects system-level boundaries** — FedRAMP SSPs (System Security
  Plans) describe a system, not individual components

### Tiered Model Benefits

- **Pragmatic boundary:** The FedRAMP-ready core contains the most sensitive
  components (auth, audit, policy) and excludes components that vary per
  deployment (blocks, external APIs)
- **Flexible deployment:** Non-government deployments can run the full system
  without boundary restrictions; government deployments constrain the data
  plane as needed
- **Progressive certification:** An operator can certify the core first
  (smaller scope, faster timeline) and expand the boundary later

## Consequences

### Positive

- **Clear certification scope:** The FedRAMP-ready core is a well-defined set
  of components with documented security controls
- **Air-gapped capability:** The tiered model naturally supports air-gapped
  deployment by removing Tier 2 external connections
- **Reusable for other frameworks:** The same boundary architecture applies to
  EUCS, SecNumCloud, C5, IRAP, and ISMAP — only the control mappings differ
- **Reduced certification cost:** Smaller boundary = fewer controls to
  document, fewer components to assess, faster ATO (Authority to Operate)

### Negative

- **Interconnection documentation:** Every connection between the core (Tier 1)
  and external services (Tier 2 external) requires a CA-9 interconnection
  agreement document
- **Operational complexity:** Government deployments may need separate
  infrastructure for Tier 1 (GovCloud, isolated VPC) while development/test
  environments use unrestricted infrastructure
- **Block trust implications:** Blocks running inside the FedRAMP boundary
  must undergo additional review (the capability-based security model from
  METR-0002 §11 is necessary but not sufficient for FedRAMP)

## Implementation Notes

### NIST SP 800-53 Control Mapping

Meteridian will provide a machine-readable control mapping (`controls.json`)
that maps NIST 800-53 Rev5 controls to specific Meteridian configurations:

```json
{
  "AU-2": {
    "control": "Event Logging",
    "implementation": "Audit trail service logs all events defined in METR-0008 §5.1",
    "evidence": "/api/v1/compliance/audit-trail?event_type=*",
    "automated": true
  },
  "AC-2": {
    "control": "Account Management",
    "implementation": "RBAC with tenant isolation; API key lifecycle management",
    "evidence": "/api/v1/compliance/reports/generate?type=access-management",
    "automated": true
  }
}
```

This JSON artifact can be consumed by GRC tools (Vanta, Drata, Secureframe)
and is aligned with FedRAMP's push toward machine-readable authorization
artifacts (2026 FedRAMP Consolidated Rules).

### Deployment Topology

```
┌─────────────────────────────────────────────────┐
│              Authorization Boundary               │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ API      │  │ Policy   │  │ Audit Trail  │   │
│  │ Gateway  │  │ Engine   │  │ Service      │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ IAM /    │  │ Config   │  │ Key Mgmt     │   │
│  │ RBAC     │  │ Mgmt     │  │ (BYOK/HSM)   │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
│  ┌──────────┐  ┌──────────┐                      │
│  │ Postgres │  │ Valkey   │                      │
│  │ /TSDB    │  │          │                      │
│  └──────────┘  └──────────┘                      │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │          Block Runtime (Constrained)         │ │
│  │  Only reviewed/approved blocks run here      │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │          Redpanda Connect                    │ │
│  │  Data collection within boundary             │ │
│  └─────────────────────────────────────────────┘ │
└─────────────┬───────────────────┬───────────────┘
              │ CA-9              │ CA-9
              ▼                   ▼
     ┌──────────────┐    ┌──────────────┐
     │ Tax Engines  │    │ Payment      │
     │ (Avalara,    │    │ Providers    │
     │  Vertex)     │    │ (Stripe,etc) │
     └──────────────┘    └──────────────┘
```

### EUCS and National Framework Alignment

The same boundary architecture maps to European and national frameworks:

| Framework | Tier 1 Mapping | Additional Requirements |
|-----------|---------------|------------------------|
| **EUCS High** | Authorization boundary = EUCS scope | EU-headquartered operator, EU data residency |
| **SecNumCloud** | Core + all data plane within French territory | French-law entity, immunity from extraterritorial laws |
| **C5** | BSI audit scope covers Tier 1 | IT-Grundschutz baseline |
| **IRAP** | IRAP assessment covers Tier 1 + relevant Tier 2 | ASD Essential Eight |
| **ISMAP** | ISMAP registration covers Tier 1 | Japanese data residency for government data |

### Continuous Monitoring

FedRAMP requires continuous monitoring (CA-7). Meteridian provides this through:

1. **Compliance policy engine** (METR-0008) continuously evaluates NIST 800-53
   controls
2. **Audit trail** captures all security-relevant events
3. **Vulnerability scanning integration** — hooks for Prisma Cloud, Aqua,
   Trivy, and other container security scanners
4. **POA&M tracking** — Plan of Action and Milestones management via the
   compliance reporting API

The compliance reporting API (METR-0008 §12.4) generates FedRAMP-format
continuous monitoring reports (monthly vulnerability scans, quarterly security
assessments, annual authorization renewal artifacts).
