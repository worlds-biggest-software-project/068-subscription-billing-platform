# Subscription Billing Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source subscription billing platform unifying usage-based metering, ASC 606 revenue recognition, and intelligent dunning in one engine.

The Subscription Billing Platform is a developer-first billing system for SaaS, AI, and cloud-infrastructure companies that need flexible recurring, usage-based, and hybrid pricing without paying enterprise vendor pricing. It targets engineering and finance teams who today are forced to choose between a proprietary suite (Zuora, Chargebee, Stripe Billing) or to stitch together open-source metering with manual revenue recognition.

---

## Why Subscription Billing Platform?

- Enterprise platforms like Zuora start at ~$75K/year and can exceed $500K/year, with implementation timelines of 6–18 months — pricing out most growth-stage SaaS companies.
- Stripe Billing, Chargebee, and Recurly charge a percentage of billing volume (0.7%–0.75%) or thousands per month, scaling cost directly with revenue rather than usage.
- Lago is the only mature open-source option but lacks ASC 606 automation, AI dunning, and enterprise contract modelling; its AGPLv3 licence creates friction for ISVs embedding billing in commercial products.
- All current dunning systems use static retry schedules; per-customer behavioural learning to optimise retry timing and channel has not been shipped by any incumbent.
- Following Stripe's 2024 acquisitions of Metronome and Orb, the usage-billing stack is consolidating under a single proprietary vendor — leaving an open alternative underserved.

---

## Key Features

### Subscription & Pricing Engine

- Flexible subscription lifecycle: create, amend, pause, cancel, renew with proration and mid-cycle amendments
- Hybrid pricing models: flat recurring plus usage overage in a single invoice
- Graduated, package, percentage, volume, and tiered pricing rules
- Prepaid credit ledgers and wallet management for commit-based contracts
- Grandfathering and plan migration tools for pricing changes

### Real-Time Usage Metering

- Event ingestion and aggregation engine designed for high-throughput consumption pricing
- Idempotent event ingestion with replay guarantees
- Customer-facing real-time usage dashboards and invoice previews
- Pricing simulation engine to model revenue impact of pricing changes against historical data before rollout

### Revenue Recognition & Finance

- Deferred revenue schedules with period-end recognition reports
- ASC 606 / IFRS 15 automation including multi-element arrangement decomposition and variable-consideration handling
- Auditable journal entry generation
- SaaS metrics dashboards: MRR, ARR, churn, LTV, CAC cohort analysis
- ERP and accounting sync (NetSuite, QuickBooks, Xero)

### Dunning & Payments

- Configurable retry schedules, notification templates, and payment fallback routing
- Multi-gateway support (Stripe, Adyen, Braintree, GoCardless, PayPal)
- Hosted checkout pages and customer self-service portal for plan and payment-method management
- Multi-currency invoicing with tax line-item passthrough via external tax engines

### Integrations & API

- REST API with idempotent event ingestion, webhook emission, and comprehensive audit log
- Pre-built connectors for Salesforce, HubSpot, NetSuite, QuickBooks
- Self-hosted deployment with full data residency control

---

## AI-Native Advantage

AI is integrated where incumbents rely on static rules or manual finance work. Intelligent dunning learns per-customer payment behaviour — card type, time of day, retry history — to optimise retry timing and channel, targeting recovery rates above the 60–70% industry baseline. An AI engine decomposes multi-element ASC 606 contracts and allocates transaction prices, eliminating days of finance effort per period close. Pricing recommendation analyses usage patterns and willingness-to-pay signals to suggest tier structures, while churn prediction surfaces usage-decline and failed-payment risk weeks before cancellation.

---

## Tech Stack & Deployment

The platform is designed API-first with self-hosted (Docker / Kubernetes) and managed-cloud deployment options. Payment processing integrates with Stripe, Adyen, Braintree, GoCardless, and PayPal via tokenisation to maintain PCI DSS Level 1 posture without direct card storage. Standards alignment includes ASC 606 / IFRS 15 for revenue recognition, SEPA for European direct debit, ISO 20022 for international payment messaging, and SOC 2 Type II as a target operational baseline. Webhooks and a comprehensive REST API drive integration with downstream CRM, ERP, and data-warehouse systems.

---

## Market Context

The global subscription billing management market was valued at ~$9.16B in 2025 and is projected to reach $37.36B by 2035 (Astute Analytica), with cloud-based solutions representing roughly 75% of the market and usage-based billing growing fastest. Incumbent pricing ranges from 0.7% of volume (Stripe Billing) and ~$599/month (Maxio) to ~$75K/year (Zuora mid-market) and $200K–$500K+/year (Zuora enterprise). Primary buyers are SaaS founders and CTOs at seed–Series B companies, VPs of Finance at $5M–$50M ARR mid-market SaaS, enterprise revenue operations leaders, and AI/cloud infrastructure companies with high-volume metered billing.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
