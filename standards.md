# Standards & API Reference

> Project: Subscription Billing Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### Accounting & Revenue Recognition Standards

**ASC 606 — Revenue from Contracts with Customers (US GAAP)**
- Issuer: Financial Accounting Standards Board (FASB)
- URL: https://www.fasb.org/page/PageContent?pageId=/standards/asc606.html
- The five-step model for recognising revenue when performance obligations are satisfied. Subscription billing platforms must support deferred revenue schedules, variable consideration, and multi-element arrangement splitting to be compliant. Non-compliant handling is a common audit finding at Series B+ SaaS companies.

**IFRS 15 — Revenue from Contracts with Customers**
- Issuer: International Accounting Standards Board (IASB)
- URL: https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/
- The international counterpart to ASC 606, developed jointly with FASB and using the same five-step model. Required for companies reporting under IFRS (most non-US public companies). Subscription billing platforms targeting global enterprise buyers must generate IFRS 15-compliant revenue schedules.

### Payment Security Standards

**PCI DSS v4.0.1 — Payment Card Industry Data Security Standard**
- Issuer: PCI Security Standards Council (PCI SSC)
- URL: https://www.pcisecuritystandards.org/document_library/
- The mandatory security framework for any entity storing, processing, or transmitting cardholder data. Version 4.0.1 introduced future-dated requirements that became mandatory on 31 March 2026. Tokenisation is recommended (not mandated) to reduce PCI scope. Billing platforms typically achieve Level 1 PCI certification via third-party payment gateways and vault tokenisation. Key requirements include FIPS-validated crypto modules, key rotation schedules, and network segmentation.

**PCI SSC Tokenisation Guidelines**
- Issuer: PCI Security Standards Council
- URL: https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement.pdf
- Supplementary guidance on implementing tokenisation as a scope-reduction tool. Specifies that recovering the original PAN from a token must be computationally infeasible, and that tokenisation systems must be isolated from out-of-scope networks.

### Payment Messaging & Transfer Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- Issuer: International Organization for Standardization
- URL: https://www.iso20022.org/
- The global standard for financial messaging, replacing legacy SWIFT MT formats. As of November 2025, 75%+ of SWIFT market infrastructure traffic and 97%+ of payments use ISO 20022. Structured country codes and town names in payment instructions are mandatory by end of 2026. Relevant for subscription billing platforms processing international bank transfers or integrating with bank rails directly.

**ISO 4217 — Currency Codes**
- Issuer: International Organization for Standardization
- URL: https://www.iso.org/iso-4217-currency-codes.html
- Three-letter alpha codes and three-digit numeric codes for all world currencies. Every reputable payment processor, accounting platform, and ERP uses ISO 4217 alpha-3 as the wire format (e.g. `USD`, `EUR`, `GBP`). Multi-currency billing platforms must store and transmit amounts paired with their ISO 4217 code.

**SEPA — Single Euro Payments Area**
- Issuer: European Payments Council (EPC)
- URL: https://www.europeanpaymentscouncil.eu/
- Framework governing euro-denominated direct debits and credit transfers within the EEA. SEPA Direct Debit (SDD) is the primary mechanism for recurring bank-based subscription collection in Europe. Platforms targeting European SMBs must support SEPA mandates and mandate management.

### European Regulatory Standards

**PSD2 — Revised Payment Services Directive (EU 2015/2366)**
- Issuer: European Parliament / European Banking Authority (EBA)
- URL: https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money
- Mandates Strong Customer Authentication (SCA) for electronic payments within the EU/EEA. Recurring subscription payments are exempt from SCA after the first authenticated transaction, provided the amount and merchant do not change. Billing platforms must ensure the initial payment captures SCA, and subsequent automated charges use the MIT (Merchant-Initiated Transaction) exemption pathway.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- Issuer: European Parliament / European Data Protection Board
- URL: https://gdpr-info.eu/
- Governs collection, storage, and processing of personal data for EU/EEA individuals. Financial records (billing addresses, transaction histories, card metadata) are personal data under GDPR. Data retention: financial records must be retained seven years in most EU jurisdictions; personal data must be deleted when no longer needed. Billing platforms must implement right-to-erasure workflows that respect conflicting financial retention obligations.

### E-Invoicing Standards

**Peppol BIS Billing 3.0**
- Issuer: OpenPeppol / European Commission
- URL: https://docs.peppol.eu/poacc/billing/3.0/
- European standard for structured electronic invoices, based on EN 16931 and built on UBL 2.1 XML. Mandatory for B2B invoicing in Belgium from 1 January 2026; similar mandates are rolling out across the EU. Subscription billing platforms targeting European B2B customers need to generate Peppol-compliant invoice XML.

**OASIS UBL 2.1 — Universal Business Language**
- Issuer: OASIS Open
- URL: https://docs.oasis-open.org/ubl/UBL-2.1.html
- XML vocabulary for business documents including invoices, credit notes, and order documents. Forms the technical foundation of Peppol BIS Billing 3.0 and EN 16931. Used by governments and large enterprise buyers in procurement workflows.

**ISO 9735 / UN/EDIFACT — Electronic Data Interchange**
- Issuer: ISO / UN/ECE
- URL: https://www.iso.org/standard/17592.html
- Legacy EDI standard still required for invoicing integrations with large enterprise ERPs (SAP, Oracle). The EDIFACT INVOIC message type is the structured format for electronic invoices in traditional B2B supply chains. Less relevant for modern SaaS billing but required for enterprise integrations.

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- Issuer: IETF
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework for third-party API access. All major subscription billing platforms (Stripe, Chargebee, Zuora) use OAuth 2.0 for partner and marketplace integrations. An AI-native billing platform should expose OAuth 2.0 endpoints for CRM, ERP, and accounting system integrations.

**RFC 7519 — JSON Web Token (JWT)**
- Issuer: IETF
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Compact, URL-safe token format for representing authenticated claims. Used in combination with OAuth 2.0 for stateless API authentication. Billing platform APIs should issue and validate JWTs for service-to-service calls and webhook signature verification.

**RFC 7807 — Problem Details for HTTP APIs**
- Issuer: IETF
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard JSON error response format for HTTP APIs. Billing APIs should adopt this format for consistent error reporting across subscription lifecycle events (payment failures, proration errors, limit breaches).

**OpenAPI Specification 3.1 (OAS 3.1)**
- Issuer: OpenAPI Initiative / Linux Foundation
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de-facto standard for describing REST APIs, including webhook definitions (introduced in OAS 3.1). Billing platforms should publish an OpenAPI 3.1 spec to enable automatic SDK generation, documentation, and integration validation. Paddle and Lago both publish OpenAPI specs for their billing APIs.

### Event & Metering Standards

**CloudEvents v1.0 (CNCF Graduated)**
- Issuer: Cloud Native Computing Foundation (CNCF) Serverless Working Group
- URL: https://cloudevents.io/
- Vendor-neutral specification for describing event data in a common format, enabling interoperability across services, platforms, and cloud providers. Graduated CNCF project (January 2024). Usage-metering systems (OpenMeter, Lago) adopt CloudEvents as the ingestion format for billing events. An AI-native billing platform should accept CloudEvents-formatted usage events for idempotent metering at scale.

---

## Similar Products — Developer Documentation & APIs

### Stripe Billing

- **Description:** Payment-native subscription and usage billing platform built on Stripe's payment infrastructure. Powers a significant share of SaaS subscription billing globally.
- **API Documentation:** https://docs.stripe.com/billing/billing-apis
- **Full API Reference:** https://docs.stripe.com/api
- **Webhooks Guide:** https://docs.stripe.com/billing/subscriptions/webhooks
- **SDKs/Libraries:** Ruby, Python, PHP, Java, Node.js, Go, .NET — https://docs.stripe.com/sdks
- **Developer Guide:** https://docs.stripe.com/billing
- **Standards:** REST/JSON; OpenAPI 3.1 spec available; webhook signatures via HMAC-SHA256
- **Authentication:** API key (Bearer token); OAuth 2.0 for Connect integrations
- **Notes:** Webhooks use Stripe-Signature header with timestamp tolerance (default 5 minutes). Current API version: 2026-04-22.

### Chargebee

- **Description:** Mid-market subscription billing platform with strong revenue recognition (ASC 606/IFRS 15), dunning management, and a comprehensive REST API.
- **API Documentation:** https://apidocs.chargebee.com/docs/api/
- **Getting Started:** https://apidocs.chargebee.com/docs/api/getting-started
- **Authentication Docs:** https://apidocs.chargebee.com/docs/api/auth
- **SDKs/Libraries:** Node.js, Python, PHP, Java, Go, Ruby, .NET; framework adapters for Laravel and Next.js
- **Developer Guide:** https://www.chargebee.com/docs/2.0/developer_resources.html
- **Standards:** REST/JSON; HTTP Basic Auth; form-encoded request bodies
- **Authentication:** HTTP Basic Auth using API key as username; environment-specific keys (test/live)

### Recurly

- **Description:** Subscription billing platform with best-in-class dunning and churn recovery tools. Native mobile SDK support for Android and iOS.
- **API Documentation:** https://recurly.com/developers/api/
- **Developer Hub:** https://recurly.com/developers/
- **Full Docs:** https://docs.recurly.com/
- **SDKs/Libraries:** Node.js (v4.74.0+), additional language libraries listed on developer hub; native mobile SDKs for Android and iOS
- **Developer Guide:** https://recurly.com/developers/guides/manage-subscription.html
- **Standards:** REST/JSON; webhooks via POST to registered endpoints
- **Authentication:** API keys; REST API keys documented at https://docs.recurly.com/recurly-subscriptions/docs/api-keys

### Paddle

- **Description:** Merchant-of-record billing platform handling payments, global tax (VAT/GST), fraud, and compliance. Developer portal includes an MCP server for AI-assistant integration.
- **API Documentation:** https://developer.paddle.com/api-reference/overview
- **Developer Home:** https://developer.paddle.com/
- **SDKs/Libraries:** Go, Node.js, PHP, Python — https://developer.paddle.com/resources/overview
- **Postman Collection:** https://www.postman.com/paddlehq/paddle-billing/overview
- **MCP Server:** https://developer.paddle.com/changelog/2026/docs-mcp
- **Standards:** REST/JSON; OpenAPI 3.1 spec available for download; webhooks documented in spec
- **Authentication:** API key (Bearer); client-side tokens for checkout

### Zuora

- **Description:** Enterprise subscription management and revenue recognition platform. Full order-to-revenue lifecycle including complex contract modifications, amendments, and multi-element arrangements.
- **API Documentation:** https://developer.zuora.com/v1-api-reference/
- **Developer Center:** https://developer.zuora.com/
- **Product Docs:** https://docs.zuora.com/en/zuora-platform/integration/apis/rest-api
- **SDKs/Libraries:** Java, Node.js, Python, C# — examples included in API reference
- **Developer Guide:** https://developer.zuora.com/docs/get-started/api-tutorials/
- **Standards:** REST/JSON; OAuth 2.0 for authentication; OpenAPI spec available
- **Authentication:** OAuth 2.0 (client credentials flow); requires creating an OAuth client in the Zuora UI first

### Maxio (formerly Chargify + SaaSOptics)

- **Description:** Subscription billing and B2B revenue operations platform for scaling SaaS companies. Merged product from Chargify (billing) and SaaSOptics (financial reporting).
- **API Documentation:** https://developers.maxio.com/http/getting-started/overview
- **Developer Portal:** https://developers.maxio.com/
- **Product Docs:** https://docs.maxio.com/
- **Developer Guide:** https://docs.maxio.com/hc/en-us/articles/28271323360397-Developer-One-Sheet
- **Standards:** REST/JSON; API powered by APIMatic
- **Authentication:** API keys — https://docs.maxio.com/hc/en-us/articles/24294819360525-Advanced-Billing-API-Keys

### Lago

- **Description:** Open-source metering and usage-based billing platform. Self-hostable (Docker Compose) or managed cloud. The only mature open-source option in the subscription billing space.
- **API Documentation:** https://getlago.com/docs/api-reference/intro
- **Developer Guide:** https://getlago.com/docs/guide/introduction/welcome-to-lago
- **GitHub:** https://github.com/getlago/lago
- **SDKs/Libraries:** Node.js (`lago-javascript-client`), Python (`lago-python-client`), Ruby (`lago-ruby-client`), Go (`lago-go-client`)
- **Standards:** REST/JSON; full OpenAPI specification available; CloudEvents-compatible event ingestion
- **Authentication:** API key (Bearer token)
- **Licence:** AGPLv3 (self-hosted); commercial cloud offering available

### Orb

- **Description:** Usage-based billing engine purpose-built for complex metered pricing — tiered, graduated, per-seat, and hybrid models. Aimed at AI, API, and infrastructure companies.
- **API Documentation:** https://docs.withorb.com/overview
- **Homepage:** https://www.withorb.com/
- **SDKs/Libraries:** TypeScript/JavaScript (`orb-billing` on npm — includes TypeScript type definitions); Python, Go available
- **Standards:** REST/JSON; OpenAPI spec available; full API reference in `api.md` within SDK packages
- **Authentication:** API key

### OpenMeter

- **Description:** Open-source real-time metering and billing engine for AI, API, and DevOps usage-based billing. Ingests CloudEvents at high volume (250K+ events/second) and integrates with billing providers.
- **API Documentation:** https://openmeter.io/docs/
- **GitHub:** https://github.com/openmeterio/openmeter
- **SDKs/Libraries:** Node.js, Python, Go; OpenAPI spec available
- **Standards:** REST/JSON; CloudEvents v1.0 for event ingestion; OpenAPI 3.1 spec
- **Authentication:** API key
- **Licence:** Apache 2.0

---

## Notes

**Emerging standards to watch:**

- **OAuth 2.1** — A consolidation of OAuth 2.0 best practices currently in IETF draft. Removes implicit and password grant flows; mandates PKCE. MCP (Model Context Protocol) has adopted OAuth 2.1 for AI agent authorization, making it relevant for billing platforms integrating with AI agents.

**Usage-metering gap:** There is no formal ISO or W3C standard specifically for subscription metering or dunning workflows. CloudEvents (CNCF) is the closest broadly-adopted specification for billing event ingestion, but aggregation logic, retry policies, and proration methods remain unstandarised and vary significantly between platforms.

**E-invoicing acceleration:** EU member states are rapidly mandating structured electronic invoicing (Peppol BIS 3.0 / EN 16931). Any billing platform targeting European enterprise customers should prioritise UBL 2.1 / Peppol invoice generation, particularly given the Belgian B2B mandate effective January 2026 and similar mandates in other EU member states through 2027.

**PCI DSS 4.0.1 timeline:** The final tranche of future-dated PCI DSS 4.0.1 requirements became mandatory 31 March 2026. Platforms that have not completed their Level 1 assessment under v4.0.1 are now out of compliance.
