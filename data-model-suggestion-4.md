# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Subscription Billing Platform · Created: 2026-05-12

## Philosophy

The graph-relational hybrid model uses standard PostgreSQL relational tables for operational billing data (invoices, payments, usage events) while adding a property graph layer for the complex relationship networks that billing platforms must navigate. Subscription billing is more relationship-heavy than it first appears: customers have hierarchical organizational structures (parent/child accounts), subscriptions link to plans that compose charges that reference meters, invoices tie to payments that flow through payment methods via gateways, revenue contracts chain to performance obligations that fan out to recognition schedules, and credit commits create drawdown relationships across billing periods.

The graph layer is implemented using a generic `graph_nodes` and `graph_edges` table pair in PostgreSQL, using `ltree` for hierarchical paths and recursive CTEs for traversal. This is not a separate graph database -- it is a relational encoding of graph structure that PostgreSQL handles efficiently with GiST indexes on ltree columns. The operational billing tables reference graph nodes where relationships matter (e.g., a customer's `graph_node_id` enables traversing the customer hierarchy), while the graph layer enables queries that would be expensive or impossible in a purely relational model.

This architecture is particularly valuable for billing platforms serving enterprise customers with multi-entity organizational structures, where consolidated billing across subsidiaries, volume-based pricing across business units, and hierarchical credit allocation are common requirements. Zuora, the dominant enterprise billing platform, supports multi-entity billing precisely because enterprise customers need parent/child account hierarchies. A graph-relational model makes these hierarchical relationships first-class rather than bolted-on.

**Best for:** Teams building an enterprise-grade billing platform where customer hierarchies, consolidated billing, complex contract relationships, and AI-powered relationship analysis (churn contagion, usage pattern correlation) are key requirements.

**Trade-offs:**
- (+) Hierarchical customer structures (parent/child, business units) are natural graph operations
- (+) Consolidated billing across organizational trees uses graph traversal, not recursive self-joins
- (+) Relationship-aware AI: churn prediction considers organizational context, not just individual accounts
- (+) Contract dependency graphs enable impact analysis ("what happens if we change this plan?")
- (+) Revenue attribution across hierarchies is a graph operation
- (+) All implemented in PostgreSQL -- no separate graph database infrastructure
- (-) Graph layer adds conceptual complexity for developers unfamiliar with graph patterns
- (-) Two data access patterns (relational SQL + graph traversal) increase learning curve
- (-) Graph traversal queries are harder to optimize than simple JOINs
- (-) ltree paths must be maintained when hierarchy changes (re-parenting operations)
- (-) Overhead of maintaining graph consistency alongside relational tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Contract-to-obligation-to-schedule relationships modeled as graph edges; enables traversal-based revenue waterfall queries |
| ISO 4217 | Currency codes on all monetary columns and graph edge properties |
| ISO 3166-1 | Customer jurisdiction modeled as graph nodes in a geographic hierarchy |
| CloudEvents v1.0 | Usage events with CloudEvents envelope; event attribution traverses customer graph |
| PCI DSS v4.0.1 | Payment methods store only tokenized references in relational tables |
| SEPA | SEPA mandates linked to payment method nodes in the graph |
| LEI (ISO 17442) | Legal Entity Identifier stored on organizational graph nodes for enterprise customers |
| Peppol BIS 3.0 | E-invoicing document references on invoice records |

---

## Graph Foundation Layer

```sql
-- Enable ltree extension for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

-- Generic graph node table
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,
    -- Node types: 'organization', 'customer', 'subscription', 'plan',
    --   'charge', 'meter', 'invoice', 'contract', 'obligation',
    --   'jurisdiction', 'payment_method', 'wallet'
    entity_id       UUID NOT NULL,                   -- FK to the domain table
    label           TEXT NOT NULL,                   -- human-readable label
    path            ltree NOT NULL,                  -- hierarchical path for tree queries
    properties      JSONB NOT NULL DEFAULT '{}',     -- node-level metadata
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_gn_tenant_type ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_gn_entity ON graph_nodes(entity_id);
CREATE INDEX idx_gn_path ON graph_nodes USING gist(path);
CREATE INDEX idx_gn_properties ON graph_nodes USING gin(properties jsonb_path_ops);

-- Generic graph edge table
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type       TEXT NOT NULL,
    -- Edge types: 'parent_of', 'subscribes_to', 'priced_by', 'metered_by',
    --   'billed_to', 'paid_by', 'obligated_under', 'recognized_in',
    --   'credits_from', 'consolidated_into', 'governed_by'
    properties      JSONB NOT NULL DEFAULT '{}',     -- edge-level metadata
    weight          NUMERIC(10,4) DEFAULT 1.0,       -- for weighted graph algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                     -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type, valid_from)
);

CREATE INDEX idx_ge_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_ge_tenant_type ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_ge_active ON graph_edges(edge_type) WHERE valid_to IS NULL;
CREATE INDEX idx_ge_properties ON graph_edges USING gin(properties jsonb_path_ops);
```

## Customer & Organization Hierarchy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    lei             TEXT,                            -- Legal Entity Identifier (ISO 17442)
    org_type        TEXT NOT NULL DEFAULT 'company'
                    CHECK (org_type IN ('company', 'division', 'department', 'team', 'individual')),
    billing_entity  BOOLEAN NOT NULL DEFAULT false,  -- can this org receive invoices?
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    billing_address JSONB,
    tax_config      JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Graph reference
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_tenant ON organizations(tenant_id);
CREATE INDEX idx_orgs_lei ON organizations(lei) WHERE lei IS NOT NULL;

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    organization_id UUID REFERENCES organizations(id),
    external_id     TEXT,
    name            TEXT NOT NULL,
    email           TEXT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    locale          TEXT DEFAULT 'en',
    billing_address JSONB,
    tax_config      JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Graph reference
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_org ON customers(organization_id);

-- Example: Building the customer hierarchy graph
--
-- INSERT INTO graph_nodes (tenant_id, node_type, entity_id, label, path) VALUES
--   ($tenant, 'organization', $acme_id,     'Acme Corp',        'acme'),
--   ($tenant, 'organization', $acme_us_id,  'Acme US',          'acme.us'),
--   ($tenant, 'organization', $acme_eu_id,  'Acme EU',          'acme.eu'),
--   ($tenant, 'customer',     $cust_sf_id,  'Acme SF Office',   'acme.us.sf'),
--   ($tenant, 'customer',     $cust_ny_id,  'Acme NY Office',   'acme.us.ny'),
--   ($tenant, 'customer',     $cust_ldn_id, 'Acme London',      'acme.eu.london');
--
-- INSERT INTO graph_edges (tenant_id, source_node_id, target_node_id, edge_type) VALUES
--   ($tenant, $acme_node,    $acme_us_node,  'parent_of'),
--   ($tenant, $acme_node,    $acme_eu_node,  'parent_of'),
--   ($tenant, $acme_us_node, $cust_sf_node,  'parent_of'),
--   ($tenant, $acme_us_node, $cust_ny_node,  'parent_of'),
--   ($tenant, $acme_eu_node, $cust_ldn_node, 'parent_of');
```

## Product Catalog & Pricing

```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    metadata        JSONB NOT NULL DEFAULT '{}',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL REFERENCES products(id),
    name            TEXT NOT NULL,
    billing_period  TEXT NOT NULL,
    billing_interval_months INT NOT NULL DEFAULT 1,
    trial_days      INT DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'active',
    pricing_config  JSONB NOT NULL DEFAULT '[]',     -- charge definitions with tiers
    metadata        JSONB NOT NULL DEFAULT '{}',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON plans(tenant_id);
CREATE INDEX idx_plans_product ON plans(product_id);

CREATE TABLE meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    event_name      TEXT NOT NULL,
    aggregation_config JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, event_name)
);

-- Graph edges connect plan -> charge -> meter relationships:
-- plan_node --[priced_by]--> charge_node --[metered_by]--> meter_node
-- This enables: "Which meters are affected if we change Plan X?"
```

## Subscriptions

```sql
CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('trialing', 'active', 'paused', 'past_due', 'cancelled', 'expired')),
    currency        CHAR(3) NOT NULL,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    trial_end       TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    billing_anchor  TIMESTAMPTZ NOT NULL,
    overrides       JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Graph reference
    graph_node_id   UUID REFERENCES graph_nodes(id),
    -- Consolidated billing: which org receives the invoice?
    billing_org_id  UUID REFERENCES organizations(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subs_customer ON subscriptions(customer_id);
CREATE INDEX idx_subs_billing_org ON subscriptions(billing_org_id);
CREATE INDEX idx_subs_tenant_status ON subscriptions(tenant_id, status);

-- Graph edges:
-- customer_node --[subscribes_to]--> subscription_node
-- subscription_node --[priced_by]--> plan_node
-- subscription_node --[billed_to]--> org_node (for consolidated billing)
```

## Usage Metering

```sql
CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    subscription_id UUID,
    -- CloudEvents
    ce_id           TEXT NOT NULL,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    quantity        NUMERIC(20,8),
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ce_time);

CREATE UNIQUE INDEX idx_ue_dedup ON usage_events(tenant_id, ce_id);
CREATE INDEX idx_ue_customer ON usage_events(customer_id, ce_time);

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
    breakdown       JSONB,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subscription_id, meter_id, period_start, period_end)
);
```

## Invoicing

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    -- Consolidated billing: invoice target may be an organization, not a customer
    billing_org_id  UUID REFERENCES organizations(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'finalized', 'paid', 'partially_paid', 'past_due', 'void', 'uncollectible')),
    invoice_type    TEXT NOT NULL DEFAULT 'subscription',
    -- Consolidation type
    consolidation_type TEXT DEFAULT 'single'
                    CHECK (consolidation_type IN ('single', 'consolidated', 'child')),
    parent_invoice_id UUID REFERENCES invoices(id),  -- for consolidated invoice children
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
    tax_breakdown   JSONB,
    e_invoice_config JSONB,
    pdf_url         TEXT,
    memo            TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_inv_customer ON invoices(customer_id);
CREATE INDEX idx_inv_billing_org ON invoices(billing_org_id);
CREATE INDEX idx_inv_parent ON invoices(parent_invoice_id);
CREATE INDEX idx_inv_status ON invoices(tenant_id, status);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    charge_key      TEXT,
    description     TEXT NOT NULL,
    line_type       TEXT NOT NULL,
    quantity        NUMERIC(20,8) NOT NULL DEFAULT 1,
    unit_amount     NUMERIC(20,8) NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    usage_detail    JSONB,
    -- Source attribution for consolidated invoices
    source_customer_id UUID REFERENCES customers(id),
    source_subscription_id UUID REFERENCES subscriptions(id),
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_li_invoice ON invoice_line_items(invoice_id);
CREATE INDEX idx_li_source_customer ON invoice_line_items(source_customer_id) WHERE source_customer_id IS NOT NULL;

CREATE TABLE credit_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    credit_note_number TEXT NOT NULL,
    reason          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'issued',
    currency        CHAR(3) NOT NULL,
    total           NUMERIC(20,2) NOT NULL,
    refund_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(20,2) NOT NULL DEFAULT 0,
    items           JSONB NOT NULL DEFAULT '[]',
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, credit_note_number)
);
```

## Payments

```sql
CREATE TABLE payment_methods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    organization_id UUID REFERENCES organizations(id), -- org-level payment methods
    method_type     TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active',
    gateway_config  JSONB NOT NULL,
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pm_customer ON payment_methods(customer_id);
CREATE INDEX idx_pm_org ON payment_methods(organization_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    gateway_response JSONB NOT NULL DEFAULT '{}',
    refunded_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_invoice ON payments(invoice_id);
CREATE INDEX idx_pay_customer ON payments(customer_id);

CREATE TABLE refunds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    credit_note_id  UUID REFERENCES credit_notes(id),
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    reason          TEXT,
    gateway_response JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Credit Ledger

```sql
CREATE TABLE credit_wallets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID REFERENCES customers(id),
    organization_id UUID REFERENCES organizations(id), -- org-level wallets for consolidated credits
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(20,8) NOT NULL DEFAULT 0,
    wallet_config   JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'active',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (customer_id IS NOT NULL OR organization_id IS NOT NULL)
);

CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id       UUID NOT NULL REFERENCES credit_wallets(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('credit', 'debit', 'expiration', 'void', 'transfer')),
    amount          NUMERIC(20,8) NOT NULL,
    balance_after   NUMERIC(20,8) NOT NULL,
    description     TEXT,
    -- Transfer tracking for inter-entity credit movements
    transfer_from_wallet_id UUID REFERENCES credit_wallets(id),
    transfer_to_wallet_id   UUID REFERENCES credit_wallets(id),
    reference_type  TEXT,
    reference_id    UUID,
    idempotency_key TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ct_wallet ON credit_transactions(wallet_id, created_at);
```

## Dunning

```sql
CREATE TABLE dunning_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    rules           JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dunning_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    campaign_id     UUID NOT NULL REFERENCES dunning_campaigns(id),
    step_number     INT NOT NULL,
    attempt_number  INT NOT NULL,
    action_taken    TEXT NOT NULL,
    result          TEXT NOT NULL,
    payment_id      UUID REFERENCES payments(id),
    ai_context      JSONB,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    executed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dun_invoice ON dunning_attempts(invoice_id);
CREATE INDEX idx_dun_scheduled ON dunning_attempts(scheduled_at) WHERE result = 'pending';
```

## Revenue Recognition

```sql
CREATE TABLE revenue_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    contract_number TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    inception_date  DATE NOT NULL,
    total_transaction_price NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    asc606_analysis JSONB NOT NULL DEFAULT '{}',
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

-- Graph edges for revenue:
-- contract_node --[obligated_under]--> obligation_node
-- obligation_node --[recognized_in]--> schedule_node
-- This enables: "Show me all unrecognized revenue across the Acme Corp hierarchy"

CREATE TABLE revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES revenue_contracts(id),
    obligation_key  TEXT NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    recognized      BOOLEAN NOT NULL DEFAULT false,
    recognized_at   TIMESTAMPTZ,
    journal_entry_ref TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rs_contract ON revenue_schedules(contract_id);
CREATE INDEX idx_rs_period ON revenue_schedules(period_start, period_end) WHERE NOT recognized;

CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entry_number    TEXT NOT NULL,
    entry_date      DATE NOT NULL,
    description     TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    source_id       UUID NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    lines           JSONB NOT NULL,
    posted_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entry_number)
);
```

## Webhooks & Audit

```sql
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

CREATE INDEX idx_whd_retry ON webhook_deliveries(next_retry_at) WHERE status = 'pending';

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_al_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_al_actor ON audit_log(tenant_id, actor_id, created_at);
```

---

## Key Graph Query Patterns

### Get all customers in an organizational hierarchy

```sql
-- "Show me all customers under Acme Corp and its subsidiaries"
SELECT c.*
FROM customers c
JOIN graph_nodes gn ON gn.entity_id = c.id AND gn.node_type = 'customer'
WHERE gn.path <@ (
    SELECT path FROM graph_nodes
    WHERE entity_id = $acme_org_id AND node_type = 'organization'
);
```

### Consolidated billing: aggregate usage across hierarchy

```sql
-- "Total API usage across all Acme Corp subsidiaries for May 2026"
WITH org_tree AS (
    SELECT gn.entity_id
    FROM graph_nodes gn
    WHERE gn.path <@ (
        SELECT path FROM graph_nodes
        WHERE entity_id = $acme_org_id AND node_type = 'organization'
    )
    AND gn.node_type IN ('customer', 'organization')
)
SELECT
    m.name AS meter_name,
    SUM(ua.value) AS total_usage,
    COUNT(DISTINCT ua.customer_id) AS active_customers
FROM usage_aggregates ua
JOIN meters m ON m.id = ua.meter_id
WHERE ua.customer_id IN (SELECT entity_id FROM org_tree)
  AND ua.period_start >= '2026-05-01'
  AND ua.period_end <= '2026-06-01'
GROUP BY m.name;
```

### Volume discount: calculate tier based on organizational aggregate

```sql
-- "Acme Corp gets volume pricing based on total org usage, not per-subsidiary"
WITH org_usage AS (
    SELECT SUM(ua.value) AS total_usage
    FROM usage_aggregates ua
    WHERE ua.customer_id IN (
        SELECT gn.entity_id
        FROM graph_nodes gn
        WHERE gn.path <@ (SELECT path FROM graph_nodes WHERE entity_id = $acme_org_id AND node_type = 'organization')
        AND gn.node_type = 'customer'
    )
    AND ua.meter_id = $api_calls_meter_id
    AND ua.period_start = $period_start
)
SELECT total_usage FROM org_usage;
-- Application then applies volume pricing tiers against the aggregate total
```

### Revenue recognition waterfall across hierarchy

```sql
-- "Unrecognized deferred revenue across the entire Acme Corp hierarchy"
WITH org_customers AS (
    SELECT gn.entity_id AS customer_id
    FROM graph_nodes gn
    WHERE gn.path <@ (SELECT path FROM graph_nodes WHERE entity_id = $acme_org_id AND node_type = 'organization')
    AND gn.node_type = 'customer'
)
SELECT
    DATE_TRUNC('month', rs.period_start) AS month,
    SUM(rs.amount) AS deferred_revenue,
    rc.currency
FROM revenue_schedules rs
JOIN revenue_contracts rc ON rc.id = rs.contract_id
WHERE rc.customer_id IN (SELECT customer_id FROM org_customers)
  AND rs.recognized = false
GROUP BY DATE_TRUNC('month', rs.period_start), rc.currency
ORDER BY month;
```

### Impact analysis: what happens if we change a plan?

```sql
-- "Which customers and subscriptions are affected by changing Plan X?"
WITH RECURSIVE impact AS (
    -- Start from the plan node
    SELECT gn.id AS node_id, gn.node_type, gn.entity_id, gn.label, 0 AS depth
    FROM graph_nodes gn
    WHERE gn.entity_id = $plan_id AND gn.node_type = 'plan'

    UNION ALL

    -- Traverse edges to find connected entities
    SELECT gn2.id, gn2.node_type, gn2.entity_id, gn2.label, i.depth + 1
    FROM impact i
    JOIN graph_edges ge ON ge.target_node_id = i.node_id AND ge.valid_to IS NULL
    JOIN graph_nodes gn2 ON gn2.id = ge.source_node_id
    WHERE i.depth < 3  -- limit traversal depth
)
SELECT node_type, entity_id, label
FROM impact
WHERE node_type IN ('subscription', 'customer', 'organization')
ORDER BY depth, node_type;
```

### Churn contagion analysis

```sql
-- "Find organizations where >50% of child accounts have declining usage"
WITH org_children AS (
    SELECT
        parent_gn.entity_id AS org_id,
        parent_gn.label AS org_name,
        child_gn.entity_id AS customer_id
    FROM graph_nodes parent_gn
    JOIN graph_nodes child_gn ON child_gn.path <@ parent_gn.path
        AND child_gn.node_type = 'customer'
    WHERE parent_gn.node_type = 'organization'
      AND parent_gn.tenant_id = $tenant_id
),
usage_trends AS (
    SELECT
        oc.org_id,
        oc.customer_id,
        CASE WHEN ua_current.value < ua_prev.value * 0.8 THEN true ELSE false END AS declining
    FROM org_children oc
    LEFT JOIN usage_aggregates ua_current ON ua_current.customer_id = oc.customer_id
        AND ua_current.period_start = $current_period_start
    LEFT JOIN usage_aggregates ua_prev ON ua_prev.customer_id = oc.customer_id
        AND ua_prev.period_start = $previous_period_start
        AND ua_prev.meter_id = ua_current.meter_id
)
SELECT
    org_id,
    COUNT(*) AS total_customers,
    COUNT(*) FILTER (WHERE declining) AS declining_customers,
    ROUND(COUNT(*) FILTER (WHERE declining)::NUMERIC / COUNT(*)::NUMERIC, 2) AS decline_ratio
FROM usage_trends
GROUP BY org_id
HAVING COUNT(*) FILTER (WHERE declining)::NUMERIC / COUNT(*)::NUMERIC > 0.5
ORDER BY decline_ratio DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Foundation | 2 | Nodes (with ltree), edges (with temporal validity) |
| Organizations & Customers | 2 | Separate org entity for hierarchy modeling |
| Product Catalog | 3 | Products, plans (JSONB pricing), meters |
| Subscriptions | 1 | With org-level billing reference |
| Usage Metering | 2 | Events (partitioned), aggregates |
| Invoicing | 3 | Invoices (with consolidation), line items, credit notes |
| Payments | 3 | Methods (org-level), payments, refunds |
| Credit Ledger | 2 | Wallets (org-level), transactions (with transfers) |
| Dunning | 2 | Campaigns, attempts |
| Revenue Recognition | 3 | Contracts, schedules, journal entries |
| Webhooks & Audit | 3 | Endpoints, deliveries, audit log (partitioned) |
| **Total** | **26** | Plus 2 graph tables that encode all relationships |

---

## Key Design Decisions

1. **Generic graph layer in PostgreSQL** -- `graph_nodes` and `graph_edges` tables with ltree paths provide graph traversal capabilities without requiring a separate graph database (Neo4j, etc.). This keeps the infrastructure simple while enabling hierarchical and relationship queries.

2. **Temporal edges with `valid_from`/`valid_to`** -- graph relationships are versioned, enabling historical queries ("who was the billing entity for this customer in Q1?") and supporting organizational restructuring without data loss.

3. **Organization as first-class entity** -- unlike the other models that only have `customers`, this model introduces `organizations` as a separate entity that can form hierarchies. This is essential for enterprise consolidated billing where an invoice aggregates charges across multiple child accounts.

4. **Consolidated invoice model** -- invoices have `consolidation_type` (single/consolidated/child) and `parent_invoice_id`, enabling a parent invoice that rolls up line items from child invoices across an organizational hierarchy.

5. **Credit transfers between wallets** -- `credit_transactions` supports `transfer` type with `transfer_from_wallet_id`/`transfer_to_wallet_id`, enabling credit reallocation within an organizational hierarchy (e.g., parent company distributes credits to subsidiaries).

6. **Organization-level payment methods** -- payment methods can be attached to an organization, not just a customer. This supports scenarios where a parent company pays for all subsidiaries with a single payment method.

7. **LEI (ISO 17442) on organizations** -- Legal Entity Identifiers enable unambiguous identification of enterprise customers and integration with financial regulatory systems.

8. **Graph-powered impact analysis** -- traversing the plan->subscription->customer->organization chain via graph edges enables "what-if" queries for pricing changes, plan deprecation, and migration planning.

9. **Source attribution on invoice line items** -- `source_customer_id` and `source_subscription_id` on line items track which subsidiary generated each charge in a consolidated invoice, essential for internal cost allocation.

10. **Churn contagion via graph traversal** -- the organizational graph enables AI models to detect churn signals that propagate through customer hierarchies (e.g., if 3 of 5 subsidiaries show usage decline, the parent account is at risk), a pattern impossible to detect with flat customer tables.
