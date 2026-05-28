# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Subscription Billing Platform · Created: 2026-05-12

## Philosophy

The hybrid relational + JSONB model takes a pragmatic middle ground: core billing entities (customers, subscriptions, invoices, payments) are fully relational with typed columns and foreign keys, while inherently variable data (pricing configurations, jurisdiction-specific fields, usage event properties, plan metadata) is stored in PostgreSQL JSONB columns with GIN indexes. This approach recognizes that subscription billing spans a spectrum from highly structured (invoice line items, payment amounts) to highly variable (pricing tiers that differ per plan, tax rules that differ per jurisdiction, event properties that differ per meter).

This is the pattern used by Lago (PostgreSQL with JSONB for pricing rules), Stripe (hybrid relational with flexible metadata fields), and many modern SaaS platforms. It avoids the table explosion of full normalization while retaining referential integrity for the relationships that matter most. New pricing models, custom fields, and jurisdiction-specific requirements can be added without schema migrations -- just extend the JSONB structure and update the application validation.

The key insight is that a subscription billing platform must accommodate pricing models that don't exist yet. When a customer asks for a pricing structure that combines percentage-of-revenue with tiered usage and committed minimums, a fully normalized schema needs new tables and migrations. A hybrid schema needs a new JSONB structure in the `pricing_config` column and updated application logic. For a platform targeting rapid iteration on pricing models, this flexibility is decisive.

**Best for:** Teams building a fast-moving billing platform that needs to support diverse and evolving pricing models across multiple jurisdictions without constant schema migrations.

**Trade-offs:**
- (+) Core relationships enforced relationally; variable data flexible in JSONB
- (+) No schema migration needed for new pricing models, custom fields, or jurisdiction rules
- (+) Fewer tables than normalized model (30-40% reduction)
- (+) GIN indexes on JSONB enable fast queries on variable fields
- (+) PostgreSQL-native: no additional infrastructure beyond a standard PG deployment
- (+) Fast MVP development: new features often just add JSONB fields
- (-) JSONB fields lack database-enforced constraints (validation pushed to application)
- (-) JSONB fields are harder to query with standard SQL reporting tools
- (-) Schema evolution of JSONB structures must be managed in application code
- (-) Developers need to understand when to use JSONB vs. relational columns
- (-) Deep JSONB nesting can make queries complex and harder to optimize

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue contracts and performance obligations are relational; variable consideration rules stored as JSONB configuration |
| ISO 4217 | Currency as typed `CHAR(3)` column on all monetary tables |
| ISO 3166-1 | Country codes in customer addresses; jurisdiction JSONB includes ISO 3166 subdivision codes |
| CloudEvents v1.0 | Usage events stored with CloudEvents envelope fields; event `properties` in JSONB |
| PCI DSS v4.0.1 | Payment methods store only gateway tokens; card metadata as structured relational fields |
| SEPA | SEPA mandate details in `payment_methods.gateway_config` JSONB |
| Peppol BIS 3.0 | E-invoicing metadata in `invoices.e_invoice_config` JSONB, adaptable per jurisdiction |
| GDPR | Tenant-level JSONB `data_retention_config` defines per-jurisdiction retention rules |

---

## Tenant & Customer Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'cancelled')),
    -- Flexible tenant configuration in JSONB
    billing_config  JSONB NOT NULL DEFAULT '{}',
    -- Example billing_config:
    -- {
    --   "default_currency": "USD",
    --   "supported_currencies": ["USD", "EUR", "GBP"],
    --   "invoice_number_format": "INV-{YYYY}-{SEQ:6}",
    --   "net_terms_days": 30,
    --   "auto_collection": true,
    --   "dunning_campaign_id": "dc_abc",
    --   "tax_provider": "avalara",
    --   "tax_provider_config": {"account_id": "...", "license_key": "..."}
    -- }
    data_retention_config JSONB NOT NULL DEFAULT '{}',
    -- Example: {"default_days": 2555, "gdpr_pii_days": 365, "financial_records_days": 2555}
    feature_flags   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ai_dunning_enabled": true, "asc606_automation": false, "peppol_invoicing": true}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    external_id     TEXT,
    name            TEXT NOT NULL,
    email           TEXT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    locale          TEXT DEFAULT 'en',
    -- Structured address with ISO 3166 codes
    billing_address JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Main St",
    --   "line2": "Suite 400",
    --   "city": "San Francisco",
    --   "state": "CA",
    --   "postal_code": "94105",
    --   "country_code": "US"          -- ISO 3166-1 alpha-2
    -- }
    tax_config      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "tax_id": "DE123456789",
    --   "tax_id_type": "eu_vat",
    --   "tax_exempt": false,
    --   "tax_exemption_certificate": null
    -- }
    -- Open-ended custom fields
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_email ON customers(tenant_id, email);
CREATE INDEX idx_customers_metadata ON customers USING gin(metadata jsonb_path_ops);
```

## Product Catalog with Flexible Pricing

```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived')),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_tenant ON products(tenant_id);

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    name            TEXT NOT NULL,
    description     TEXT,
    billing_period  TEXT NOT NULL CHECK (billing_period IN ('monthly', 'quarterly', 'semi_annual', 'annual', 'custom')),
    billing_interval_months INT NOT NULL DEFAULT 1,
    trial_days      INT DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'grandfathered')),
    -- THE KEY INNOVATION: pricing as JSONB
    -- This single JSONB column replaces plan_charges + charge_tiers tables
    pricing_config  JSONB NOT NULL DEFAULT '[]',
    -- Example pricing_config:
    -- [
    --   {
    --     "key": "base_fee",
    --     "name": "Platform Fee",
    --     "charge_type": "recurring",
    --     "pricing_model": "flat",
    --     "amount": 99.00,
    --     "invoice_display_name": "Pro Plan - Monthly"
    --   },
    --   {
    --     "key": "api_calls",
    --     "name": "API Calls",
    --     "charge_type": "usage",
    --     "pricing_model": "graduated",
    --     "meter_id": "mtr_abc123",
    --     "tiers": [
    --       {"from": 0, "to": 10000, "unit_amount": 0.00, "flat_amount": 0},
    --       {"from": 10001, "to": 100000, "unit_amount": 0.01, "flat_amount": 0},
    --       {"from": 100001, "to": null, "unit_amount": 0.005, "flat_amount": 0}
    --     ],
    --     "min_amount": 50.00,
    --     "included_units": 10000
    --   },
    --   {
    --     "key": "storage",
    --     "name": "Storage",
    --     "charge_type": "usage",
    --     "pricing_model": "per_unit",
    --     "meter_id": "mtr_def456",
    --     "unit_amount": 0.50,
    --     "unit_label": "GB"
    --   },
    --   {
    --     "key": "seats",
    --     "name": "Additional Seats",
    --     "charge_type": "recurring",
    --     "pricing_model": "per_unit",
    --     "unit_amount": 15.00,
    --     "included_units": 5
    --   },
    --   {
    --     "key": "setup",
    --     "name": "Onboarding",
    --     "charge_type": "one_time",
    --     "pricing_model": "flat",
    --     "amount": 500.00
    --   }
    -- ]
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON plans(tenant_id);
CREATE INDEX idx_plans_product ON plans(product_id);
CREATE INDEX idx_plans_pricing ON plans USING gin(pricing_config jsonb_path_ops);

CREATE TABLE coupons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    code            TEXT,
    discount_config JSONB NOT NULL,
    -- Example discount_config:
    -- {
    --   "type": "percentage",
    --   "value": 20,
    --   "duration": "repeating",
    --   "duration_months": 3,
    --   "applies_to_charges": ["base_fee"],     -- charge keys from pricing_config
    --   "max_discount_amount": 100.00
    -- }
    max_redemptions INT,
    redemption_count INT NOT NULL DEFAULT 0,
    valid_from      TIMESTAMPTZ,
    valid_until     TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coupons_tenant_code ON coupons(tenant_id, code);
```

## Usage Metering

```sql
CREATE TABLE meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    event_name      TEXT NOT NULL,
    aggregation_config JSONB NOT NULL,
    -- Example aggregation_config:
    -- {
    --   "type": "sum",
    --   "field": "properties.tokens",
    --   "dedup_field": "properties.request_id",
    --   "filters": [
    --     {"field": "properties.model", "op": "eq", "value": "gpt-4"}
    --   ],
    --   "group_by": ["properties.region"]
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_name)
);

CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    subscription_id UUID,
    -- CloudEvents envelope
    ce_id           TEXT NOT NULL,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    -- Flexible event properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "tokens": 1500,
    --   "model": "gpt-4",
    --   "region": "us-east-1",
    --   "request_id": "req_abc123",
    --   "latency_ms": 450
    -- }
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ce_time);

CREATE UNIQUE INDEX idx_usage_events_dedup ON usage_events(tenant_id, ce_id);
CREATE INDEX idx_usage_events_customer ON usage_events(customer_id, ce_time);
CREATE INDEX idx_usage_events_type ON usage_events(ce_type, ce_time);
CREATE INDEX idx_usage_events_props ON usage_events USING gin(properties jsonb_path_ops);

-- Pre-aggregated usage for billing
CREATE TABLE usage_aggregates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    subscription_id UUID NOT NULL,
    meter_id        UUID NOT NULL REFERENCES meters(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    value           NUMERIC(20,8) NOT NULL,
    event_count     BIGINT NOT NULL DEFAULT 0,
    breakdown       JSONB,                           -- group_by breakdown if configured
    -- Example breakdown: {"us-east-1": 12000, "eu-west-1": 3000}
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscription_id, meter_id, period_start, period_end)
);
```

## Subscription Management

```sql
CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    external_id     TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('trialing', 'active', 'paused', 'past_due', 'cancelled', 'expired')),
    currency        CHAR(3) NOT NULL,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    trial_end       TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    billing_anchor  TIMESTAMPTZ NOT NULL,
    -- Per-subscription overrides stored as JSONB
    overrides       JSONB NOT NULL DEFAULT '{}',
    -- Example overrides:
    -- {
    --   "charges": {
    --     "api_calls": {
    --       "unit_amount_override": 0.008,
    --       "included_units_override": 25000
    --     },
    --     "seats": {
    --       "quantity": 12
    --     }
    --   },
    --   "net_terms_days": 45,
    --   "auto_collection": false,
    --   "custom_billing_period_days": null
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subs_customer ON subscriptions(customer_id);
CREATE INDEX idx_subs_tenant_status ON subscriptions(tenant_id, status);
CREATE INDEX idx_subs_period_end ON subscriptions(current_period_end);

CREATE TABLE applied_coupons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    coupon_id       UUID NOT NULL REFERENCES coupons(id),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'exhausted', 'removed')),
    remaining_periods INT,
    amount_remaining NUMERIC(20,8),
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Invoicing

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'finalized', 'paid', 'partially_paid', 'past_due', 'void', 'uncollectible')),
    invoice_type    TEXT NOT NULL DEFAULT 'subscription',
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(20,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(20,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    total           NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(20,2) NOT NULL DEFAULT 0,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    issued_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    -- Tax breakdown in JSONB (jurisdiction-flexible)
    tax_breakdown   JSONB,
    -- Example:
    -- [
    --   {"jurisdiction": "US-CA", "type": "sales_tax", "rate": 0.0875, "amount": 8.66, "name": "CA Sales Tax"},
    --   {"jurisdiction": "US-CA-SF", "type": "district_tax", "rate": 0.0125, "amount": 1.24, "name": "SF District Tax"}
    -- ]
    -- E-invoicing config (varies by jurisdiction)
    e_invoice_config JSONB,
    -- Example for Peppol:
    -- {
    --   "standard": "peppol_bis_3",
    --   "document_id": "urn:oasis:names:...",
    --   "endpoint_id": "0088:1234567890",
    --   "xml_url": "https://storage.example.com/invoices/inv_abc.xml"
    -- }
    pdf_url         TEXT,
    memo            TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(tenant_id, status);
CREATE INDEX idx_invoices_due ON invoices(due_at) WHERE status IN ('finalized', 'past_due');

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    charge_key      TEXT,                            -- maps to pricing_config charge key
    description     TEXT NOT NULL,
    line_type       TEXT NOT NULL CHECK (line_type IN ('charge', 'proration', 'credit', 'tax', 'discount', 'minimum_adjustment')),
    quantity        NUMERIC(20,8) NOT NULL DEFAULT 1,
    unit_amount     NUMERIC(20,8) NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    -- Usage detail embedded when relevant
    usage_detail    JSONB,
    -- Example:
    -- {
    --   "meter_id": "mtr_abc123",
    --   "meter_name": "API Calls",
    --   "total_usage": 45000,
    --   "included_usage": 10000,
    --   "billable_usage": 35000,
    --   "tier_breakdown": [
    --     {"tier": 1, "from": 10001, "to": 45000, "quantity": 35000, "unit_amount": 0.01, "amount": 350.00}
    --   ]
    -- }
    sort_order      INT NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

CREATE TABLE credit_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    credit_note_number TEXT NOT NULL,
    reason          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'issued' CHECK (status IN ('issued', 'voided')),
    currency        CHAR(3) NOT NULL,
    total           NUMERIC(20,2) NOT NULL,
    refund_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    items           JSONB NOT NULL DEFAULT '[]',     -- credit note line items as JSONB
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, credit_note_number)
);
```

## Payments & Dunning

```sql
CREATE TABLE payment_methods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    method_type     TEXT NOT NULL CHECK (method_type IN ('card', 'bank_account', 'sepa_debit', 'ach', 'paypal', 'wire')),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'failed', 'revoked')),
    -- Gateway-specific details in JSONB (varies by gateway and method type)
    gateway_config  JSONB NOT NULL,
    -- Example for Stripe card:
    -- {
    --   "gateway": "stripe",
    --   "token": "pm_1234567890",
    --   "card_brand": "visa",
    --   "card_last4": "4242",
    --   "card_exp_month": 12,
    --   "card_exp_year": 2027,
    --   "card_fingerprint": "fp_abc",
    --   "sca_authenticated": true
    -- }
    -- Example for SEPA debit:
    -- {
    --   "gateway": "gocardless",
    --   "token": "md_000abc",
    --   "mandate_id": "MD0001234",
    --   "iban_last4": "3456",
    --   "bank_name": "Deutsche Bank",
    --   "mandate_status": "active",
    --   "mandate_reference": "SEPA-2026-0042"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pm_customer ON payment_methods(customer_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'succeeded', 'failed', 'refunded', 'partially_refunded')),
    -- Gateway response in JSONB
    gateway_response JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "gateway": "stripe",
    --   "payment_id": "pi_abc123",
    --   "charge_id": "ch_def456",
    --   "failure_code": null,
    --   "failure_message": null,
    --   "risk_score": 12,
    --   "receipt_url": "https://pay.stripe.com/receipts/..."
    -- }
    refunded_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_customer ON payments(customer_id);

CREATE TABLE dunning_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    -- Campaign rules in JSONB (flexible step definitions)
    rules           JSONB NOT NULL,
    -- Example:
    -- {
    --   "max_attempts": 5,
    --   "cancel_after_days": 30,
    --   "steps": [
    --     {"step": 1, "delay_days": 1, "action": "retry_payment", "use_ai_timing": true},
    --     {"step": 2, "delay_days": 3, "action": "retry_payment", "fallback_method": true},
    --     {"step": 3, "delay_days": 3, "action": "send_email", "template": "payment_failed_soft"},
    --     {"step": 4, "delay_days": 7, "action": "retry_payment", "use_ai_timing": true},
    --     {"step": 5, "delay_days": 14, "action": "send_email", "template": "final_notice"},
    --     {"step": 6, "delay_days": 2, "action": "cancel_subscription"}
    --   ],
    --   "ai_config": {
    --     "model_version": "dunning-v3.2",
    --     "min_confidence": 0.6,
    --     "features": ["payment_history", "card_type", "time_of_day", "day_of_week"]
    --   }
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dunning_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    campaign_id     UUID NOT NULL REFERENCES dunning_campaigns(id),
    step_number     INT NOT NULL,
    attempt_number  INT NOT NULL,
    action_taken    TEXT NOT NULL,
    result          TEXT NOT NULL CHECK (result IN ('success', 'failed', 'skipped', 'pending')),
    payment_id      UUID REFERENCES payments(id),
    -- AI decision context
    ai_context      JSONB,
    -- Example:
    -- {
    --   "confidence": 0.87,
    --   "model_version": "dunning-v3.2",
    --   "predicted_optimal_time": "2026-05-13T10:30:00Z",
    --   "features_used": {
    --     "card_type": "visa",
    --     "prev_failures": 1,
    --     "days_past_due": 3,
    --     "customer_lifetime_days": 456,
    --     "historical_success_rate": 0.92
    --   }
    -- }
    scheduled_at    TIMESTAMPTZ NOT NULL,
    executed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dunning_invoice ON dunning_attempts(invoice_id);
CREATE INDEX idx_dunning_scheduled ON dunning_attempts(scheduled_at) WHERE result = 'pending';
```

## Credit Ledger

```sql
CREATE TABLE credit_wallets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(20,8) NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active',
    wallet_config   JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "type": "prepaid_commit",
    --   "committed_amount": 10000.00,
    --   "rollover_enabled": false,
    --   "rollover_max_percent": 0,
    --   "auto_topup_enabled": false,
    --   "auto_topup_threshold": null,
    --   "auto_topup_amount": null,
    --   "expires_at": "2027-05-12T00:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallets_customer ON credit_wallets(customer_id);

CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id       UUID NOT NULL REFERENCES credit_wallets(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('credit', 'debit', 'expiration', 'void', 'topup')),
    amount          NUMERIC(20,8) NOT NULL,
    balance_after   NUMERIC(20,8) NOT NULL,
    description     TEXT,
    reference_type  TEXT,                            -- 'invoice', 'manual', 'auto_topup'
    reference_id    UUID,
    idempotency_key TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credit_txns_wallet ON credit_transactions(wallet_id, created_at);
```

## Revenue Recognition

```sql
CREATE TABLE revenue_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    inception_date  DATE NOT NULL,
    total_transaction_price NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    -- ASC 606 analysis in JSONB
    asc606_analysis JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "step1_contract_identified": true,
    --   "step2_obligations": [
    --     {
    --       "id": "po_001",
    --       "description": "SaaS Platform Access",
    --       "type": "over_time",
    --       "standalone_selling_price": 1200.00,
    --       "allocated_price": 1140.00,
    --       "satisfaction_method": "straight_line",
    --       "charge_key": "base_fee"
    --     },
    --     {
    --       "id": "po_002",
    --       "description": "Onboarding Services",
    --       "type": "point_in_time",
    --       "standalone_selling_price": 500.00,
    --       "allocated_price": 475.00,
    --       "charge_key": "setup"
    --     }
    --   ],
    --   "step3_transaction_price": 1615.00,
    --   "step3_variable_consideration": {
    --     "usage_estimate_method": "expected_value",
    --     "constraint_applied": true
    --   },
    --   "step4_allocation_method": "relative_ssp",
    --   "ai_classification_confidence": 0.94,
    --   "ai_model_version": "asc606-v2.1"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

CREATE TABLE revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES revenue_contracts(id),
    obligation_key  TEXT NOT NULL,                    -- maps to asc606_analysis.step2_obligations[].id
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    recognized      BOOLEAN NOT NULL DEFAULT false,
    recognized_at   TIMESTAMPTZ,
    journal_entry_ref TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_sched_contract ON revenue_schedules(contract_id);
CREATE INDEX idx_rev_sched_period ON revenue_schedules(period_start, period_end) WHERE NOT recognized;

CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    entry_number    TEXT NOT NULL,
    entry_date      DATE NOT NULL,
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    source_id       UUID NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'posted', 'reversed')),
    lines           JSONB NOT NULL,
    -- Example:
    -- [
    --   {"account_code": "4000", "account_name": "Revenue", "debit": 99.00, "credit": 0},
    --   {"account_code": "2500", "account_name": "Deferred Revenue", "debit": 0, "credit": 99.00}
    -- ]
    posted_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entry_number)
);
```

## Webhooks & Audit

```sql
CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example: {"retry_policy": "exponential", "max_retries": 5, "timeout_ms": 30000}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id),
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    response_status INT,
    attempt_count   INT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'pending',
    next_retry_at   TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wh_deliveries_retry ON webhook_deliveries(next_retry_at) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_type      TEXT NOT NULL,
    actor_id        TEXT NOT NULL,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    request_context JSONB,
    -- Example: {"ip": "1.2.3.4", "user_agent": "...", "api_key_prefix": "sk_live_abc"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(tenant_id, actor_id, created_at);
```

---

## Key Query Examples

### Find plans with graduated pricing

```sql
SELECT id, name, pricing_config
FROM plans
WHERE tenant_id = $1
  AND pricing_config @> '[{"pricing_model": "graduated"}]';
```

### Calculate usage-based charges with tier breakdown

```sql
-- Application fetches the pricing_config JSONB and calculates tiers in code,
-- but can also query usage aggregates with JSONB breakdown:
SELECT
    ua.meter_id,
    m.name AS meter_name,
    ua.value AS total_usage,
    ua.breakdown
FROM usage_aggregates ua
JOIN meters m ON m.id = ua.meter_id
WHERE ua.subscription_id = $1
  AND ua.period_start = $2
  AND ua.period_end = $3;
```

### Query dunning attempts with AI context

```sql
SELECT
    da.attempt_number,
    da.action_taken,
    da.result,
    da.ai_context->>'confidence' AS ai_confidence,
    da.ai_context->'features_used'->>'historical_success_rate' AS hist_success_rate,
    da.executed_at
FROM dunning_attempts da
WHERE da.invoice_id = $1
ORDER BY da.attempt_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Customer | 2 | JSONB for addresses, tax config, tenant settings |
| Product Catalog | 3 | Plans with JSONB pricing_config (replaces charge/tier tables) |
| Usage Metering | 3 | Events (partitioned), meters with JSONB aggregation config, aggregates |
| Subscriptions | 2 | JSONB overrides for per-customer pricing |
| Invoicing | 3 | JSONB for tax breakdown, e-invoice config, usage detail |
| Payments & Dunning | 4 | JSONB for gateway config, AI context, campaign rules |
| Credit Ledger | 2 | JSONB for wallet configuration |
| Revenue Recognition | 3 | JSONB for ASC 606 analysis, journal entry lines |
| Webhooks & Audit | 3 | JSONB payloads, partitioned audit log |
| **Total** | **25** | ~30% fewer tables than normalized model |

---

## Key Design Decisions

1. **Pricing configuration as JSONB** -- the `plans.pricing_config` column replaces 2 normalized tables (`plan_charges` + `charge_tiers`). New pricing models (percentage-of-revenue, committed-minimum, bundle discounts) are added by extending the JSONB structure without schema migrations.

2. **Gateway-specific data in JSONB** -- `payment_methods.gateway_config` and `payments.gateway_response` accommodate the different data shapes returned by Stripe, Adyen, GoCardless, and Braintree without separate gateway-specific tables.

3. **Tax breakdown as JSONB array** -- tax rules vary dramatically by jurisdiction (US state/county/city, EU VAT, AU GST). Storing the tax breakdown as a JSONB array on invoices avoids a complex tax jurisdiction table hierarchy.

4. **AI decision context as JSONB** -- `dunning_attempts.ai_context` captures the model version, confidence score, and feature values used for each AI-driven retry decision. As the ML model evolves, the feature set changes without schema changes.

5. **ASC 606 analysis as JSONB** -- the five-step analysis for each revenue contract is stored as a structured JSONB document. This is intentionally not normalized because the analysis structure varies by contract complexity and may be recomputed by different AI model versions.

6. **Core financial fields remain relational** -- amounts, currencies, dates, and statuses are typed columns with CHECK constraints. JSONB is used only for variable/extensible data, not for core financial fields that must be queryable and constrained.

7. **GIN indexes on frequently queried JSONB** -- `usage_events.properties`, `plans.pricing_config`, and `customers.metadata` have GIN indexes for containment queries, enabling filtered aggregation without full table scans.

8. **Journal entry lines as JSONB** -- a single invoice typically generates 2-4 journal entry lines. Embedding these as a JSONB array on the journal entry avoids a join-heavy pattern for a simple read operation.

9. **Subscription overrides replace per-customer pricing tables** -- instead of a `subscription_items` table with override columns, per-customer pricing adjustments are stored as `subscriptions.overrides` JSONB, keyed by charge key from the plan's pricing_config.

10. **Wallet configuration supports diverse prepaid models** -- `credit_wallets.wallet_config` JSONB supports prepaid commits, auto-topup, rollover, and expiration policies without separate configuration tables for each wallet behavior.
