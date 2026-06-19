# ADR-0014: Cryptographic Audit Trail Architecture

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq
- **Related:** METR-0008 (Compliance-as-Code), METR-0007 (Legal and Regulatory Compliance)

## Context

Meteridian must provide an immutable audit trail that satisfies regulatory
requirements across jurisdictions (GoBD in Germany, SOX in the US, GST in
India, ZATCA in Saudi Arabia — see METR-0007 §8). The audit trail must be
tamper-evident: any modification or deletion of historical entries must be
detectable without relying on trust in the platform operator.

Three approaches were considered:

1. **Append-only table with no integrity verification** — simple PostgreSQL/
   TimescaleDB table with no DELETE/UPDATE permissions. Used by Kill Bill and
   most billing platforms.
2. **Hash chain (sequential linking)** — each entry includes the hash of the
   previous entry, forming a linked chain. Tampering with any entry invalidates
   all subsequent hashes.
3. **Merkle tree** — entries are organized in a binary tree structure where each
   non-leaf node is the hash of its children. Used by Certificate Transparency
   logs and blockchain systems.

## Decision

**Use hash chains (Option 2) with periodic external anchoring.**

Each audit entry includes a `previous_hash` field containing the SHA-256 hash
of the immediately preceding entry. The `entry_hash` is computed as:

```
entry_hash = SHA-256(entry_id || timestamp || event_type || actor ||
                     resource || action || details || previous_hash)
```

Hash chains are maintained **per-tenant** to allow independent verification
of each tenant's audit trail without exposing cross-tenant data.

Every N entries (configurable, default 10,000) or every T hours (configurable,
default 24), a **checkpoint** is created:

1. The checkpoint hash is submitted to an external timestamping service
   (RFC 3161 TSA or a blockchain anchoring service)
2. The timestamping proof is stored alongside the checkpoint entry
3. This provides non-repudiation: even if the database operator tampers with
   entries between checkpoints, the external anchor proves the chain state at
   checkpoint time

For air-gapped deployments, external timestamping is optional. The hash chain
itself provides tamper detection; the external anchor adds non-repudiation
against the operator.

## Rationale

### Why Not Append-Only Only (Option 1)?

Append-only tables prevent deletion through database permissions, but:

- A database administrator can bypass permissions (ALTER TABLE, direct file
  manipulation, restore from modified backup)
- There is no way for an auditor to verify that no entries have been modified
  without comparing against an independent copy
- This is the approach used by Kill Bill, and it is adequate for many use cases,
  but it does not meet the "tamper-evident" requirement of GoBD or SOX

### Why Not Merkle Trees (Option 3)?

Merkle trees offer superior properties for selective verification (prove that a
specific entry exists without revealing other entries), but:

- **Complexity:** Merkle tree maintenance requires rebalancing or fixed-size
  leaf batching. For a high-throughput audit trail (100K+ entries/second),
  tree management adds non-trivial overhead.
- **Sequential verification is the primary use case.** Auditors typically want
  to verify a contiguous range of entries (e.g., "all invoices in Q3 2026"),
  not selective membership proofs. Hash chains are optimal for sequential
  verification.
- **Certificate Transparency use case is different.** CT logs need to prove
  membership of a specific certificate in a large set. Meteridian's audit trail
  needs to prove that a contiguous sequence of events has not been modified.
- **Hash chains can be upgraded to Merkle trees later** if selective
  verification becomes a requirement. The entry hash computation is compatible
  with both structures.

### Why Per-Tenant Chains?

- **Data isolation:** Tenants can verify their own audit trail without
  accessing other tenants' data
- **Performance:** Chain computation is parallelizable across tenants
- **Data residency:** Per-tenant chains allow audit data to be region-pinned
  without cross-region hash dependencies

### Why SHA-256?

- Universally accepted by regulatory authorities
- No known practical attacks
- Fast in software (hardware acceleration via SHA-NI on modern CPUs)
- If a jurisdiction requires a different algorithm (e.g., GOST R 34.11-2012
  for Russia), the hash function can be made configurable per-tenant without
  changing the chain structure

## Consequences

### Positive

- **Tamper detection without trust:** Any party can verify the audit trail by
  recomputing the hash chain
- **External non-repudiation:** RFC 3161 timestamps prove chain state at
  specific points in time
- **Regulatory satisfaction:** Meets GoBD (tamper-evident), SOX (audit trail
  integrity), ZATCA (invoice hash verification) requirements
- **Low overhead:** SHA-256 computation adds <0.1ms per entry

### Negative

- **Sequential bottleneck:** Hash chain computation within a tenant is
  inherently sequential (each entry depends on the previous). This is mitigated
  by batching: compute hashes for a batch of entries using the last committed
  hash as the starting point, then persist the batch atomically.
- **Recovery complexity:** If the database is corrupted mid-chain, recovery
  requires replaying from the last valid checkpoint. This is acceptable because
  audit trail corruption is an exceptional event, not a normal operational
  scenario.
- **Checkpoint cost:** External timestamping services charge per timestamp
  (typically $0.01-$0.10 per stamp). At default settings (every 24h per
  tenant), this is negligible for most deployments.

### Storage Overhead

Each audit entry adds ~100 bytes for the hash fields (`previous_hash` and
`entry_hash` as 64-character hex strings). At 100K entries/second, this is
~10MB/second of additional storage — manageable within TimescaleDB's
compression capabilities.

## Implementation Notes

- Hash chain state (last committed hash + sequence number) is maintained in
  Valkey for sub-millisecond reads during chain extension
- Persistent state is in PostgreSQL and TimescaleDB for durability
- Batch writes use PostgreSQL transactions to ensure atomicity of
  hash-chain-linked entry groups
- The verification API recomputes hashes over a requested range and compares
  against stored values
