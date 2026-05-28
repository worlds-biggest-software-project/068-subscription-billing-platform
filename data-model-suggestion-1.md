# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Subscription Billing Platform · Created: 2026-05-12

## Philosophy

The entity-centric normalized relational model follows classic database normalization principles (3NF/BCNF), giving every domain concept its own table with strict foreign key relationships. Each subscription, invoice, payment, usage event, revenue schedule, and dunning attempt is a first-class entity with well-defined columns and constraints. This mirrors how Stripe, Chargebee, and Zuora structure their internal data -- separate resources for customers, subscriptions, invoices, invoice line items, payments, payment methods, and so on.

This approach prioritizes data integrity above all else. Every relationship is enforced at the database level. Every amount is stored in a dedicated typed column. Every state transition is validated by CHECK constraints. The schema is self-documenting: a developer reading the DDL understands the business domain without needing external documentation.

The trade-off is table proliferation. A full subscription billing system in normalized form easily reaches 50-70 tables. Queries that span the billing lifecycle (e.g., "show me this customer's entire financial history") require multi-table JOINs. But for a system that must comply with ASC 606, GDPR, and PCI DSS, the clarity and auditability of normalized data is a significant advantage.

**Best for:** Teams building a compliance-first billing platform where data integrity, auditability, and clear regulatory alignment are paramount.

**Trade-offs:**
- (+) Strong referential integrity enforced at the database level
- (+) Clean alignment with ASC 606 five-step model (contracts, performance obligations, transaction prices are separate entities)
- (+) Easy to build REST API resources that map 1:1 to tables
- (+) Well-understood by most backend engineers
- (-) High table count (60+ tables) increases schema complexity
- (-) Multi-table JOINs for cross-entity queries can be expensive at scale
- (-) Adding new pricing models or jurisdiction-specific fields requires schema migrations
- (-) Usage event tables at high volume (billions/month) require partitioning strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Dedicated tables for contracts, performance obligations, standalone selling prices, and revenue schedules map directly to the five-step model |
| ISO 4217 | All monetary columns paired with a `currency CHAR(3)` column referencing ISO 4217 alpha-3 codes |
| ISO 3166-1 | Customer and tax jurisdiction addresses use `country_code CHAR(2)` per ISO 3166-1 alpha-2 |
| CloudEvents v1.0 | Usage events table includes `ce_id`, `ce_source`, `ce_type`, `ce_time` columns matching CloudEvents required attributes |
| PCI DSS v4.0.1 | No raw card data stored; `payment_methods` table holds only gateway tokens and card metadata (last4, brand, expiry) |
| SEPA | SEPA mandate fields stored in `payment_methods` for direct debit collection |
| Peppol BIS 3.0 / UBL 2.1 | Invoice table includes `peppol_document_id` and `ubl_xml_url` for e-invoicing compliance |
| RFC 7807 | Error responses follow Problem Details format (not a schema concern but informs API error table design) |
| OAuth 2.0 / JWT | API credentials table stores OAuth client IDs and hashed secrets |

---

## Core Identity & Tenant Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'cancelled')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    external_id     TEXT,                          -- caller's own customer ID
    name            TEXT NOT NULL,
    email           TEXT,
    billing_address JSONB,                         -- structured: line1, line2, city, state, postal_code, country_code (ISO 3166-1)
    shipping_address JSONB,
    tax_id          TEXT,                           -- VAT number, EIN, etc.
    tax_id_type     TEXT,                           -- e.g., 'eu_vat', 'us_ein', 'au_abn'
    currency        CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    locale          TEXT DEFAULT 'en',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_email ON customers(tenant_id, email);
```

## Product Catalog & Pricing

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
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'grandfathered')),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON plans(tenant_id);
CREATE INDEX idx_plans_product ON plans(product_id);

CREATE TABLE plan_charges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    name            TEXT NOT NULL,
    charge_type     TEXT NOT NULL CHECK (charge_type IN ('recurring', 'usage', 'one_time', 'setup')),
    pricing_model   TEXT NOT NULL CHECK (pricing_model IN ('flat', 'per_unit', 'tiered', 'volume', 'graduated', 'percentage', 'package')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    amount          NUMERIC(20,8),                 -- for flat/per_unit; NULL for tiered/volume
    meter_id        UUID REFERENCES meters(id),    -- for usage charges
    min_amount      NUMERIC(20,8),                 -- minimum charge per period
    max_amount      NUMERIC(20,8),                 -- maximum charge per period
    included_units  NUMERIC(20,8) DEFAULT 0,       -- free units before charging
    proration_enabled BOOLEAN NOT NULL DEFAULT true,
    invoice_display_name TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_charges_plan ON plan_charges(plan_id);

CREATE TABLE charge_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_charge_id  UUID NOT NULL REFERENCES plan_charges(id),
    tier_number     INT NOT NULL,
    up_to           NUMERIC(20,8),                 -- NULL means unlimited (final tier)
    unit_amount     NUMERIC(20,8) NOT NULL,
    flat_amount     NUMERIC(20,8) DEFAULT 0,       -- flat fee for entering this tier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (plan_charge_id, tier_number)
);

CREATE TABLE coupons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    code            TEXT,
    discount_type   TEXT NOT NULL CHECK (discount_type IN ('percentage', 'fixed_amount')),
    discount_value  NUMERIC(20,8) NOT NULL,
    currency        CHAR(3),                       -- for fixed_amount discounts
    duration        TEXT NOT NULL CHECK (duration IN ('once', 'repeating', 'forever')),
    duration_months INT,                           -- for repeating
    max_redemptions INT,
    redemption_count INT NOT NULL DEFAULT 0,
    applies_to_plan_ids UUID[],
    valid_from      TIMESTAMPTZ,
    valid_until     TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coupons_tenant ON coupons(tenant_id);
CREATE INDEX idx_coupons_code ON coupons(tenant_id, code);
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
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    trial_start     TIMESTAMPTZ,
    trial_end       TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    paused_at       TIMESTAMPTZ,
    resume_at       TIMESTAMPTZ,
    billing_anchor  TIMESTAMPTZ NOT NULL,          -- date that anchors billing cycle
    net_terms_days  INT NOT NULL DEFAULT 0,        -- payment due N days after invoice
    auto_collection BOOLEAN NOT NULL DEFAULT true,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_customer ON subscriptions(customer_id);
CREATE INDEX idx_subscriptions_tenant_status ON subscriptions(tenant_id, status);
CREATE INDEX idx_subscriptions_period_end ON subscriptions(current_period_end);

CREATE TABLE subscription_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    plan_charge_id  UUID NOT NULL REFERENCES plan_charges(id),
    quantity        NUMERIC(20,8) NOT NULL DEFAULT 1,
    unit_amount_override NUMERIC(20,8),            -- customer-specific price override
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscription_items_sub ON subscription_items(subscription_id);

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

## Usage Metering

```sql
CREATE TABLE meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    event_name      TEXT NOT NULL,                  -- CloudEvents ce_type to match
    aggregation_type TEXT NOT NULL CHECK (aggregation_type IN ('count', 'sum', 'max', 'unique_count', 'latest')),
    aggregation_field TEXT,                         -- JSON path into event properties for sum/max
    dedup_field     TEXT,                           -- field used for idempotent deduplication
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_name)
);

CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    meter_id        UUID NOT NULL REFERENCES meters(id),
    -- CloudEvents v1.0 fields
    ce_id           TEXT NOT NULL,                  -- CloudEvents id (idempotency key)
    ce_source       TEXT NOT NULL,                  -- CloudEvents source
    ce_type         TEXT NOT NULL,                  -- CloudEvents type (matches meter.event_name)
    ce_time         TIMESTAMPTZ NOT NULL,           -- CloudEvents time (when the event occurred)
    -- Billing fields
    properties      JSONB NOT NULL DEFAULT '{}',    -- arbitrary event properties
    quantity        NUMERIC(20,8),                  -- pre-extracted billable quantity (if applicable)
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed       BOOLEAN NOT NULL DEFAULT false
) PARTITION BY RANGE (ce_time);

-- Partition by month for time-range queries
-- CREATE TABLE usage_events_2026_05 PARTITION OF usage_events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE UNIQUE INDEX idx_usage_events_dedup ON usage_events(tenant_id, ce_id);
CREATE INDEX idx_usage_events_customer_time ON usage_events(customer_id, ce_time);
CREATE INDEX idx_usage_events_meter_time ON usage_events(meter_id, ce_time);
CREATE INDEX idx_usage_events_sub ON usage_events(subscription_id, ce_time);

CREATE TABLE usage_aggregates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    meter_id        UUID NOT NULL REFERENCES meters(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    aggregated_value NUMERIC(20,8) NOT NULL,
    event_count     BIGINT NOT NULL DEFAULT 0,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscription_id, meter_id, period_start, period_end)
);

CREATE INDEX idx_usage_agg_sub_period ON usage_aggregates(subscription_id, period_start, period_end);
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
    invoice_type    TEXT NOT NULL DEFAULT 'subscription' CHECK (invoice_type IN ('subscription', 'one_time', 'credit_note')),
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(20,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(20,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    total           NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(20,2) NOT NULL DEFAULT 0,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    issued_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    voided_at       TIMESTAMPTZ,
    -- E-invoicing (Peppol / UBL)
    peppol_document_id TEXT,
    ubl_xml_url     TEXT,
    -- PDF
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
    subscription_item_id UUID REFERENCES subscription_items(id),
    description     TEXT NOT NULL,
    line_type       TEXT NOT NULL CHECK (line_type IN ('charge', 'proration', 'credit', 'tax', 'discount', 'minimum_adjustment')),
    quantity        NUMERIC(20,8) NOT NULL DEFAULT 1,
    unit_amount     NUMERIC(20,8) NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    meter_id        UUID REFERENCES meters(id),
    usage_quantity  NUMERIC(20,8),                  -- raw usage if usage-based
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
    reason          TEXT NOT NULL CHECK (reason IN ('duplicate', 'fraudulent', 'order_change', 'product_unsatisfactory', 'other')),
    status          TEXT NOT NULL DEFAULT 'issued' CHECK (status IN ('issued', 'voided')),
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(20,2) NOT NULL,
    tax_amount      NUMERIC(20,2) NOT NULL DEFAULT 0,
    total           NUMERIC(20,2) NOT NULL,
    refund_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,  -- amount added to customer balance
    memo            TEXT,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, credit_note_number)
);
```

## Payments & Payment Methods

```sql
CREATE TABLE payment_methods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    gateway         TEXT NOT NULL,                  -- 'stripe', 'adyen', 'braintree', 'gocardless'
    gateway_token   TEXT NOT NULL,                  -- tokenized reference (PCI DSS compliant)
    method_type     TEXT NOT NULL CHECK (method_type IN ('card', 'bank_account', 'sepa_debit', 'ach', 'paypal', 'wire')),
    -- Card metadata (no raw PAN per PCI DSS)
    card_brand      TEXT,                           -- 'visa', 'mastercard', 'amex'
    card_last4      CHAR(4),
    card_exp_month  SMALLINT,
    card_exp_year   SMALLINT,
    -- SEPA mandate fields
    sepa_mandate_id TEXT,
    sepa_iban_last4 CHAR(4),
    -- State
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'failed', 'revoked')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_methods_customer ON payment_methods(customer_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    gateway         TEXT NOT NULL,
    gateway_payment_id TEXT,                        -- external payment reference
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'succeeded', 'failed', 'refunded', 'partially_refunded')),
    failure_code    TEXT,
    failure_message TEXT,
    refunded_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_customer ON payments(customer_id);
CREATE INDEX idx_payments_gateway ON payments(gateway, gateway_payment_id);

CREATE TABLE refunds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    credit_note_id  UUID REFERENCES credit_notes(id),
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'succeeded', 'failed')),
    reason          TEXT,
    gateway_refund_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Credit Ledger & Wallets

```sql
CREATE TABLE credit_wallets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    name            TEXT NOT NULL DEFAULT 'Default',
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(20,8) NOT NULL DEFAULT 0,
    consumed        NUMERIC(20,8) NOT NULL DEFAULT 0,
    expired         NUMERIC(20,8) NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'depleted', 'expired', 'terminated')),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallets_customer ON credit_wallets(customer_id);

CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id       UUID NOT NULL REFERENCES credit_wallets(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('credit', 'debit', 'expiration', 'void')),
    amount          NUMERIC(20,8) NOT NULL,         -- positive for credit, negative for debit
    balance_after   NUMERIC(20,8) NOT NULL,
    description     TEXT,
    invoice_id      UUID REFERENCES invoices(id),   -- linked invoice for debit transactions
    idempotency_key TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credit_txns_wallet ON credit_transactions(wallet_id, created_at);
```

## Dunning & Payment Recovery

```sql
CREATE TABLE dunning_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    max_attempts    INT NOT NULL DEFAULT 4,
    cancel_after_days INT NOT NULL DEFAULT 30,      -- cancel subscription if unpaid
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dunning_campaign_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES dunning_campaigns(id),
    step_number     INT NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('retry_payment', 'send_email', 'send_sms', 'pause_subscription', 'cancel_subscription')),
    delay_days      INT NOT NULL,                   -- days after previous step
    email_template_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (campaign_id, step_number)
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
    failure_reason  TEXT,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    executed_at     TIMESTAMPTZ,
    -- AI dunning fields
    ai_confidence   NUMERIC(5,4),                   -- AI model confidence for this retry timing
    ai_model_version TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dunning_invoice ON dunning_attempts(invoice_id);
CREATE INDEX idx_dunning_scheduled ON dunning_attempts(scheduled_at) WHERE result = 'pending';
```

## ASC 606 Revenue Recognition

```sql
CREATE TABLE revenue_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'modified', 'completed', 'cancelled')),
    inception_date  DATE NOT NULL,
    expiration_date DATE,
    total_transaction_price NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_contracts_customer ON revenue_contracts(customer_id);

CREATE TABLE performance_obligations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES revenue_contracts(id),
    description     TEXT NOT NULL,
    obligation_type TEXT NOT NULL CHECK (obligation_type IN ('point_in_time', 'over_time')),
    standalone_selling_price NUMERIC(20,2) NOT NULL,
    allocated_price NUMERIC(20,2) NOT NULL,         -- after relative SSP allocation
    currency        CHAR(3) NOT NULL,
    satisfaction_method TEXT CHECK (satisfaction_method IN ('output', 'input', 'straight_line')),
    satisfied_percent NUMERIC(5,2) NOT NULL DEFAULT 0,
    fully_satisfied_at TIMESTAMPTZ,
    plan_charge_id  UUID REFERENCES plan_charges(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_perf_obligations_contract ON performance_obligations(contract_id);

CREATE TABLE revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    performance_obligation_id UUID NOT NULL REFERENCES performance_obligations(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    recognized      BOOLEAN NOT NULL DEFAULT false,
    recognized_at   TIMESTAMPTZ,
    journal_entry_id UUID REFERENCES journal_entries(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_sched_po ON revenue_schedules(performance_obligation_id);
CREATE INDEX idx_rev_sched_period ON revenue_schedules(period_start, period_end) WHERE NOT recognized;

CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    entry_number    TEXT NOT NULL,
    entry_date      DATE NOT NULL,
    description     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'posted', 'reversed')),
    source_type     TEXT NOT NULL,                   -- 'revenue_recognition', 'payment', 'refund', 'credit_note'
    source_id       UUID NOT NULL,
    posted_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entry_number)
);

CREATE TABLE journal_entry_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entries(id),
    account_code    TEXT NOT NULL,                   -- GL account code
    account_name    TEXT NOT NULL,
    debit_amount    NUMERIC(20,2) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jel_entry ON journal_entry_lines(journal_entry_id);
CREATE INDEX idx_jel_account ON journal_entry_lines(account_code);
```

## Webhooks & API Integration

```sql
CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,                 -- e.g., ['invoice.finalized', 'payment.succeeded']
    secret          TEXT NOT NULL,                   -- HMAC signing secret
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'disabled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id),
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    response_status INT,
    response_body   TEXT,
    attempt_count   INT NOT NULL DEFAULT 1,
    max_attempts    INT NOT NULL DEFAULT 5,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'delivered', 'failed')),
    next_retry_at   TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_retry ON webhook_deliveries(next_retry_at) WHERE status = 'pending';
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'api_key', 'system', 'webhook')),
    actor_id        TEXT NOT NULL,
    action          TEXT NOT NULL,                   -- e.g., 'subscription.created', 'invoice.voided'
    resource_type   TEXT NOT NULL,                   -- e.g., 'subscription', 'invoice'
    resource_id     UUID NOT NULL,
    changes         JSONB,                           -- before/after snapshot
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_actor ON audit_log(tenant_id, actor_id, created_at);
```

## Tax Configuration

```sql
CREATE TABLE tax_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,                   -- e.g., 'US-CA Sales Tax', 'DE VAT Standard'
    jurisdiction    TEXT NOT NULL,                   -- ISO 3166 code or subdivision
    rate            NUMERIC(7,4) NOT NULL,           -- e.g., 0.0875 for 8.75%
    tax_type        TEXT NOT NULL CHECK (tax_type IN ('vat', 'gst', 'sales_tax', 'none')),
    inclusive       BOOLEAN NOT NULL DEFAULT false,   -- tax included in price?
    valid_from      DATE NOT NULL,
    valid_until     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tax_rates_jurisdiction ON tax_rates(tenant_id, jurisdiction);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Customer | 2 | Core identity |
| Product Catalog & Pricing | 5 | Products, plans, charges, tiers, coupons |
| Subscription Management | 3 | Subscriptions, items, applied coupons |
| Usage Metering | 3 | Meters, events (partitioned), aggregates |
| Invoicing | 3 | Invoices, line items, credit notes |
| Payments | 4 | Methods, payments, refunds, tax rates |
| Credit Ledger | 2 | Wallets, transactions |
| Dunning | 3 | Campaigns, steps, attempts |
| Revenue Recognition | 4 | Contracts, obligations, schedules, journal entries + lines |
| Webhooks & API | 2 | Endpoints, deliveries |
| Audit | 1 | Partitioned audit log |
| **Total** | **32** | Plus junction/supporting tables |

---

## Key Design Decisions

1. **UUID primary keys everywhere** -- enables distributed ID generation without coordination, essential for multi-region deployment and API-first architecture.

2. **All monetary amounts use `NUMERIC(20,2)` for invoicing and `NUMERIC(20,8)` for unit pricing** -- avoids floating-point rounding errors in financial calculations. The higher precision for unit amounts accommodates per-API-call pricing (e.g., $0.00001 per token).

3. **Usage events table is partitioned by `ce_time`** -- at high volumes (millions of events/day), time-range partitioning keeps queries fast and enables efficient data lifecycle management (archive old partitions).

4. **CloudEvents fields stored as first-class columns** -- `ce_id`, `ce_source`, `ce_type`, `ce_time` enable direct CloudEvents ingestion without translation, with `ce_id` providing natural idempotency.

5. **ASC 606 modeled as separate entity chain** -- `revenue_contracts` -> `performance_obligations` -> `revenue_schedules` -> `journal_entries` directly maps the five-step revenue recognition model to database entities, making audit compliance straightforward.

6. **Credit wallet uses append-only transaction log** -- following double-entry ledger principles, balance is verified against the sum of all transactions. The `balance_after` column provides O(1) balance lookups while the transaction log provides auditability.

7. **Dunning includes AI confidence fields** -- `ai_confidence` and `ai_model_version` columns on `dunning_attempts` enable tracking AI-optimized retry decisions alongside rule-based ones, supporting gradual rollout of AI dunning.

8. **No raw card data stored** -- `payment_methods` stores only gateway tokens and card metadata (last4, brand, expiry) per PCI DSS requirements. All actual card processing delegated to payment gateways.

9. **Tenant-scoped with row-level isolation** -- every table includes `tenant_id` for multi-tenant isolation. Row-level security policies can be layered on top for PostgreSQL-native tenant isolation.

10. **Audit log is append-only and partitioned** -- captures before/after state for every mutation, partitioned by time for efficient retention management and GDPR compliance.
