# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Subscription Billing Platform · Created: 2026-05-12

## Philosophy

The event-sourced model treats every state change in the billing system as an immutable event appended to a log. Instead of storing the "current state" of a subscription in a mutable row, the system stores the full sequence of events that produced that state: `SubscriptionCreated`, `PlanChanged`, `SubscriptionPaused`, `InvoiceGenerated`, `PaymentAttempted`, `PaymentSucceeded`. The current state is derived by replaying the event stream, and read-optimized "projections" (materialized views) serve API queries.

This architecture is built on the CQRS (Command Query Responsibility Segregation) pattern. The write side accepts commands, validates business rules against the current aggregate state, and emits domain events to an append-only event store. The read side maintains denormalized projection tables optimized for specific query patterns (customer dashboard, invoice listing, revenue reporting). This separation means the write path and read path can scale independently.

Event sourcing is used in production by financial infrastructure companies (Stripe's ledger system, Modern Treasury, banks using Icon Solutions' IPF platform) precisely because financial systems need complete audit trails, temporal querying ("what was the balance on March 15?"), and the ability to retroactively correct and replay. For a subscription billing platform that must support ASC 606 revenue recognition, AI-powered dunning optimization, and regulatory audit trails, event sourcing provides the strongest foundation for compliance and data-driven intelligence.

**Best for:** Teams building an audit-critical, AI-powered billing platform where full event history enables temporal queries, revenue recognition replays, and machine learning on billing behavior patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail by construction -- every state change is permanently recorded
- (+) Temporal queries are trivial: replay events to any point in time for ASC 606 as-of reporting
- (+) Natural fit for AI/ML: event streams are training data for dunning optimization and churn prediction
- (+) Supports event replay for bug fixes, retroactive corrections, and pricing simulations
- (+) Write path is fast (append-only inserts)
- (-) Increased storage requirements (every state change stored, not just current state)
- (-) Eventually consistent read models require careful design of projection rebuilds
- (-) Higher engineering complexity: developers must understand event sourcing patterns
- (-) Projection rebuild can be slow for aggregates with long event histories
- (-) Debugging requires understanding event sequences rather than inspecting single rows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue recognition computed by replaying contract events through the five-step model; any restatement replays events with updated rules |
| CloudEvents v1.0 | Usage metering events AND internal domain events both follow CloudEvents envelope format for consistency |
| ISO 4217 | Currency codes embedded in event payloads and projection tables |
| ISO 3166-1 | Jurisdiction codes in customer and tax events |
| PCI DSS v4.0.1 | Payment events contain only tokenized references; raw card data never enters the event store |
| SEPA | SEPA mandate events track mandate lifecycle (created, amended, cancelled) |
| Peppol BIS 3.0 | Invoice finalization events trigger Peppol document generation; e-invoice references stored in projections |
| GDPR | Event store supports "crypto-shredding" -- encrypting PII with per-customer keys that can be destroyed for right-to-erasure |

---

## Event Store (Write Side)

The event store is the single source of truth. All other tables are derived projections.

```sql
-- Core event store: append-only, immutable
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    aggregate_type  TEXT NOT NULL,                   -- 'subscription', 'invoice', 'payment', 'customer', 'contract'
    aggregate_id    UUID NOT NULL,                   -- the entity this event belongs to
    event_type      TEXT NOT NULL,                   -- e.g., 'SubscriptionCreated', 'InvoiceFinalized'
    event_version   INT NOT NULL,                    -- schema version for this event type
    sequence_number BIGINT NOT NULL,                 -- ordering within the aggregate
    -- CloudEvents envelope
    ce_id           TEXT NOT NULL UNIQUE,             -- CloudEvents id (idempotency)
    ce_source       TEXT NOT NULL DEFAULT 'billing-platform',
    ce_type         TEXT NOT NULL,                   -- CloudEvents type (mirrors event_type with namespace)
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Event data
    payload         JSONB NOT NULL,                  -- the event body
    metadata        JSONB NOT NULL DEFAULT '{}',     -- actor, IP, correlation_id, causation_id
    -- Crypto-shredding support (GDPR)
    encryption_key_id TEXT,                          -- reference to per-customer encryption key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, sequence_number)
) PARTITION BY RANGE (created_at);

-- Partitioned monthly for lifecycle management
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_tenant ON events(tenant_id, created_at);
CREATE INDEX idx_events_ce_type ON events(ce_type, ce_time);

-- Snapshot store: periodic snapshots to avoid replaying full history
CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    sequence_number BIGINT NOT NULL,                 -- snapshot taken at this sequence
    state           JSONB NOT NULL,                  -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, sequence_number)
);

CREATE INDEX idx_snapshots_aggregate ON snapshots(aggregate_type, aggregate_id, sequence_number DESC);
```

### Domain Event Catalog

Below are the key event types with example payloads:

```sql
-- Example: SubscriptionCreated event
-- payload: {
--   "customer_id": "cust_abc123",
--   "plan_id": "plan_xyz789",
--   "items": [
--     {"plan_charge_id": "chg_001", "quantity": 1}
--   ],
--   "billing_period": "monthly",
--   "currency": "USD",
--   "trial_end": "2026-06-12T00:00:00Z",
--   "billing_anchor": "2026-05-12T00:00:00Z"
-- }

-- Example: UsageEventIngested event
-- payload: {
--   "customer_id": "cust_abc123",
--   "meter_name": "api_calls",
--   "quantity": 1,
--   "properties": {"endpoint": "/v1/generate", "model": "gpt-4"},
--   "original_ce_id": "evt_ext_12345",
--   "original_ce_time": "2026-05-12T14:30:00Z"
-- }

-- Example: InvoiceFinalized event
-- payload: {
--   "invoice_number": "INV-2026-0042",
--   "customer_id": "cust_abc123",
--   "subscription_id": "sub_def456",
--   "currency": "USD",
--   "subtotal": 299.00,
--   "tax_amount": 23.92,
--   "total": 322.92,
--   "line_items": [
--     {"description": "Pro Plan - May 2026", "amount": 99.00, "type": "recurring"},
--     {"description": "API Calls (12,450 @ $0.01)", "amount": 124.50, "type": "usage"},
--     {"description": "Storage (75 GB @ $1.00)", "amount": 75.00, "type": "usage"}
--   ],
--   "due_at": "2026-06-11T00:00:00Z"
-- }

-- Example: PaymentAttempted event
-- payload: {
--   "invoice_id": "inv_ghi789",
--   "amount": 322.92,
--   "currency": "USD",
--   "payment_method_type": "card",
--   "gateway": "stripe",
--   "gateway_token": "pm_tok_abc",
--   "card_last4": "4242",
--   "dunning_attempt": 1,
--   "ai_retry_confidence": 0.87,
--   "ai_model_version": "dunning-v3.2"
-- }

-- Example: RevenueRecognized event
-- payload: {
--   "contract_id": "rc_jkl012",
--   "performance_obligation_id": "po_mno345",
--   "period": "2026-05",
--   "amount": 99.00,
--   "currency": "USD",
--   "recognition_method": "straight_line",
--   "journal_entry": {
--     "debit": {"account": "4000", "name": "Revenue", "amount": 99.00},
--     "credit": {"account": "2500", "name": "Deferred Revenue", "amount": 99.00}
--   }
-- }
```

### Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    current_version INT NOT NULL DEFAULT 1,
    description     TEXT NOT NULL,
    json_schema     JSONB NOT NULL,                  -- JSON Schema for payload validation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed data examples:
-- INSERT INTO event_type_registry VALUES
--   ('CustomerCreated',       'customer',     1, 'A new customer was registered', '{"type":"object",...}'),
--   ('SubscriptionCreated',   'subscription', 1, 'A new subscription was started', '{"type":"object",...}'),
--   ('SubscriptionAmended',   'subscription', 1, 'Subscription plan or quantity changed mid-cycle', '{"type":"object",...}'),
--   ('SubscriptionPaused',    'subscription', 1, 'Subscription was paused', '{"type":"object",...}'),
--   ('SubscriptionResumed',   'subscription', 1, 'Subscription was resumed from pause', '{"type":"object",...}'),
--   ('SubscriptionCancelled', 'subscription', 1, 'Subscription was cancelled', '{"type":"object",...}'),
--   ('SubscriptionRenewed',   'subscription', 1, 'Subscription renewed for a new period', '{"type":"object",...}'),
--   ('UsageEventIngested',    'metering',     1, 'A usage event was ingested', '{"type":"object",...}'),
--   ('UsagePeriodClosed',     'metering',     1, 'Usage aggregation completed for billing period', '{"type":"object",...}'),
--   ('InvoiceDrafted',        'invoice',      1, 'Invoice draft was created', '{"type":"object",...}'),
--   ('InvoiceFinalized',      'invoice',      1, 'Invoice was finalized and sent', '{"type":"object",...}'),
--   ('InvoiceVoided',         'invoice',      1, 'Invoice was voided', '{"type":"object",...}'),
--   ('PaymentAttempted',      'payment',      1, 'Payment attempt was initiated', '{"type":"object",...}'),
--   ('PaymentSucceeded',      'payment',      1, 'Payment was successfully processed', '{"type":"object",...}'),
--   ('PaymentFailed',         'payment',      1, 'Payment attempt failed', '{"type":"object",...}'),
--   ('RefundIssued',          'payment',      1, 'Refund was issued', '{"type":"object",...}'),
--   ('CreditApplied',         'wallet',       1, 'Credit was added to customer wallet', '{"type":"object",...}'),
--   ('CreditConsumed',        'wallet',       1, 'Credit was consumed against invoice', '{"type":"object",...}'),
--   ('RevenueRecognized',     'contract',     1, 'Revenue was recognized for a period', '{"type":"object",...}'),
--   ('ContractModified',      'contract',     1, 'Revenue contract was modified', '{"type":"object",...}'),
--   ('DunningStepExecuted',   'dunning',      1, 'A dunning workflow step was executed', '{"type":"object",...}');
```

## Projection Tables (Read Side)

Projections are rebuilt from events. They can be dropped and reconstructed at any time.

### Customer Projection

```sql
CREATE TABLE proj_customers (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    external_id     TEXT,
    name            TEXT NOT NULL,
    email           TEXT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    billing_address JSONB,
    tax_id          TEXT,
    tax_id_type     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Derived metrics
    total_mrr       NUMERIC(20,2) NOT NULL DEFAULT 0,
    active_subscriptions INT NOT NULL DEFAULT 0,
    lifetime_value  NUMERIC(20,2) NOT NULL DEFAULT 0,
    last_payment_at TIMESTAMPTZ,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_customers_tenant ON proj_customers(tenant_id);
```

### Subscription Projection

```sql
CREATE TABLE proj_subscriptions (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    plan_id         UUID NOT NULL,
    plan_name       TEXT NOT NULL,                   -- denormalized for read performance
    status          TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    mrr             NUMERIC(20,2) NOT NULL DEFAULT 0,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    trial_end       TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    items           JSONB NOT NULL DEFAULT '[]',     -- denormalized subscription items
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_subs_customer ON proj_subscriptions(customer_id);
CREATE INDEX idx_proj_subs_tenant_status ON proj_subscriptions(tenant_id, status);
CREATE INDEX idx_proj_subs_period ON proj_subscriptions(current_period_end);
```

### Invoice Projection

```sql
CREATE TABLE proj_invoices (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    subscription_id UUID,
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(20,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(20,2) NOT NULL DEFAULT 0,
    total           NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(20,2) NOT NULL DEFAULT 0,
    amount_due      NUMERIC(20,2) NOT NULL DEFAULT 0,
    line_items      JSONB NOT NULL DEFAULT '[]',     -- denormalized for display
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    issued_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    pdf_url         TEXT,
    peppol_document_id TEXT,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_inv_customer ON proj_invoices(customer_id);
CREATE INDEX idx_proj_inv_tenant_status ON proj_invoices(tenant_id, status);
CREATE INDEX idx_proj_inv_due ON proj_invoices(due_at) WHERE status IN ('finalized', 'past_due');
```

### Payment Projection

```sql
CREATE TABLE proj_payments (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    customer_id     UUID NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL,
    gateway         TEXT NOT NULL,
    gateway_payment_id TEXT,
    payment_method_type TEXT,
    card_last4      CHAR(4),
    failure_code    TEXT,
    failure_message TEXT,
    dunning_attempt INT,
    ai_retry_confidence NUMERIC(5,4),
    paid_at         TIMESTAMPTZ,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_pay_invoice ON proj_payments(invoice_id);
CREATE INDEX idx_proj_pay_customer ON proj_payments(customer_id, paid_at);
```

### Revenue Recognition Projection

```sql
CREATE TABLE proj_revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    contract_id     UUID NOT NULL,
    customer_id     UUID NOT NULL,
    performance_obligation_id UUID NOT NULL,
    obligation_description TEXT NOT NULL,
    period_year     INT NOT NULL,
    period_month    INT NOT NULL,
    deferred_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    recognized_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    recognition_method TEXT NOT NULL,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_rev_period ON proj_revenue_schedules(tenant_id, period_year, period_month);
CREATE INDEX idx_proj_rev_contract ON proj_revenue_schedules(contract_id);

-- Waterfall view: monthly revenue bridge
CREATE TABLE proj_revenue_waterfall (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_year     INT NOT NULL,
    period_month    INT NOT NULL,
    new_business    NUMERIC(20,2) NOT NULL DEFAULT 0,
    expansion       NUMERIC(20,2) NOT NULL DEFAULT 0,
    contraction     NUMERIC(20,2) NOT NULL DEFAULT 0,
    churn           NUMERIC(20,2) NOT NULL DEFAULT 0,
    reactivation    NUMERIC(20,2) NOT NULL DEFAULT 0,
    net_mrr         NUMERIC(20,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, period_year, period_month, currency)
);
```

### Credit & Wallet Projection

```sql
CREATE TABLE proj_credit_wallets (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(20,8) NOT NULL DEFAULT 0,
    total_credited  NUMERIC(20,8) NOT NULL DEFAULT 0,
    total_consumed  NUMERIC(20,8) NOT NULL DEFAULT 0,
    total_expired   NUMERIC(20,8) NOT NULL DEFAULT 0,
    expires_at      TIMESTAMPTZ,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_wallets_customer ON proj_credit_wallets(customer_id);
```

### Usage Metering Projection

```sql
CREATE TABLE proj_usage_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    subscription_id UUID NOT NULL,
    meter_name      TEXT NOT NULL,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    aggregated_value NUMERIC(20,8) NOT NULL DEFAULT 0,
    event_count     BIGINT NOT NULL DEFAULT 0,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscription_id, meter_name, period_start, period_end)
);

CREATE INDEX idx_proj_usage_sub ON proj_usage_summaries(subscription_id, period_start);
```

### Dunning & AI Analytics Projection

```sql
CREATE TABLE proj_dunning_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL UNIQUE,
    subscription_id UUID NOT NULL,
    customer_id     UUID NOT NULL,
    attempts        INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_attempt_at TIMESTAMPTZ,
    current_step    INT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'recovered', 'exhausted', 'cancelled')),
    recovered_at    TIMESTAMPTZ,
    -- AI features
    payment_history JSONB NOT NULL DEFAULT '[]',     -- recent payment outcomes for ML features
    predicted_recovery_probability NUMERIC(5,4),
    optimal_retry_hour INT,                          -- AI-predicted best hour to retry
    optimal_retry_day_of_week INT,                   -- AI-predicted best day
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_dunning_next ON proj_dunning_state(next_attempt_at) WHERE status = 'active';
CREATE INDEX idx_proj_dunning_customer ON proj_dunning_state(customer_id);

-- AI training data: historical dunning outcomes (materialized from events)
CREATE TABLE proj_dunning_training_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    attempt_number  INT NOT NULL,
    -- Features
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    payment_method_type TEXT NOT NULL,
    card_brand      TEXT,
    day_of_week     INT NOT NULL,                    -- 0=Sunday
    hour_of_day     INT NOT NULL,
    days_past_due   INT NOT NULL,
    customer_lifetime_days INT NOT NULL,
    previous_failure_count INT NOT NULL DEFAULT 0,
    -- Label
    outcome         TEXT NOT NULL CHECK (outcome IN ('success', 'failure')),
    attempted_at    TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dunning_training_tenant ON proj_dunning_training_data(tenant_id, attempted_at);
```

## Catalog Tables (Reference Data)

These are NOT projections -- they are managed through the command side but serve as reference data.

```sql
CREATE TABLE catalog_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE catalog_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL REFERENCES catalog_products(id),
    name            TEXT NOT NULL,
    billing_period  TEXT NOT NULL,
    billing_interval_months INT NOT NULL DEFAULT 1,
    trial_days      INT DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active',
    charges         JSONB NOT NULL DEFAULT '[]',     -- charge definitions including tiers
    -- Example charges JSONB:
    -- [
    --   {
    --     "name": "Base Fee",
    --     "charge_type": "recurring",
    --     "pricing_model": "flat",
    --     "amount": 99.00,
    --     "currency": "USD"
    --   },
    --   {
    --     "name": "API Calls",
    --     "charge_type": "usage",
    --     "pricing_model": "tiered",
    --     "meter_name": "api_calls",
    --     "tiers": [
    --       {"up_to": 10000, "unit_amount": 0.00},
    --       {"up_to": 100000, "unit_amount": 0.01},
    --       {"up_to": null, "unit_amount": 0.005}
    --     ]
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE catalog_meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    event_name      TEXT NOT NULL,
    aggregation_type TEXT NOT NULL,
    aggregation_field TEXT,
    dedup_field     TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_name)
);

CREATE TABLE catalog_dunning_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    steps           JSONB NOT NULL DEFAULT '[]',
    -- Example steps JSONB:
    -- [
    --   {"step": 1, "delay_days": 1, "action": "retry_payment"},
    --   {"step": 2, "delay_days": 3, "action": "retry_payment"},
    --   {"step": 3, "delay_days": 3, "action": "send_email", "template": "payment_failed"},
    --   {"step": 4, "delay_days": 7, "action": "retry_payment"},
    --   {"step": 5, "delay_days": 14, "action": "cancel_subscription"}
    -- ]
    max_attempts    INT NOT NULL DEFAULT 4,
    cancel_after_days INT NOT NULL DEFAULT 30,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Outbox Pattern (Event Publishing)

```sql
-- Transactional outbox for reliable event publishing to external consumers
CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,                   -- references events.id
    topic           TEXT NOT NULL,                    -- destination: 'webhooks', 'analytics', 'revenue'
    payload         JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'published', 'failed')),
    retry_count     INT NOT NULL DEFAULT 0,
    max_retries     INT NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_pending ON event_outbox(next_retry_at) WHERE status = 'pending';

-- Webhook endpoint configuration
CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Key Query Patterns

### Replay subscription state at a point in time

```sql
-- "What was the state of subscription sub_abc on March 15?"
SELECT payload
FROM events
WHERE aggregate_type = 'subscription'
  AND aggregate_id = 'sub_abc'
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY sequence_number ASC;

-- Application code replays these events through the Subscription aggregate
-- to reconstruct the state as of that timestamp.
```

### Revenue recognition replay with updated rules

```sql
-- Replay all contract events for Q1 2026 through updated ASC 606 rules
SELECT e.*
FROM events e
WHERE e.aggregate_type = 'contract'
  AND e.ce_time BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.aggregate_id, e.sequence_number ASC;

-- Application replays each contract's event stream through the new
-- revenue recognition engine, generating corrected RevenueRecognized events
```

### Build dunning ML training dataset from events

```sql
-- Extract payment attempt features for ML training
SELECT
    e1.payload->>'customer_id' AS customer_id,
    e1.payload->>'amount' AS amount,
    e1.payload->>'payment_method_type' AS method_type,
    e1.payload->>'card_last4' AS card_last4,
    EXTRACT(DOW FROM e1.ce_time) AS day_of_week,
    EXTRACT(HOUR FROM e1.ce_time) AS hour_of_day,
    e1.payload->>'dunning_attempt' AS attempt_number,
    CASE
        WHEN e2.event_type = 'PaymentSucceeded' THEN 'success'
        ELSE 'failure'
    END AS outcome
FROM events e1
LEFT JOIN events e2 ON e2.aggregate_id = e1.aggregate_id
    AND e2.sequence_number = e1.sequence_number + 1
    AND e2.event_type = 'PaymentSucceeded'
WHERE e1.event_type = 'PaymentAttempted'
ORDER BY e1.ce_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | Events (partitioned), snapshots, event type registry |
| Event Publishing | 2 | Outbox, webhook endpoints |
| Catalog (Reference Data) | 4 | Products, plans, meters, dunning campaigns |
| Customer Projection | 1 | Denormalized customer with metrics |
| Subscription Projection | 1 | Denormalized with embedded items |
| Invoice Projection | 1 | Denormalized with embedded line items |
| Payment Projection | 1 | With AI dunning fields |
| Revenue Projection | 2 | Schedules, waterfall |
| Credit Projection | 1 | Wallet balance |
| Usage Projection | 1 | Period summaries |
| Dunning Projection | 2 | Current state, ML training data |
| **Total** | **19** | Events table is the source of truth; projections are disposable |

---

## Key Design Decisions

1. **Single `events` table as source of truth** -- all domain state derives from the append-only event log. Projections can be dropped and rebuilt from events at any time, making the system self-healing and auditable.

2. **CloudEvents envelope for all events** -- both external usage events and internal domain events share the same CloudEvents structure (`ce_id`, `ce_source`, `ce_type`, `ce_time`), enabling a unified event processing pipeline.

3. **Aggregate-scoped event sequences** -- `(aggregate_type, aggregate_id, sequence_number)` ensures strict ordering within each entity while allowing concurrent writes across different aggregates.

4. **Snapshots for performance** -- periodic snapshots of aggregate state avoid replaying hundreds of events for long-lived subscriptions. The application loads the latest snapshot and replays only subsequent events.

5. **Crypto-shredding for GDPR** -- PII in event payloads is encrypted with a per-customer key. Right-to-erasure destroys the key, rendering events unreadable without losing the event stream structure.

6. **Projections are disposable** -- every projection table includes `last_event_sequence` to track which events have been processed. Projections can be rebuilt from scratch by replaying from the event store.

7. **Catalog tables are mutable reference data** -- product/plan/meter definitions are not event-sourced (they change infrequently and don't need temporal queries). Plan charges are embedded as JSONB in catalog_plans for read convenience.

8. **Transactional outbox pattern** -- events are written to both the event store and an outbox table in the same database transaction, guaranteeing exactly-once delivery to webhook consumers and downstream projections.

9. **Denormalized projections with embedded JSONB** -- projection tables embed related data (line items in invoices, items in subscriptions) as JSONB arrays to eliminate JOINs on the read path, accepting the trade-off of larger rows for faster queries.

10. **AI training data as a first-class projection** -- `proj_dunning_training_data` materializes feature-engineered records from payment events, ready for ML model training without requiring ad-hoc event log queries.
