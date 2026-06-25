# ADR-0016: E-Invoice Canonical Model and Format Selection

- **Status:** Accepted
- **Date:** 2026-06-19
- **Deciders:** @pgarciaq
- **Related:** METR-0009 (E-Invoicing Engine), METR-0007 (Legal and Regulatory Compliance)

## Context

Meteridian's native e-invoicing engine (METR-0009) needs an internal data model
that can represent invoices for any jurisdiction. The model must capture all
fields required by any e-invoicing mandate — from EU ViDA to India GST to
Brazil NF-e — while remaining manageable and not becoming a lowest-common-
denominator schema that satisfies no jurisdiction fully.

Three approaches were considered:

1. **EN 16931 as canonical model** — adopt the European e-invoicing standard
   as the internal representation, with jurisdiction-specific extensions for
   non-European formats
2. **UBL 2.1 as canonical model** — use the OASIS Universal Business Language
   XML schema directly as the internal model
3. **Custom canonical model** — design a Meteridian-specific invoice data model
   that maps to all target formats

## Decision

**Use EN 16931 as the semantic canonical model (Option 1), represented as Go
structs internally (not XML). Use UBL 2.1 as the primary syntactic binding
for XML output.**

The distinction matters:

- **EN 16931** is a _semantic_ standard — it defines the _information elements_
  (business terms) of an invoice, their cardinality, and their business rules,
  independent of any syntax (XML, JSON, PDF)
- **UBL 2.1** is a _syntactic_ standard — it defines how to serialize an
  invoice as XML. UBL 2.1 is one of two official syntactic bindings of EN 16931
  (the other is UN/CEFACT CII)

Meteridian uses EN 16931 semantics internally (Go structs with field names
matching EN 16931 business terms like BT-1, BT-2, BG-4, etc.) and generates
UBL 2.1 XML (or other formats) as output.

For non-European jurisdictions that use their own invoice schemas (Brazil NF-e,
Mexico CFDI, China Golden Tax fapiao), the canonical model includes extension
groups (prefixed `MG-*` for Meteridian Groups) that capture jurisdiction-
specific fields not present in EN 16931. Format generators for these
jurisdictions map from the canonical model (EN 16931 core + MG extensions) to
the jurisdiction-specific schema.

## Rationale

### Why Not UBL 2.1 Directly (Option 2)?

UBL 2.1 is an XML schema with 65+ document types and deeply nested structures.
Using it as the internal model would mean:

- **XML coupling:** The internal representation would be tied to XML semantics
  (namespaces, element ordering, attribute vs. element decisions) that are
  irrelevant for internal processing
- **Over-specification:** UBL 2.1 includes many document types beyond invoices
  (orders, despatch advice, etc.) that Meteridian does not need
- **Under-specification:** UBL 2.1 lacks Meteridian-specific fields (metering
  metadata, credit and prepaid information, rating details) that would need to be
  shoehorned into UBL extension mechanisms

EN 16931 is a semantic model — it defines what information an invoice must
contain, not how to serialize it. This gives Meteridian freedom to represent
invoices internally as efficient Go structs while generating UBL 2.1 XML (or
any other format) as output.

### Why Not Custom Model (Option 3)?

A custom model would require mapping to every target format without the benefit
of an existing standard. EN 16931 already has well-defined mappings to:

- UBL 2.1 (official binding)
- UN/CEFACT CII (official binding)
- Factur-X and ZUGFeRD (profile of EN 16931)
- Peppol BIS 3.0 (profile of EN 16931 + UBL 2.1)
- XRechnung (German CIUS of EN 16931)

For non-EN 16931 formats, a custom model would not have an advantage because
the mapping work is the same regardless of the source model.

### Why EN 16931 Specifically?

- **Most comprehensive:** EN 16931 was designed by CEN/TC 434 to be
  jurisdiction-agnostic while capturing the superset of requirements from EU
  Member States with diverse invoicing traditions
- **Extensible by design:** Core Information Elements (CIE) are mandated, but
  Additional Information Elements can be added through "extensions" and "CIUS"
  (Core Invoice Usage Specifications) — exactly the mechanism Meteridian uses
  for jurisdiction-specific fields
- **Active maintenance:** EN 16931 is under active revision; the TC 434
  committee continues to publish errata and new element lists
- **Regulatory foundation:** EN 16931 is mandated by EU Directive 2014/55/EU
  for all B2G invoicing in the EU, and will be the basis for the ViDA cross-
  border B2B mandate (July 2030)

## Consequences

### Positive

- **Single canonical model** serves all jurisdictions — format generators
  transform from one source, not N
- **Standard-aligned** — EN 16931 compliance comes free for EU markets
- **Future-proof** — as more jurisdictions align with EN 16931 (Australia and
  Singapore via Peppol, which uses EN 16931), Meteridian's model is natively
  compatible
- **Validation reuse** — EN 16931 business rules (CEN/TC 434 validation
  artifacts) can be used directly for European format validation
- **Community familiarity** — e-invoicing developers worldwide are familiar
  with EN 16931 terminology

### Negative

- **Mapping overhead for non-European formats** — Brazil NF-e, Mexico CFDI,
  and China Golden Tax have their own data models that do not align 1:1 with
  EN 16931. Format generators for these jurisdictions must handle non-trivial
  mappings. However, this mapping work would exist regardless of the canonical
  model choice.
- **Extension management** — Meteridian-specific extension groups (MG-*) are
  non-standard. Care must be taken to ensure they do not leak into standardized
  output formats.
- **EN 16931 evolution** — if EN 16931 makes breaking changes (unlikely given
  the EU mandate, but possible), Meteridian must track those changes.

## Implementation Notes

- The canonical model is defined as Go structs in `pkg/einvoice/model/`
- EN 16931 business term identifiers (BT-1, BG-4, etc.) are included as
  struct tags for documentation and validation reference
- Extension groups use a dedicated `Extensions` map to separate standard fields
  from Meteridian additions
- Format generators receive the canonical model via Arrow RecordBatch (block
  interface) or direct Go function call (in-process blocks)
- A reference mapping document maps each EN 16931 BT/BG to the corresponding
  Go struct field for contributor reference
