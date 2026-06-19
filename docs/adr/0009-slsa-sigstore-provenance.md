# ADR-0009: SLSA/Sigstore for Marketplace Block Provenance

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Meteridian's block marketplace (METR-0002) will host third-party processing
blocks that handle sensitive billing data: tenant identifiers, usage quantities,
monetary amounts, payment tokens, and invoice details. A compromised or
malicious block could exfiltrate financial data, manipulate billing amounts, or
inject backdoors into the metering pipeline.

The marketplace needs to answer three critical supply chain security questions
for every published block:

1. **Who built this block?** (Identity) — The block must be traceable to a
   verified developer or organization. Anonymous blocks with no attributable
   authorship are inherently higher risk.

2. **How was it built?** (Provenance) — The block must be linked to a specific
   source commit, built by a known build system, with a reproducible build
   process. This prevents scenarios where a developer publishes a binary that
   doesn't match the reviewed source code.

3. **Has it been tampered with?** (Integrity) — The block artifact must be
   cryptographically signed, and the signature must be verifiable by the
   platform and by users. The signature chain must be publicly auditable
   through a transparency log.

Traditional approaches to software signing (GPG keys, custom PKI) place
significant burden on both developers (key management, rotation, revocation)
and platform operators (certificate authority operation, key distribution).
The cloud-native ecosystem has converged on a better approach: **Sigstore**
for keyless signing with transparency logs, and **SLSA** for standardized
build provenance levels.

## Decision

Meteridian adopts three complementary standards for marketplace block supply
chain security:

**SLSA (Supply-chain Levels for Software Artifacts):**
SLSA defines four levels of supply chain security, from basic documentation
(Level 1) to hermetic, reproducible builds (Level 4). Meteridian requires
increasing SLSA levels for higher trust tiers:

| Trust Tier | SLSA Level | Requirements |
|-----------|-----------|-------------|
| Tier 0 (Unverified) | Level 1 | Build process documented |
| Tier 1 (Community) | Level 2 | Hosted build service, signed provenance |
| Tier 2 (Verified Partner) | Level 3 | Hardened build platform, reproducible builds |
| Tier 3 (Official) | Level 3+ | All of the above, plus Meteridian team review |

**Sigstore (Cosign, Fulcio, Rekor):**
- **Cosign** signs block artifacts (WASM modules, container images, source
  packages). Cosign supports keyless signing via OIDC identity providers
  (GitHub, Google, Microsoft), eliminating the need for developers to manage
  long-lived signing keys.
- **Fulcio** is a free certificate authority that issues short-lived signing
  certificates based on the developer's OIDC identity. Certificates are valid
  for minutes (not years), reducing the blast radius of key compromise.
- **Rekor** is a tamper-evident transparency log that records every signing
  event. Anyone can verify that a specific artifact was signed by a specific
  identity at a specific time. This makes supply chain attacks detectable:
  if an attacker publishes a malicious block under a compromised identity,
  the signing event is permanently recorded in Rekor and can be audited.

**in-toto Attestation Format:**
All provenance metadata is encoded in the in-toto attestation format, which
provides a standard structure for linking build inputs (source commits,
dependencies) to outputs (signed artifacts). This enables automated policy
enforcement: the marketplace can programmatically verify that a block was
built from a specific commit by a specific build system, without trusting
the developer's self-reported metadata.

**Continuous vulnerability scanning:**
Every published block is scanned with Trivy and Grype upon publication and
daily thereafter. Known CVEs block initial publication. Critical CVEs
discovered after publication trigger a 7-day remediation window; if the
developer does not publish a patched version, the block is automatically
delisted from the marketplace (existing installations continue running but
with a security warning).

## Consequences

### Positive

- **Keyless signing eliminates developer friction**: Developers sign blocks
  using their existing GitHub, Google, or Microsoft identities. No GPG key
  generation, no key distribution, no key rotation ceremonies. This removes
  the single biggest barrier to signed software: key management.

- **Transparent audit trail**: Every signing event is recorded in Rekor's
  public transparency log. Security researchers, customers, and automated
  systems can verify the complete signing history of any block. This makes
  supply chain attacks detectable even after the fact.

- **Tiered security scales with trust**: Tier 0 blocks (anyone can publish)
  have minimal security requirements but are sandboxed to WASM with no
  capabilities. Tier 2+ blocks have strong provenance requirements but gain
  access to broader capabilities. This gradient encourages developers to
  invest in supply chain security as they grow their block's user base.

- **Industry alignment**: SLSA, Sigstore, and in-toto are the emerging
  standards for cloud-native supply chain security. They are backed by the
  Linux Foundation, adopted by major package registries (npm, PyPI, Maven
  Central), and supported by CI/CD platforms (GitHub Actions, GitLab CI).
  Using these standards ensures compatibility with the broader ecosystem and
  avoids vendor lock-in.

- **Reproducible builds for Tier 2+**: Requiring reproducible builds for
  Verified Partners means that anyone can verify a block's binary matches its
  source code. This is the gold standard for open-source security and
  provides the strongest assurance short of formal verification.

### Negative

- **SLSA Level 3 is hard**: Reproducible builds require deterministic
  toolchains, pinned dependencies, and hermetic build environments. Most
  developers have never built reproducible binaries. Meteridian must provide
  build templates (GitHub Actions workflows, CI/CD configs) that achieve
  SLSA Level 3 out of the box to make this practical.

- **Sigstore infrastructure dependency**: Fulcio and Rekor are external
  services operated by the Sigstore project. If these services experience
  downtime, block publishing is blocked. Meteridian can mitigate this by
  running a private Sigstore instance for Tier 3 (Official) blocks, but
  community blocks will depend on the public infrastructure.

- **Vulnerability scanning false positives**: Trivy and Grype report
  vulnerabilities in dependencies that may not be exploitable in the block's
  context. WASM blocks that don't use networking may flag a network library
  vulnerability that cannot be exploited. A manual review process for
  disputed vulnerabilities is needed, adding operational overhead.

- **Delisting disruption**: Automatically delisting blocks after 7 days of
  unpatched critical CVEs may disrupt production pipelines that depend on the
  block. The 7-day window is a compromise between security and stability. A
  "pin version despite CVE" escape hatch for enterprise customers may be
  needed.

### Neutral

- The signing infrastructure (Cosign, Fulcio, Rekor) is open-source and can
  be self-hosted for air-gapped or enterprise deployments. Meteridian will
  provide Helm charts for private Sigstore deployment.

- SLSA and Sigstore are already adopted by the container ecosystem (Docker
  Content Trust v2, OCI image signing). Extending them to WASM modules and
  source packages is a natural evolution.

## Alternatives Considered

### GPG Signing Only

Traditional GPG/PGP signing where each developer generates a keypair and the
marketplace verifies signatures against published public keys.

**Rejected because:** GPG key management is notoriously painful. Developers
must generate keys, distribute public keys, handle key rotation, deal with
key compromise (revocation certificates), and manage key expiration. In
practice, most developers never rotate keys, use weak key parameters, and
store private keys insecurely. There is no transparency log — if a key is
compromised, there is no public record of what was signed and when. GPG
signing also provides no build provenance: a developer can sign a binary
that was built from a different commit than the reviewed source.

The Sigstore approach (keyless signing via OIDC + transparency log) provides
strictly stronger security guarantees with less developer friction.

### Custom PKI (Platform-Managed Certificate Authority)

Operating a Meteridian-specific certificate authority that issues signing
certificates to registered developers.

**Rejected because:** Running a certificate authority is an enormous
operational undertaking. It requires HSMs for key protection, a certificate
revocation infrastructure (CRL or OCSP), a registration authority for
identity verification, disaster recovery procedures, and ongoing compliance
audits. Enterprises that have operated internal CAs consistently report
high costs and operational complexity. Sigstore provides all of these
capabilities as a managed service (public) or as a deployable open-source
stack (private), without requiring Meteridian to become a CA operator.

### Docker Content Trust (Notary v2)

Using the OCI distribution specification's signature support (Notary v2) for
signing block artifacts.

**Rejected because:** Notary v2 is designed specifically for OCI container
images. Meteridian blocks include WASM modules and source packages that are
not OCI images. While we could package everything as OCI artifacts (which
Notary v2 can sign), this adds unnecessary complexity and couples the
signing mechanism to the OCI ecosystem. Cosign (from Sigstore) supports
signing arbitrary blobs — WASM files, tarballs, and OCI images — providing
a more flexible foundation. Additionally, Cosign integrates with the Rekor
transparency log, which Notary v2 does not.

## References

- [SLSA](https://slsa.dev/) — Supply-chain Levels for Software Artifacts
- [Sigstore](https://www.sigstore.dev/) — Keyless signing and verification
- [Cosign](https://docs.sigstore.dev/cosign/overview/) — Container and artifact signing
- [Fulcio](https://docs.sigstore.dev/fulcio/overview/) — Certificate authority for code signing
- [Rekor](https://docs.sigstore.dev/rekor/overview/) — Transparency log
- [in-toto](https://in-toto.io/) — Supply chain integrity framework
- [Trivy](https://trivy.dev/) — Comprehensive vulnerability scanner
- [Grype](https://github.com/anchore/grype) — Vulnerability scanner for containers and filesystems
- [SLSA GitHub Actions Generator](https://github.com/slsa-framework/slsa-github-generator) — SLSA provenance for GitHub Actions
