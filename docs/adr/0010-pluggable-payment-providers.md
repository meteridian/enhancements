# ADR-0010: Pluggable Payment Providers with Paddle MoR Default

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Payment processing in the Meteridian ecosystem operates on two independent
dimensions that must be addressed simultaneously:

**Dimension 1: Marketplace payments.** Block developers who publish paid blocks
on the Meteridian marketplace need to receive payouts for their sales. The
platform must handle purchase transactions, calculate revenue splits, manage
global tax compliance (VAT, GST, sales tax across 100+ jurisdictions), process
refunds, and distribute payouts to developers worldwide.

**Dimension 2: Customer billing.** Meteridian customers (companies that use the
platform to meter and bill their own end users) need to connect their preferred
payment provider for invoice collection. Different customers use different
payment providers based on their geography, transaction volume, customer
demographics, and existing vendor relationships. A US SaaS company may prefer
Stripe; a European marketplace may prefer Adyen; a Southeast Asian platform may
need local payment methods that only regional providers support.

These two dimensions have different requirements. Marketplace payments need a
provider that handles global tax compliance and chargeback liability — the
Meteridian team should not be in the business of filing VAT returns in 30+
countries. Customer billing needs a pluggable interface where the customer
chooses and configures their own provider.

The payment processing industry has two fundamental models:

- **Payment processor** (Stripe, Adyen, PayPal): The merchant (Meteridian or
  its customer) is the merchant of record. They are responsible for tax
  compliance, chargeback disputes, and regulatory filings. The processor
  handles only the payment mechanics. Headline rates are lower but the true
  all-in cost includes tax software, chargeback losses, and compliance staff.

- **Merchant of Record (MoR)** (Paddle, LemonSqueezy): The MoR is the legal
  seller. They handle tax compliance, chargebacks, refunds, and regulatory
  filings on behalf of the actual software vendor. Headline rates are higher
  but the vendor has zero tax and chargeback burden. Total cost of ownership
  is often comparable to or lower than a payment processor when global sales
  are involved.

## Decision

Meteridian implements payment processing as a **pluggable block interface** and
makes specific default choices for each dimension:

**Dimension 1 (Marketplace): Paddle as default Merchant of Record.**

Paddle is the default payment provider for the block marketplace. As a Merchant
of Record, Paddle is the legal seller for marketplace transactions. This means:
- Paddle handles all global tax compliance (VAT, GST, sales tax in 200+
  countries/regions)
- Paddle assumes chargeback liability (not Meteridian, not the developer)
- Paddle handles refund processing
- Developers receive net payouts after Paddle's fee and platform revenue share

Fee structure: 5% + $0.50 per transaction (all-inclusive: tax, international
cards, chargebacks). Revenue split: **80% developer / 20% platform** (the 20%
platform share is taken from Paddle's net payout after fees).

Alternative marketplace providers (Stripe Connect, LemonSqueezy, Adyen for
Platforms) are available for developers in the Verified Partner tier or higher
who prefer faster payouts or lower effective rates at high volume.

**Dimension 2 (Customer billing): Standard PaymentProviderBlock interface.**

For customer billing, the payment provider is a standard Meteridian block type
with a defined Arrow input schema (rated invoices) and output schema (payment
results). Meteridian ships with four built-in implementations:

| Block | Provider | Strengths |
|-------|----------|-----------|
| `meteridian/stripe-payment` | Stripe | Broadest API, cards + ACH + SEPA, best for US SaaS |
| `meteridian/paddle-payment` | Paddle | MoR, zero tax burden, best for global SMB |
| `meteridian/adyen-payment` | Adyen | Interchange++, local payment methods, best for enterprise |
| `meteridian/paypal-payment` | PayPal | PayPal + Venmo + BNPL, best for consumer-facing |

Third-party developers can publish additional payment provider blocks on the
marketplace (regional providers, crypto payments, BNPL solutions, ERP billing
integrations).

The PaymentProviderBlock interface uses Arrow schemas for both input and output:
- **Input**: Invoice data (invoice_id, tenant_id, amount as `Decimal128`,
  currency, line items, billing address, payment method token)
- **Output**: Payment results (payment_status, provider_reference, settled
  amount, fees, error details)

This interface is intentionally minimal — it covers the common denominator of
all payment providers. Provider-specific features (Stripe subscriptions,
Adyen 3D Secure, Paddle checkout overlays) are exposed through provider-
specific configuration in the block's config section, not through schema
extensions.

## Consequences

### Positive

- **Zero tax compliance burden for marketplace**: Paddle handles VAT registration,
  filing, and remittance in every jurisdiction where marketplace blocks are sold.
  The Meteridian team does not need to become tax compliance experts or engage
  tax advisors in 30+ countries. For a startup-stage project, this is a massive
  reduction in operational complexity.

- **Predictable economics**: Paddle's 5% + $0.50 all-inclusive rate makes unit
  economics simple to model. There are no surprise fees for international cards,
  currency conversion, tax calculation, or chargeback disputes. The 80/20
  revenue split gives developers a clear, predictable income.

- **Pluggable customer billing**: Customers are not locked into a specific
  payment provider. They can switch from Stripe to Adyen by changing a block
  in their pipeline, without modifying any other part of their metering or
  rating logic. This flexibility is a selling point for enterprises with
  existing payment provider relationships.

- **Third-party payment innovation**: By making the payment provider a block
  interface, Meteridian enables payment innovation on the marketplace. A
  developer in Brazil can publish a PIX payment block; a crypto startup can
  publish a stablecoin settlement block; a BNPL provider can publish a
  buy-now-pay-later block. Meteridian does not need to integrate with every
  payment method — the ecosystem handles it.

### Negative

- **Paddle's 5% rate is above commodity processing rates**: For high-volume
  marketplace transactions, Paddle's 5% + $0.50 is significantly more expensive
  than Stripe Connect's 2.9% + $0.30 (before tax and chargeback costs). As
  marketplace volume grows, the economics may favor switching the default to
  Stripe Connect with a tax calculation add-on. The decision to start with
  Paddle is optimized for simplicity and speed-to-market, not for long-term
  cost optimization.

- **MoR model limits pricing flexibility**: As Merchant of Record, Paddle
  controls the checkout experience and pricing presentation. Custom checkout
  flows, complex subscription models, and usage-based pricing with real-time
  metering may be constrained by Paddle's capabilities. Stripe Connect provides
  more flexibility for complex billing scenarios.

- **Payment provider blocks handle sensitive data**: Payment method tokens,
  billing addresses, and transaction amounts flow through payment provider
  blocks. Third-party payment blocks on the marketplace must be held to the
  highest security standards. Only Tier 2+ (Verified Partner) blocks should be
  permitted to implement the PaymentProviderBlock interface, and they must
  undergo additional PCI-DSS compliance review.

- **Developer payout logistics**: Monthly net-30 payouts with a $50 minimum
  threshold mean small developers may wait months for their first payout. This
  could discourage early-stage marketplace participation. Consider reducing the
  minimum threshold to $10 for the first year to incentivize participation.

### Neutral

- The 80/20 revenue split is comparable to other developer marketplaces (Apple
  App Store: 70/30, Shopify App Store: 85/15 for first $1M, AWS Marketplace:
  varies). The 80/20 split positions Meteridian competitively while still
  generating meaningful platform revenue.

- Stablecoin settlement (USDC/USDT) for the DePIN ecosystem is experimental in
  v1. It provides an additional option for web3-native developers but is not
  expected to be a significant volume driver initially.

## Alternatives Considered

### Stripe Connect Only

Using Stripe Connect as the sole marketplace payment provider. Stripe Connect
supports multi-party payments with configurable revenue splits, developer
payouts (Express or Custom accounts), and 25+ payment methods.

**Rejected as the default because:** Stripe is a payment processor, not a
Merchant of Record. Using Stripe Connect means Meteridian (or each developer
individually) is the merchant of record and bears responsibility for:
- Tax compliance in every jurisdiction (requires Stripe Tax at +0.5% per
  transaction, plus registration and filing in each jurisdiction)
- Chargeback disputes ($15 per dispute, regardless of outcome)
- Refund processing and customer communication
- Regulatory filings (EU DAC7, US 1099-K)

For a startup-stage platform with global ambitions, the operational burden
of being merchant of record is prohibitive. Stripe Connect is offered as an
alternative for high-volume Verified Partners (Tier 2+) who prefer lower
per-transaction rates and faster payouts, and who are willing to handle their
own tax compliance.

Fee comparison for a $100 international transaction:

| Provider | Base fee | Int'l surcharge | Tax addon | Chargeback risk | Effective total |
|----------|---------|----------------|-----------|-----------------|----------------|
| Paddle (MoR) | 5% + $0.50 | Included | Included | $0 (Paddle's) | ~$5.50 |
| Stripe Connect | 2.9% + $0.30 | +1.5% | +0.5% | $0.50 avg* | ~$5.70 |

*Assumes 3% chargeback rate at $15 per dispute = ~$0.45 per transaction on average.

At this transaction size and dispute rate, Paddle and Stripe Connect are nearly
equivalent in total cost. Paddle wins on simplicity; Stripe wins on flexibility.

### Build Custom Payment Infrastructure

Building a custom payment orchestration layer that connects directly to
payment networks (Visa Direct, Mastercard Send) and handles tax compliance
in-house.

**Rejected because:** Custom payment infrastructure requires PCI DSS Level 1
compliance (annual audit, $50K-500K), money transmitter licenses in each
US state (~$500K-1M in legal fees), EU Payment Services Directive (PSD2)
authorization, and dedicated compliance staff. The total investment to
build and maintain custom payment infrastructure is $1-5M annually. This is
justified for companies with $100M+ in transaction volume (like Stripe itself)
but wildly disproportionate for a developer tools marketplace.

### Cryptocurrency Only

Using stablecoin (USDC, USDT) or cryptocurrency payments as the sole
marketplace payment method, eliminating traditional payment processing
entirely.

**Rejected as the sole method because:** Cryptocurrency-only payment alienates
the vast majority of enterprise and SMB customers who do not hold or use
cryptocurrency. Regulatory uncertainty in the US, EU, and Asia creates legal
risk for a platform that handles fiat-denominated billing data. Stablecoin
settlement is offered as an additional option for the DePIN ecosystem, but
traditional payment methods are required for mainstream adoption.

## References

- [Paddle](https://www.paddle.com/) — Merchant of Record for software
- [LemonSqueezy](https://www.lemonsqueezy.com/) — Alternative Merchant of Record
- [Stripe Connect](https://stripe.com/connect) — Multi-party payment processing
- [Adyen for Platforms](https://www.adyen.com/platforms) — Enterprise payment platform
- [Stripe Tax](https://stripe.com/tax) — Automated tax calculation and compliance
- [PCI DSS](https://www.pcisecuritystandards.org/) — Payment Card Industry Data Security Standard
- [EU DAC7](https://taxation-customs.ec.europa.eu/dac7_en) — EU platform reporting directive
