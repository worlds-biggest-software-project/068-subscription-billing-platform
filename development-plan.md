# Subscription Billing Platform -- Development Plan

> Project: Subscription Billing Platform (Candidate #68)
> Created: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Data Model](#phase-1-foundation--core-data-model)
5. [Phase 2: Product Catalog & Pricing Engine](#phase-2-product-catalog--pricing-engine)
6. [Phase 3: Subscription Lifecycle Engine](#phase-3-subscription-lifecycle-engine)
7. [Phase 4: Usage Metering & Event Ingestion](#phase-4-usage-metering--event-ingestion)
8. [Phase 5: Invoicing & Tax Integration](#phase-5-invoicing--tax-integration)
9. [Phase 6: Payment Processing & Gateway Integration](#phase-6-payment-processing--gateway-integration)
10. [Phase 7: Dunning & Payment Recovery](#phase-7-dunning--payment-recovery)
11. [Phase 8: Revenue Recognition (ASC 606 / IFRS 15)](#phase-8-revenue-recognition-asc-606--ifrs-15)
12. [Phase 9: Credit Ledger & Commit-Based Contracts](#phase-9-credit-ledger--commit-based-contracts)
13. [Phase 10: Webhooks, Audit & API Finalization](#phase-10-webhooks-audit--api-finalization)
14. [Phase 11: AI-Native Features](#phase-11-ai-native-features)
15. [Phase 12: Self-Service Portal & Integrations](#phase-12-self-service-portal--integrations)
16. [Definition of Done (Global)](#definition-of-done-global)

---

## Technology Decisions

### Language & Runtime: TypeScript on Node.js

**Rationale:** TypeScript provides static typing critical for financial calculations and complex domain logic. The Node.js ecosystem has the strongest SDK support from payment gateways (Stripe, Adyen, Braintree, GoCardless all publish first-party Node SDKs). Lago's client libraries, Chargebee's SDK, and Orb's SDK (`orb-billing` on npm) are all TypeScript-first. Event-driven architectures map naturally to Node's async model. The developer hiring pool for TypeScript is significantly larger than for Go or Rust alternatives.

### Database: PostgreSQL 16+

**Rationale:** All four data model suggestions are PostgreSQL-native. PostgreSQL provides the required features: JSONB with GIN indexes for flexible pricing configuration, table partitioning for high-volume usage events and audit logs, `NUMERIC(20,8)` precision for financial calculations, and `ltree` extension for future hierarchy support. Every competing product in this space (Lago, Kill Bill, Stripe internally) uses PostgreSQL or a PostgreSQL-compatible store. The hybrid relational + JSONB approach (Data Model Suggestion 3) is selected as the primary architecture, with event sourcing patterns (from Suggestion 2) adopted selectively for the audit log and revenue recognition modules where temporal queries and replay are valuable.

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with Selective Event Sourcing (Suggestion 2)

**Rationale:** The hybrid model reduces table count by ~30% compared to full normalization while preserving referential integrity on core financial relationships. JSONB columns for pricing configuration, gateway responses, tax breakdowns, and AI decision context accommodate the inevitable variability across payment gateways, jurisdictions, and evolving AI models without schema migrations. The event-sourced audit trail from Suggestion 2 is adopted for the audit log and revenue recognition subsystems, where temporal replay and immutability are compliance requirements (ASC 606, SOC 2). The graph-relational layer from Suggestion 4 is deferred to a post-MVP phase for enterprise consolidated billing.

### API Framework: Fastify

**Rationale:** Fastify provides ~3x throughput over Express with built-in JSON schema validation, which aligns with the OpenAPI 3.1 spec requirement. Native support for request validation, serialization, and plugin architecture maps well to a billing platform's need for middleware (authentication, tenant isolation, rate limiting). The `@fastify/swagger` plugin auto-generates OpenAPI specs from route schemas.

### Queue / Background Jobs: BullMQ (Redis-backed)

**Rationale:** Billing platforms require reliable background job processing for invoice generation, payment retries, dunning step execution, webhook delivery, and usage aggregation. BullMQ provides delayed jobs (for dunning schedules), rate limiting (for payment gateway API limits), job prioritization, and dead-letter queues. Redis as the backing store is operationally simpler than Kafka for the initial scale target.

### Event Streaming (Usage Metering): Apache Kafka (via KafkaJS)

**Rationale:** Usage event ingestion at the target throughput (millions of events/day, scaling to billions/month for AI/cloud customers) requires a durable, high-throughput event stream. Kafka provides exactly-once semantics, partitioned consumption for parallel aggregation, and replay capability for reprocessing. OpenMeter and Metronome both use Kafka-based architectures for metering. KafkaJS is the mature Node.js client.

### Cache / Session: Redis 7+

**Rationale:** Shared between BullMQ (job queue), API rate limiting, session caching, and real-time usage counters. Redis Streams can serve as a lightweight alternative to Kafka for lower-volume deployments.

### Object Storage: S3-compatible (MinIO for self-hosted)

**Rationale:** PDF invoice storage, UBL XML e-invoice documents, and audit log exports require object storage. S3 API compatibility ensures portability between self-hosted (MinIO) and cloud (AWS S3, GCS, R2) deployments.

### Containerization: Docker + Kubernetes Helm charts

**Rationale:** Self-hosted deployment is a stated differentiator vs. Zuora/Chargebee. Docker Compose for development and single-server deployment; Kubernetes Helm charts for production. Follows Lago's deployment model.

### Testing: Vitest + Testcontainers + Playwright

**Rationale:** Vitest for unit and integration tests (faster than Jest, native ESM support). Testcontainers for database integration tests against real PostgreSQL instances. Playwright for end-to-end API testing and future self-service portal testing.

### Licence: Apache 2.0

**Rationale:** Apache 2.0 is the most permissive viable licence for an open-source billing platform. It allows ISVs to embed the platform in commercial products without copyleft obligations (unlike Lago's AGPLv3). Kill Bill uses Apache 2.0 successfully. This removes the primary adoption friction that Lago faces with AGPLv3.

---

## Project Structure

```
subscription-billing-platform/
├── docker/
│   ├── docker-compose.yml              # Local development stack
│   ├── docker-compose.test.yml         # Integration test stack
│   └── Dockerfile                      # Production image
├── helm/
│   └── subscription-billing/           # Kubernetes Helm chart
├── packages/
│   ├── core/                           # Shared domain types, errors, utilities
│   │   ├── src/
│   │   │   ├── types/                  # Domain type definitions
│   │   │   ├── errors/                 # Structured error types (RFC 7807)
│   │   │   ├── money/                  # Money arithmetic (ISO 4217)
│   │   │   └── validation/             # Shared validation schemas
│   │   └── package.json
│   ├── db/                             # Database migrations and query layer
│   │   ├── migrations/                 # Versioned SQL migrations
│   │   ├── seeds/                      # Development seed data
│   │   ├── src/
│   │   │   ├── repositories/           # Data access layer per entity
│   │   │   └── connection.ts           # Connection pool management
│   │   └── package.json
│   ├── api/                            # Fastify REST API
│   │   ├── src/
│   │   │   ├── routes/                 # Route handlers grouped by domain
│   │   │   ├── middleware/             # Auth, tenant isolation, rate limiting
│   │   │   ├── plugins/               # Fastify plugins
│   │   │   └── server.ts              # Server entry point
│   │   └── package.json
│   ├── engine/                         # Billing engine (subscription, pricing, invoicing)
│   │   ├── src/
│   │   │   ├── subscription/           # Subscription lifecycle logic
│   │   │   ├── pricing/               # Pricing calculation engine
│   │   │   ├── invoicing/             # Invoice generation
│   │   │   ├── metering/              # Usage aggregation
│   │   │   ├── dunning/               # Dunning orchestration
│   │   │   ├── revenue/               # Revenue recognition (ASC 606)
│   │   │   └── credits/               # Credit ledger
│   │   └── package.json
│   ├── gateway/                        # Payment gateway abstraction layer
│   │   ├── src/
│   │   │   ├── interface.ts            # Gateway interface contract
│   │   │   ├── stripe/                # Stripe adapter
│   │   │   ├── adyen/                 # Adyen adapter
│   │   │   ├── braintree/             # Braintree adapter
│   │   │   └── gocardless/            # GoCardless adapter (SEPA)
│   │   └── package.json
│   ├── webhooks/                       # Outbound webhook delivery system
│   │   ├── src/
│   │   │   ├── dispatcher.ts           # Webhook dispatch logic
│   │   │   ├── signer.ts             # HMAC signature generation
│   │   │   └── retry.ts              # Retry with exponential backoff
│   │   └── package.json
│   ├── workers/                        # Background job processors
│   │   ├── src/
│   │   │   ├── invoice-generator.ts
│   │   │   ├── payment-processor.ts
│   │   │   ├── dunning-scheduler.ts
│   │   │   ├── usage-aggregator.ts
│   │   │   ├── webhook-dispatcher.ts
│   │   │   └── revenue-recognizer.ts
│   │   └── package.json
│   └── ai/                            # AI/ML modules
│       ├── src/
│       │   ├── dunning/               # AI dunning optimization
│       │   ├── churn/                 # Churn prediction
│       │   ├── pricing/               # Pricing recommendation
│       │   └── revenue/               # ASC 606 contract classification
│       └── package.json
├── openapi/
│   └── openapi.yaml                    # OpenAPI 3.1 specification
├── docs/
│   ├── architecture.md
│   └── api-guide.md
├── turbo.json                          # Turborepo configuration
├── pnpm-workspace.yaml                 # pnpm workspace definition
├── tsconfig.base.json                  # Shared TypeScript configuration
└── vitest.config.ts                    # Test configuration
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Data Model
  │
  ├──> Phase 2: Product Catalog & Pricing Engine
  │      │
  │      ├──> Phase 3: Subscription Lifecycle Engine
  │      │      │
  │      │      ├──> Phase 4: Usage Metering & Event Ingestion
  │      │      │      │
  │      │      │      └──> Phase 5: Invoicing & Tax Integration
  │      │      │             │
  │      │      │             ├──> Phase 6: Payment Processing & Gateway Integration
  │      │      │             │      │
  │      │      │             │      └──> Phase 7: Dunning & Payment Recovery
  │      │      │             │
  │      │      │             └──> Phase 8: Revenue Recognition (ASC 606)
  │      │      │
  │      │      └──> Phase 9: Credit Ledger & Commit-Based Contracts
  │      │
  │      └──(Phase 2 also feeds into Phase 5 directly for pricing calculations)
  │
  └──> Phase 10: Webhooks, Audit & API Finalization
         │
         └──(Phase 10 can start after Phase 1, runs in parallel with Phases 2-9,
             finalized after Phase 9)

Phase 11: AI-Native Features
  └── Requires: Phases 7, 8, 9 complete (needs payment history, revenue data, usage data)

Phase 12: Self-Service Portal & Integrations
  └── Requires: Phase 10 complete (needs finalized API)
```

**Parallelism opportunities:**
- Phase 10 (Webhooks/Audit) can begin after Phase 1 and progress incrementally alongside Phases 2-9
- Phases 8 (Revenue Recognition) and 9 (Credit Ledger) can run in parallel after Phase 5
- Phase 4 (Usage Metering) Kafka infrastructure setup can begin during Phase 3

---

## Phase 1: Foundation & Core Data Model

**Goal:** Establish the project skeleton, tooling, core data model, multi-tenant isolation, and the foundational packages that all subsequent phases depend on.

**Duration estimate:** 3-4 weeks

### Task 1.1: Project Scaffolding & Monorepo Setup

**What:** Initialize the pnpm workspace monorepo with Turborepo, configure TypeScript, ESLint, Prettier, and Vitest across all packages. Create the Docker Compose development stack with PostgreSQL 16, Redis 7, and MinIO.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "test:integration": { "dependsOn": ["build"], "env": ["DATABASE_URL"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}

// pnpm-workspace.yaml
packages:
  - "packages/*"

// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

```yaml
# docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: billing
      POSTGRES_USER: billing
      POSTGRES_PASSWORD: billing_dev
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

**Testing:**
- `pnpm install` completes without errors
- `pnpm turbo build` builds all packages
- `pnpm turbo typecheck` passes across all packages
- `docker compose up -d` starts all services; health checks pass
- PostgreSQL accepts connections on port 5432
- Redis responds to PING on port 6379
- MinIO console accessible on port 9001

### Task 1.2: Core Domain Types & Money Arithmetic

**What:** Implement the `packages/core` package with domain types, Money value object (ISO 4217 compliant), structured error types (RFC 7807), and shared validation schemas.

**Design:**

```typescript
// packages/core/src/money/money.ts
import { z } from "zod";

export const CurrencyCode = z.string().length(3).toUpperCase();
export type CurrencyCode = z.infer<typeof CurrencyCode>;

// Currency minor unit definitions (ISO 4217)
const CURRENCY_EXPONENTS: Record<string, number> = {
  USD: 2, EUR: 2, GBP: 2, JPY: 0, BHD: 3, KWD: 3,
  // ... all ISO 4217 currencies
};

export class Money {
  private constructor(
    /** Amount in minor units (cents for USD/EUR) */
    readonly amountMinor: bigint,
    readonly currency: CurrencyCode,
    readonly exponent: number,
  ) {}

  static fromMinor(amountMinor: bigint, currency: CurrencyCode): Money {
    const exponent = CURRENCY_EXPONENTS[currency] ?? 2;
    return new Money(amountMinor, currency, exponent);
  }

  static fromDecimal(amount: string, currency: CurrencyCode): Money {
    const exponent = CURRENCY_EXPONENTS[currency] ?? 2;
    const [whole, frac = ""] = amount.split(".");
    const paddedFrac = frac.padEnd(exponent, "0").slice(0, exponent);
    const minor = BigInt(whole + paddedFrac);
    return new Money(minor, currency, exponent);
  }

  toDecimal(): string {
    const str = this.amountMinor.toString().padStart(this.exponent + 1, "0");
    if (this.exponent === 0) return str;
    const whole = str.slice(0, -this.exponent);
    const frac = str.slice(-this.exponent);
    return `${whole}.${frac}`;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amountMinor + other.amountMinor, this.currency, this.exponent);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amountMinor - other.amountMinor, this.currency, this.exponent);
  }

  multiply(quantity: bigint): Money {
    return new Money(this.amountMinor * quantity, this.currency, this.exponent);
  }

  /** Allocate amount across N parts with remainder distribution */
  allocate(ratios: number[]): Money[] {
    const total = ratios.reduce((a, b) => a + b, 0);
    const results: Money[] = [];
    let remainder = this.amountMinor;
    for (const ratio of ratios) {
      const share = (this.amountMinor * BigInt(Math.round(ratio * 1000))) / BigInt(Math.round(total * 1000));
      results.push(new Money(share, this.currency, this.exponent));
      remainder -= share;
    }
    // Distribute remainder to first allocation (banker's convention)
    if (remainder !== 0n) {
      results[0] = new Money(results[0].amountMinor + remainder, this.currency, this.exponent);
    }
    return results;
  }

  isZero(): boolean { return this.amountMinor === 0n; }
  isPositive(): boolean { return this.amountMinor > 0n; }
  isNegative(): boolean { return this.amountMinor < 0n; }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new CurrencyMismatchError(this.currency, other.currency);
    }
  }
}
```

```typescript
// packages/core/src/errors/billing-error.ts

/** RFC 7807 Problem Details error base */
export class BillingError extends Error {
  constructor(
    readonly type: string,
    readonly title: string,
    readonly status: number,
    readonly detail: string,
    readonly instance?: string,
    readonly extensions?: Record<string, unknown>,
  ) {
    super(detail);
    this.name = "BillingError";
  }

  toJSON() {
    return {
      type: `https://billing.example.com/errors/${this.type}`,
      title: this.title,
      status: this.status,
      detail: this.detail,
      ...(this.instance && { instance: this.instance }),
      ...this.extensions,
    };
  }
}

export class CurrencyMismatchError extends BillingError {
  constructor(a: string, b: string) {
    super("currency-mismatch", "Currency Mismatch", 422,
      `Cannot operate on ${a} and ${b}: currencies must match`);
  }
}

export class EntityNotFoundError extends BillingError {
  constructor(entity: string, id: string) {
    super("entity-not-found", "Entity Not Found", 404,
      `${entity} with id '${id}' was not found`);
  }
}

export class InvalidStateTransitionError extends BillingError {
  constructor(entity: string, from: string, to: string) {
    super("invalid-state-transition", "Invalid State Transition", 422,
      `Cannot transition ${entity} from '${from}' to '${to}'`);
  }
}
```

**Testing:**
- Money.fromDecimal("99.99", "USD").toDecimal() === "99.99"
- Money.fromDecimal("100", "JPY").amountMinor === 100n (zero exponent currency)
- Money.fromDecimal("1.234", "BHD").toDecimal() === "1.234" (three-decimal currency)
- Money addition with currency mismatch throws CurrencyMismatchError
- Money.allocate([1,1,1]) for $10.00 yields [$3.34, $3.33, $3.33] (remainder to first)
- BillingError.toJSON() produces RFC 7807 compliant JSON structure
- 100 randomized Money arithmetic round-trip tests (fromDecimal -> operations -> toDecimal) with no precision loss

### Task 1.3: Database Migration Framework & Core Schema

**What:** Set up database migration tooling (using `postgres-migrations` or `dbmate`) and create the initial migration for tenants, customers, and API credentials tables. Implement the connection pool with tenant-aware row-level security.

**Design:**

```sql
-- migrations/001_foundation.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'cancelled')),
    billing_config  JSONB NOT NULL DEFAULT '{}',
    data_retention_config JSONB NOT NULL DEFAULT '{}',
    feature_flags   JSONB NOT NULL DEFAULT '{}',
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
    billing_address JSONB NOT NULL DEFAULT '{}',
    tax_config      JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_email ON customers(tenant_id, email);
CREATE INDEX idx_customers_metadata ON customers USING gin(metadata jsonb_path_ops);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    key_prefix      TEXT NOT NULL,           -- 'sk_live_' or 'sk_test_'
    key_hash        TEXT NOT NULL,            -- bcrypt hash of the full key
    name            TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);

-- Row-level security policies
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_customers ON customers
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

```typescript
// packages/db/src/connection.ts
import { Pool, PoolClient } from "pg";

export class DatabasePool {
  private pool: Pool;

  constructor(connectionString: string) {
    this.pool = new Pool({
      connectionString,
      max: 20,
      idleTimeoutMillis: 30_000,
      connectionTimeoutMillis: 5_000,
    });
  }

  /** Execute a query within a tenant context (sets RLS parameter) */
  async withinTenant<T>(tenantId: string, fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.pool.connect();
    try {
      await client.query("SET app.current_tenant_id = $1", [tenantId]);
      return await fn(client);
    } finally {
      await client.query("RESET app.current_tenant_id");
      client.release();
    }
  }

  async withTransaction<T>(tenantId: string, fn: (client: PoolClient) => Promise<T>): Promise<T> {
    return this.withinTenant(tenantId, async (client) => {
      await client.query("BEGIN");
      try {
        const result = await fn(client);
        await client.query("COMMIT");
        return result;
      } catch (err) {
        await client.query("ROLLBACK");
        throw err;
      }
    });
  }
}
```

**Testing:**
- Migration runs successfully against a clean PostgreSQL 16 database
- Migration is idempotent (running twice does not error)
- Inserting a customer with an invalid tenant_id FK fails with constraint violation
- RLS policy: query with tenant A's context returns only tenant A's customers
- RLS policy: query without tenant context returns zero rows
- Connection pool handles 50 concurrent tenant-scoped queries without deadlock
- `api_keys` table correctly stores and retrieves bcrypt-hashed keys
- `UNIQUE (tenant_id, external_id)` constraint prevents duplicate external IDs within a tenant

### Task 1.4: Fastify API Server Skeleton with Authentication

**What:** Create the `packages/api` Fastify server with tenant-aware API key authentication, request validation, RFC 7807 error responses, OpenAPI spec generation, and health check endpoints.

**Design:**

```typescript
// packages/api/src/server.ts
import Fastify from "fastify";
import fastifySwagger from "@fastify/swagger";
import fastifySwaggerUI from "@fastify/swagger-ui";
import { authPlugin } from "./middleware/auth.js";
import { tenantPlugin } from "./middleware/tenant.js";
import { errorHandler } from "./middleware/error-handler.js";
import { healthRoutes } from "./routes/health.js";

export async function buildServer() {
  const app = Fastify({
    logger: true,
    genReqId: () => crypto.randomUUID(),
  });

  // OpenAPI 3.1 spec generation
  await app.register(fastifySwagger, {
    openapi: {
      openapi: "3.1.0",
      info: {
        title: "Subscription Billing Platform API",
        version: "1.0.0",
        description: "AI-native subscription billing, metering, and revenue recognition",
      },
      components: {
        securitySchemes: {
          apiKey: {
            type: "http",
            scheme: "bearer",
            description: "API key (e.g., sk_live_xxxxx)",
          },
        },
      },
    },
  });

  // Global error handler producing RFC 7807 responses
  app.setErrorHandler(errorHandler);

  // Authentication & tenant isolation
  await app.register(authPlugin);
  await app.register(tenantPlugin);

  // Routes
  await app.register(healthRoutes, { prefix: "/health" });

  return app;
}

// packages/api/src/middleware/error-handler.ts
import { FastifyError, FastifyReply, FastifyRequest } from "fastify";
import { BillingError } from "@billing/core";

export function errorHandler(error: FastifyError, request: FastifyRequest, reply: FastifyReply) {
  if (error instanceof BillingError) {
    return reply.status(error.status).header("content-type", "application/problem+json").send(error.toJSON());
  }

  // Fastify validation errors
  if (error.validation) {
    return reply.status(422).header("content-type", "application/problem+json").send({
      type: "https://billing.example.com/errors/validation-error",
      title: "Validation Error",
      status: 422,
      detail: error.message,
      errors: error.validation,
    });
  }

  // Unhandled errors
  request.log.error(error);
  return reply.status(500).header("content-type", "application/problem+json").send({
    type: "https://billing.example.com/errors/internal-error",
    title: "Internal Server Error",
    status: 500,
    detail: "An unexpected error occurred",
  });
}
```

**Testing:**
- `GET /health` returns `{ "status": "ok", "version": "1.0.0" }` with 200
- `GET /health/ready` checks PostgreSQL and Redis connectivity, returns 503 if either is down
- Request without `Authorization` header returns 401 with RFC 7807 body
- Request with invalid API key returns 401 with `type: "authentication-error"`
- Request with valid API key sets `request.tenantId` correctly for downstream handlers
- `GET /docs` serves Swagger UI with OpenAPI 3.1 spec
- Request with malformed JSON body returns 400 with RFC 7807 validation error
- Rate limiting returns 429 with `Retry-After` header

### Task 1.5: CI/CD Pipeline

**What:** Configure GitHub Actions for CI (lint, typecheck, unit tests, integration tests with Testcontainers, Docker image build) and CD (push to container registry on tag).

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint typecheck

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo test

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: billing_test
          POSTGRES_USER: billing
          POSTGRES_PASSWORD: billing_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo test:integration
        env:
          DATABASE_URL: postgres://billing:billing_test@localhost:5432/billing_test
          REDIS_URL: redis://localhost:6379
```

**Testing:**
- CI pipeline passes on a clean checkout
- Lint step catches TypeScript strict mode violations
- Unit tests run without external dependencies
- Integration tests connect to PostgreSQL and Redis service containers
- Docker build produces a valid image under 500MB
- Pipeline fails fast if lint errors are present (does not proceed to tests)

### Phase 1 -- Definition of Done

- [ ] Monorepo builds and type-checks with zero errors
- [ ] Docker Compose starts PostgreSQL 16, Redis 7, and MinIO
- [ ] Database migration creates tenants, customers, and api_keys tables
- [ ] Row-level security enforces tenant isolation at the database level
- [ ] Money class handles all ISO 4217 currency exponents without precision loss
- [ ] API server starts, serves OpenAPI 3.1 spec, authenticates via API keys
- [ ] All error responses conform to RFC 7807 Problem Details format
- [ ] CI pipeline runs lint, typecheck, unit tests, and integration tests
- [ ] Test coverage for `packages/core` exceeds 95%

---

## Phase 2: Product Catalog & Pricing Engine

**Goal:** Implement the product catalog (products, plans, pricing configurations) and the pricing calculation engine that evaluates flat, per-unit, tiered, graduated, volume, and percentage pricing models.

**Duration estimate:** 2-3 weeks

### Task 2.1: Product & Plan CRUD

**What:** Create the `products` and `plans` database tables, repository layer, and REST API endpoints. Plans store pricing configuration as JSONB (hybrid model).

**Design:**

```sql
-- migrations/002_product_catalog.sql

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
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_products ON products
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    name            TEXT NOT NULL,
    description     TEXT,
    billing_period  TEXT NOT NULL CHECK (billing_period IN ('monthly','quarterly','semi_annual','annual','custom')),
    billing_interval_months INT NOT NULL DEFAULT 1,
    trial_days      INT DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','archived','grandfathered')),
    pricing_config  JSONB NOT NULL DEFAULT '[]',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON plans(tenant_id);
CREATE INDEX idx_plans_product ON plans(product_id);
CREATE INDEX idx_plans_pricing ON plans USING gin(pricing_config jsonb_path_ops);
ALTER TABLE plans ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_plans ON plans
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

```typescript
// packages/api/src/routes/plans.ts (route schema example)
const createPlanSchema = {
  body: {
    type: "object",
    required: ["name", "product_id", "billing_period", "pricing_config"],
    properties: {
      name: { type: "string", minLength: 1, maxLength: 255 },
      product_id: { type: "string", format: "uuid" },
      billing_period: { type: "string", enum: ["monthly", "quarterly", "semi_annual", "annual"] },
      billing_interval_months: { type: "integer", minimum: 1, default: 1 },
      trial_days: { type: "integer", minimum: 0, default: 0 },
      currency: { type: "string", pattern: "^[A-Z]{3}$", default: "USD" },
      pricing_config: {
        type: "array",
        items: {
          type: "object",
          required: ["key", "name", "charge_type", "pricing_model"],
          properties: {
            key: { type: "string" },
            name: { type: "string" },
            charge_type: { type: "string", enum: ["recurring", "usage", "one_time", "setup"] },
            pricing_model: { type: "string", enum: ["flat", "per_unit", "tiered", "volume", "graduated", "percentage", "package"] },
            amount: { type: "number" },
            meter_id: { type: "string", format: "uuid" },
            tiers: { type: "array" },
            min_amount: { type: "number" },
            max_amount: { type: "number" },
            included_units: { type: "number", default: 0 },
          },
        },
      },
    },
  },
};
```

**Testing:**
- `POST /v1/products` creates a product; response includes `id`, `created_at`
- `GET /v1/products` returns paginated list filtered by tenant
- `POST /v1/plans` with flat pricing config creates plan successfully
- `POST /v1/plans` with tiered pricing config validates tier boundaries (no gaps, no overlaps)
- `POST /v1/plans` with invalid pricing_model rejects with 422
- `GET /v1/plans/:id` returns plan with full pricing_config
- `PATCH /v1/plans/:id` updates plan fields; status can transition active -> archived but not archived -> active
- Plan with missing `key` in pricing_config charge rejected with validation error
- Plan with duplicate charge keys rejected with 422

### Task 2.2: Pricing Calculation Engine

**What:** Implement the pricing engine in `packages/engine/src/pricing/` that takes a plan's pricing_config and a set of quantities/usage values and returns computed line items with amounts.

**Design:**

```typescript
// packages/engine/src/pricing/calculator.ts

export interface PricingInput {
  chargeKey: string;
  quantity: number;          // for recurring per-unit: seat count; for usage: metered quantity
  periodFractionUsed?: number; // for proration: fraction of billing period used (0-1)
}

export interface PricingOutput {
  chargeKey: string;
  chargeName: string;
  chargeType: "recurring" | "usage" | "one_time" | "setup";
  quantity: number;
  unitAmount: string;       // decimal string
  amount: Money;
  tierBreakdown?: TierBreakdownEntry[];
}

export interface TierBreakdownEntry {
  tierNumber: number;
  from: number;
  to: number | null;
  quantity: number;
  unitAmount: string;
  amount: Money;
}

export class PricingCalculator {
  calculate(pricingConfig: ChargeConfig[], inputs: PricingInput[], currency: CurrencyCode): PricingOutput[] {
    const outputs: PricingOutput[] = [];

    for (const charge of pricingConfig) {
      const input = inputs.find(i => i.chargeKey === charge.key);
      const quantity = input?.quantity ?? (charge.charge_type === "recurring" ? 1 : 0);

      switch (charge.pricing_model) {
        case "flat":
          outputs.push(this.calculateFlat(charge, quantity, currency, input?.periodFractionUsed));
          break;
        case "per_unit":
          outputs.push(this.calculatePerUnit(charge, quantity, currency, input?.periodFractionUsed));
          break;
        case "tiered":
          outputs.push(this.calculateTiered(charge, quantity, currency));
          break;
        case "graduated":
          outputs.push(this.calculateGraduated(charge, quantity, currency));
          break;
        case "volume":
          outputs.push(this.calculateVolume(charge, quantity, currency));
          break;
        case "percentage":
          outputs.push(this.calculatePercentage(charge, quantity, currency));
          break;
        case "package":
          outputs.push(this.calculatePackage(charge, quantity, currency));
          break;
      }
    }

    return outputs;
  }

  private calculateGraduated(charge: ChargeConfig, quantity: number, currency: CurrencyCode): PricingOutput {
    // Graduated: each tier prices only the units within that tier's range
    const tiers = charge.tiers!;
    let remaining = Math.max(0, quantity - (charge.included_units ?? 0));
    let totalAmount = Money.fromDecimal("0", currency);
    const breakdown: TierBreakdownEntry[] = [];
    let prevTo = 0;

    for (let i = 0; i < tiers.length; i++) {
      const tier = tiers[i];
      const tierSize = tier.to != null ? tier.to - prevTo : Infinity;
      const tierQuantity = Math.min(remaining, tierSize);

      if (tierQuantity <= 0) break;

      const tierAmount = Money.fromDecimal(tier.unit_amount.toString(), currency)
        .multiply(BigInt(Math.ceil(tierQuantity)));
      const flatAmount = tier.flat_amount
        ? Money.fromDecimal(tier.flat_amount.toString(), currency)
        : Money.fromDecimal("0", currency);
      const combinedAmount = tierAmount.add(flatAmount);

      breakdown.push({
        tierNumber: i + 1,
        from: prevTo,
        to: tier.to,
        quantity: tierQuantity,
        unitAmount: tier.unit_amount.toString(),
        amount: combinedAmount,
      });

      totalAmount = totalAmount.add(combinedAmount);
      remaining -= tierQuantity;
      prevTo = tier.to ?? prevTo;
    }

    return {
      chargeKey: charge.key,
      chargeName: charge.name,
      chargeType: charge.charge_type,
      quantity,
      unitAmount: quantity > 0 ? totalAmount.toDecimal() : "0",
      amount: totalAmount,
      tierBreakdown: breakdown,
    };
  }

  // ... calculateFlat, calculatePerUnit, calculateTiered, calculateVolume, calculatePercentage, calculatePackage
}
```

**Testing:**
- Flat pricing: $99.00/mo charge returns exactly $99.00
- Per-unit pricing: 10 seats at $15/seat returns $150.00
- Graduated pricing: 45,000 API calls with tiers [0-10k free, 10k-100k at $0.01] returns $350.00 (35,000 * $0.01)
- Volume pricing: 45,000 units where tier 2 (10k-100k) is $0.008/unit returns $360.00 (all 45,000 at $0.008)
- Tiered pricing: matches graduated but applies highest reached tier to ALL units
- Percentage pricing: 5% of $10,000 revenue returns $500.00
- Package pricing: 1,000-unit packages at $50, 2,500 units consumed returns $150.00 (3 packages)
- Proration: flat $99.00 with 15/30 days used returns $49.50
- Minimum amount enforcement: usage yields $30 but min is $50 returns $50
- Maximum amount cap: usage yields $800 but max is $500 returns $500
- Included units: 10,000 free units with 8,000 used returns $0.00
- Zero-exponent currency (JPY): 9900 JPY flat fee stores and calculates correctly
- Three-decimal currency (BHD): per-unit pricing at 0.500 BHD calculates correctly

### Task 2.3: Coupon & Discount Engine

**What:** Implement the coupons table, CRUD endpoints, and discount application logic that integrates with the pricing calculator.

**Design:**

```sql
-- migrations/003_coupons.sql
CREATE TABLE coupons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    code            TEXT,
    discount_config JSONB NOT NULL,
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

```typescript
// packages/engine/src/pricing/discount.ts
export class DiscountCalculator {
  apply(lineItems: PricingOutput[], coupon: CouponConfig): PricingOutput[] {
    const applicableItems = coupon.applies_to_charges
      ? lineItems.filter(li => coupon.applies_to_charges!.includes(li.chargeKey))
      : lineItems;

    let totalDiscount = Money.fromDecimal("0", lineItems[0].amount.currency);

    for (const item of applicableItems) {
      let discount: Money;
      if (coupon.type === "percentage") {
        discount = Money.fromDecimal(
          (Number(item.amount.toDecimal()) * coupon.value / 100).toFixed(2),
          item.amount.currency,
        );
      } else {
        discount = Money.fromDecimal(coupon.value.toString(), coupon.currency!);
      }

      // Enforce max discount amount
      if (coupon.max_discount_amount) {
        const maxDiscount = Money.fromDecimal(coupon.max_discount_amount.toString(), item.amount.currency);
        if (Number(discount.toDecimal()) > Number(maxDiscount.toDecimal())) {
          discount = maxDiscount;
        }
      }

      totalDiscount = totalDiscount.add(discount);
    }

    return lineItems; // return with discount metadata attached
  }
}
```

**Testing:**
- 20% percentage coupon on $100 recurring charge produces $20 discount
- Fixed $25 coupon applied to $100 charge produces $25 discount
- Coupon with `applies_to_charges: ["base_fee"]` does not discount usage charges
- Coupon with `max_discount_amount: 50` caps a 30% discount on $200 at $50
- Expired coupon (valid_until in the past) is rejected with 422
- Coupon with max_redemptions reached returns 422 on redemption attempt
- `repeating` duration coupon with `duration_months: 3` tracks remaining periods

### Phase 2 -- Definition of Done

- [ ] Products and Plans CRUD endpoints pass all validation and return correct responses
- [ ] Pricing calculator handles all 7 pricing models with correct arithmetic
- [ ] Tiered/graduated/volume pricing produces verifiable tier breakdowns
- [ ] Proration calculates correctly for mid-cycle amendments
- [ ] Coupon discounts apply correctly with percentage, fixed, and capped modes
- [ ] All monetary calculations use the Money class with no floating-point operations
- [ ] Test coverage for `packages/engine/src/pricing/` exceeds 95%

---

## Phase 3: Subscription Lifecycle Engine

**Goal:** Implement the full subscription lifecycle: creation, trial management, amendments (plan change, quantity change), pause/resume, cancellation, renewal, and proration.

**Duration estimate:** 3-4 weeks

### Task 3.1: Subscription CRUD & State Machine

**What:** Create the subscriptions table, implement the subscription state machine with validated transitions, and build the REST API endpoints for subscription management.

**Design:**

```sql
-- migrations/004_subscriptions.sql

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    external_id     TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('trialing','active','paused','past_due','cancelled','expired')),
    currency        CHAR(3) NOT NULL,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    trial_end       TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    billing_anchor  TIMESTAMPTZ NOT NULL,
    overrides       JSONB NOT NULL DEFAULT '{}',
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
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','exhausted','removed')),
    remaining_periods INT,
    amount_remaining NUMERIC(20,8),
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// packages/engine/src/subscription/state-machine.ts

const VALID_TRANSITIONS: Record<string, string[]> = {
  trialing:  ["active", "cancelled"],
  active:    ["paused", "past_due", "cancelled"],
  paused:    ["active", "cancelled"],
  past_due:  ["active", "cancelled"],
  cancelled: [],  // terminal state
  expired:   [],  // terminal state
};

export class SubscriptionStateMachine {
  canTransition(from: string, to: string): boolean {
    return VALID_TRANSITIONS[from]?.includes(to) ?? false;
  }

  transition(subscription: Subscription, targetStatus: string): Subscription {
    if (!this.canTransition(subscription.status, targetStatus)) {
      throw new InvalidStateTransitionError("subscription", subscription.status, targetStatus);
    }
    return {
      ...subscription,
      status: targetStatus,
      ...(targetStatus === "cancelled" && { cancelled_at: new Date() }),
      ...(targetStatus === "paused" && { paused_at: new Date() }),
      updated_at: new Date(),
    };
  }
}
```

**Testing:**
- `POST /v1/subscriptions` with valid customer_id and plan_id creates subscription in `trialing` or `active` state depending on trial_days
- Subscription with trial_days=14 starts in `trialing` with trial_end set correctly
- State transition trialing -> active succeeds
- State transition active -> cancelled succeeds, sets cancelled_at
- State transition cancelled -> active throws InvalidStateTransitionError
- State transition paused -> active succeeds, clears paused_at
- `cancel_at_period_end: true` keeps subscription active until period_end
- Subscription with per-customer price overrides stores them in overrides JSONB
- `GET /v1/subscriptions` filters by status and customer_id
- `GET /v1/subscriptions/:id` returns full subscription with plan details

### Task 3.2: Plan Amendment & Proration

**What:** Implement mid-cycle plan changes (upgrades and downgrades) with prorated credit/charge calculation.

**Design:**

```typescript
// packages/engine/src/subscription/amendment.ts

export interface AmendmentResult {
  subscription: Subscription;
  proratedCredit: Money;       // credit for unused portion of old plan
  proratedCharge: Money;       // charge for remaining portion of new plan
  effectiveDate: Date;
  lineItems: PricingOutput[];  // prorated line items to add to next invoice
}

export class SubscriptionAmendmentService {
  constructor(
    private pricingCalculator: PricingCalculator,
    private subscriptionRepo: SubscriptionRepository,
  ) {}

  async changePlan(
    subscriptionId: string,
    newPlanId: string,
    effectiveDate: Date,
    prorationBehavior: "create_prorations" | "none" | "always_invoice",
  ): Promise<AmendmentResult> {
    const subscription = await this.subscriptionRepo.findById(subscriptionId);
    const oldPlan = await this.planRepo.findById(subscription.plan_id);
    const newPlan = await this.planRepo.findById(newPlanId);

    if (prorationBehavior === "none") {
      // Switch plan at next renewal, no proration
      return {
        subscription: { ...subscription, plan_id: newPlanId },
        proratedCredit: Money.fromDecimal("0", subscription.currency),
        proratedCharge: Money.fromDecimal("0", subscription.currency),
        effectiveDate,
        lineItems: [],
      };
    }

    // Calculate unused days on current plan
    const periodTotalDays = this.daysBetween(subscription.current_period_start, subscription.current_period_end);
    const daysUsed = this.daysBetween(subscription.current_period_start, effectiveDate);
    const daysRemaining = periodTotalDays - daysUsed;
    const remainingFraction = daysRemaining / periodTotalDays;

    // Credit for unused old plan
    const oldPlanCharges = this.pricingCalculator.calculate(
      oldPlan.pricing_config, [], subscription.currency,
    );
    const oldRecurringTotal = this.sumRecurringCharges(oldPlanCharges);
    const credit = Money.fromDecimal(
      (Number(oldRecurringTotal.toDecimal()) * remainingFraction).toFixed(2),
      subscription.currency,
    );

    // Charge for remaining new plan
    const newPlanCharges = this.pricingCalculator.calculate(
      newPlan.pricing_config, [], subscription.currency,
    );
    const newRecurringTotal = this.sumRecurringCharges(newPlanCharges);
    const charge = Money.fromDecimal(
      (Number(newRecurringTotal.toDecimal()) * remainingFraction).toFixed(2),
      subscription.currency,
    );

    return { subscription: { ...subscription, plan_id: newPlanId }, proratedCredit: credit, proratedCharge: charge, effectiveDate, lineItems: [] };
  }
}
```

**Testing:**
- Upgrade from $49/mo to $99/mo on day 15 of 30: credit = $24.50, charge = $49.50, net = $25.00 additional
- Downgrade from $99/mo to $49/mo on day 10 of 30: credit = $66.00, charge = $32.67, net = -$33.33 credit
- Amendment with `prorationBehavior: "none"` produces zero credit and zero charge
- Amendment with `prorationBehavior: "always_invoice"` generates an immediate invoice
- Quantity change (10 seats to 15 seats) prorates the 5 additional seats for remaining days
- Amendment on first day of period: credit = full old plan amount, charge = full new plan amount
- Amendment on last day of period: minimal proration amounts
- Amendment during trial period: no proration (trial is free)
- Two amendments in same billing period: second proration calculates from the first amendment date, not period start

### Task 3.3: Subscription Renewal Worker

**What:** Build the background worker that processes subscription renewals. At `current_period_end`, advance the billing period, trigger invoice generation (Phase 5), and handle trial-to-active conversion.

**Design:**

```typescript
// packages/workers/src/subscription-renewer.ts
import { Queue, Worker } from "bullmq";

export class SubscriptionRenewalWorker {
  constructor(
    private subscriptionRepo: SubscriptionRepository,
    private invoiceService: InvoiceService,
    private redis: Redis,
  ) {}

  async processRenewals(): Promise<void> {
    const now = new Date();

    // Find subscriptions due for renewal
    const dueSubs = await this.subscriptionRepo.findDueForRenewal(now);

    for (const sub of dueSubs) {
      await this.renewSubscription(sub);
    }
  }

  private async renewSubscription(sub: Subscription): Promise<void> {
    if (sub.status === "trialing" && sub.trial_end && sub.trial_end <= new Date()) {
      // Trial ended: convert to active
      await this.subscriptionRepo.updateStatus(sub.id, "active");
    }

    if (sub.cancel_at_period_end) {
      // Subscription was marked for cancellation
      await this.subscriptionRepo.updateStatus(sub.id, "cancelled");
      return;
    }

    // Advance billing period
    const newPeriodStart = sub.current_period_end;
    const newPeriodEnd = this.calculateNextPeriodEnd(newPeriodStart, sub.billing_interval_months);

    await this.subscriptionRepo.advancePeriod(sub.id, newPeriodStart, newPeriodEnd);

    // Queue invoice generation (handled in Phase 5)
    await this.invoiceQueue.add("generate-invoice", {
      subscriptionId: sub.id,
      periodStart: newPeriodStart,
      periodEnd: newPeriodEnd,
    });
  }
}
```

**Testing:**
- Subscription at period_end is renewed: period advances correctly for monthly, quarterly, annual
- Monthly subscription anchored Jan 31 rolls to Feb 28 (not Mar 3)
- Annual subscription renewal advances by exactly 12 months
- Trial expiration transitions subscription from `trialing` to `active`
- `cancel_at_period_end: true` transitions to `cancelled` at renewal time
- Subscription in `paused` state is NOT renewed (skipped)
- Subscription in `past_due` is NOT renewed (handled by dunning)
- Worker processes 1000 subscriptions due for renewal within 30 seconds
- Idempotency: processing the same subscription twice does not create duplicate renewals

### Phase 3 -- Definition of Done

- [ ] Subscription CRUD endpoints fully functional with state machine enforcement
- [ ] All 6 subscription states and valid transitions are enforced
- [ ] Plan amendments produce correct prorated credits and charges
- [ ] Renewal worker advances billing periods and queues invoice generation
- [ ] Trial-to-active conversion and cancel-at-period-end work correctly
- [ ] Billing period advancement handles month-end edge cases (Jan 31 -> Feb 28)
- [ ] Test coverage for subscription lifecycle exceeds 90%

---

## Phase 4: Usage Metering & Event Ingestion

**Goal:** Build the high-throughput usage event ingestion pipeline, meter configuration, deduplication, aggregation, and real-time usage querying.

**Duration estimate:** 3-4 weeks

### Task 4.1: Meter Configuration & Usage Events Schema

**What:** Create the meters and usage_events tables. Meters define how events are matched and aggregated. Usage events follow CloudEvents v1.0 envelope format.

**Design:**

```sql
-- migrations/005_metering.sql

CREATE TABLE meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    event_name      TEXT NOT NULL,
    aggregation_config JSONB NOT NULL,
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
    ce_id           TEXT NOT NULL,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ce_time);

-- Create initial partition
CREATE TABLE usage_events_default PARTITION OF usage_events DEFAULT;

CREATE UNIQUE INDEX idx_usage_events_dedup ON usage_events(tenant_id, ce_id);
CREATE INDEX idx_usage_events_customer ON usage_events(customer_id, ce_time);
CREATE INDEX idx_usage_events_type ON usage_events(ce_type, ce_time);
CREATE INDEX idx_usage_events_props ON usage_events USING gin(properties jsonb_path_ops);

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

**Testing:**
- Meter CRUD: create meter with sum aggregation on `properties.tokens` field
- Meter CRUD: create meter with count aggregation (no field needed)
- Meter CRUD: create meter with unique_count aggregation on `properties.user_id`
- Meter validation: reject meter with `aggregation_config.type = "sum"` but no `field`
- Unique constraint on `(tenant_id, event_name)` prevents duplicate meters

### Task 4.2: Event Ingestion API & Deduplication

**What:** Build the high-throughput event ingestion endpoint that accepts CloudEvents-formatted usage events, deduplicates via `ce_id`, and writes to PostgreSQL (with Kafka integration for scale).

**Design:**

```typescript
// packages/api/src/routes/events.ts

const ingestEventSchema = {
  body: {
    type: "object",
    required: ["events"],
    properties: {
      events: {
        type: "array",
        maxItems: 1000,     // batch limit
        items: {
          type: "object",
          required: ["id", "source", "type", "time", "customer_id"],
          properties: {
            id: { type: "string" },           // CloudEvents ce_id
            source: { type: "string" },
            type: { type: "string" },          // matches meter.event_name
            time: { type: "string", format: "date-time" },
            customer_id: { type: "string" },
            subscription_id: { type: "string" },
            properties: { type: "object" },
          },
        },
      },
    },
  },
};

// packages/engine/src/metering/ingestion.ts

export class EventIngestionService {
  async ingest(tenantId: string, events: UsageEvent[]): Promise<IngestionResult> {
    const results = { accepted: 0, duplicates: 0, errors: 0 };

    // Batch insert with ON CONFLICT for deduplication
    const query = `
      INSERT INTO usage_events (tenant_id, customer_id, subscription_id, ce_id, ce_source, ce_type, ce_time, properties)
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
      ON CONFLICT (tenant_id, ce_id) DO NOTHING
      RETURNING id
    `;

    for (const event of events) {
      const result = await this.db.query(query, [
        tenantId, event.customer_id, event.subscription_id,
        event.id, event.source, event.type, event.time, event.properties,
      ]);

      if (result.rowCount > 0) {
        results.accepted++;
      } else {
        results.duplicates++;
      }
    }

    return results;
  }
}
```

**Testing:**
- `POST /v1/events` with single event returns `{ accepted: 1, duplicates: 0 }`
- `POST /v1/events` with same `ce_id` twice returns `{ accepted: 0, duplicates: 1 }` (idempotent)
- `POST /v1/events` with batch of 500 events processes within 2 seconds
- `POST /v1/events` with batch exceeding 1000 returns 422
- Event with missing required CloudEvents fields returns 422
- Event with future timestamp (>1 hour ahead) is rejected
- Event with `ce_type` not matching any meter is still stored (meters can be created later)
- Concurrent ingestion of events with same `ce_id` from two clients: exactly one is accepted

### Task 4.3: Usage Aggregation Worker

**What:** Build the background worker that aggregates raw usage events into billing-period summaries per subscription per meter. Supports sum, count, max, unique_count, and latest aggregation types.

**Design:**

```typescript
// packages/workers/src/usage-aggregator.ts

export class UsageAggregationWorker {
  async aggregateForPeriod(subscriptionId: string, periodStart: Date, periodEnd: Date): Promise<void> {
    const subscription = await this.subscriptionRepo.findById(subscriptionId);
    const plan = await this.planRepo.findById(subscription.plan_id);

    // Find meters referenced in pricing config
    const usageCharges = plan.pricing_config.filter((c: any) => c.charge_type === "usage");

    for (const charge of usageCharges) {
      const meter = await this.meterRepo.findById(charge.meter_id);

      const aggregateQuery = this.buildAggregateQuery(meter.aggregation_config);
      const result = await this.db.query(aggregateQuery, [
        subscription.tenant_id,
        subscription.customer_id,
        meter.event_name,
        periodStart,
        periodEnd,
      ]);

      await this.db.query(`
        INSERT INTO usage_aggregates (tenant_id, customer_id, subscription_id, meter_id, period_start, period_end, value, event_count, breakdown, computed_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, now())
        ON CONFLICT (subscription_id, meter_id, period_start, period_end)
        DO UPDATE SET value = $7, event_count = $8, breakdown = $9, computed_at = now()
      `, [
        subscription.tenant_id, subscription.customer_id, subscriptionId,
        meter.id, periodStart, periodEnd, result.value, result.event_count, result.breakdown,
      ]);
    }
  }

  private buildAggregateQuery(config: AggregationConfig): string {
    switch (config.type) {
      case "sum":
        return `SELECT COALESCE(SUM((properties->>'${config.field}')::numeric), 0) as value, COUNT(*) as event_count FROM usage_events WHERE tenant_id = $1 AND customer_id = $2 AND ce_type = $3 AND ce_time >= $4 AND ce_time < $5`;
      case "count":
        return `SELECT COUNT(*) as value, COUNT(*) as event_count FROM usage_events WHERE tenant_id = $1 AND customer_id = $2 AND ce_type = $3 AND ce_time >= $4 AND ce_time < $5`;
      case "max":
        return `SELECT COALESCE(MAX((properties->>'${config.field}')::numeric), 0) as value, COUNT(*) as event_count FROM usage_events WHERE tenant_id = $1 AND customer_id = $2 AND ce_type = $3 AND ce_time >= $4 AND ce_time < $5`;
      case "unique_count":
        return `SELECT COUNT(DISTINCT properties->>'${config.field}') as value, COUNT(*) as event_count FROM usage_events WHERE tenant_id = $1 AND customer_id = $2 AND ce_type = $3 AND ce_time >= $4 AND ce_time < $5`;
      case "latest":
        return `SELECT (properties->>'${config.field}')::numeric as value, 1 as event_count FROM usage_events WHERE tenant_id = $1 AND customer_id = $2 AND ce_type = $3 AND ce_time >= $4 AND ce_time < $5 ORDER BY ce_time DESC LIMIT 1`;
    }
  }
}
```

**Testing:**
- Sum aggregation: 100 events with `properties.tokens` values [10, 20, 30, ...] sums correctly
- Count aggregation: 500 events returns exactly 500
- Max aggregation: events with values [10, 50, 30, 70, 20] returns 70
- Unique count: events with `properties.user_id` values ["a","b","a","c","b"] returns 3
- Latest aggregation: returns the value from the most recent event by ce_time
- Re-aggregation (ON CONFLICT UPDATE) overwrites previous value
- Aggregation with zero events returns 0
- Aggregation respects period boundaries (events outside range excluded)
- group_by breakdown: events with `properties.region` groups and sums per region
- Dedup field: events with same dedup key counted only once

### Task 4.4: Real-Time Usage Query API

**What:** Build API endpoints for customers and operators to query current-period usage in real time.

**Design:**

```typescript
// GET /v1/subscriptions/:id/usage
// Returns current-period usage per meter with cost preview

export interface UsageQueryResponse {
  subscription_id: string;
  period_start: string;
  period_end: string;
  meters: {
    meter_id: string;
    meter_name: string;
    current_usage: number;
    included_usage: number;
    billable_usage: number;
    estimated_cost: string;
    currency: string;
  }[];
  estimated_total: string;
  currency: string;
}
```

**Testing:**
- `GET /v1/subscriptions/:id/usage` returns current period usage for all meters
- Response includes `estimated_cost` calculated using the plan's pricing_config
- Usage query returns zero for meters with no events in current period
- Usage query for subscription with included_units correctly shows billable = total - included
- Response updates within 5 seconds of new events being ingested

### Phase 4 -- Definition of Done

- [ ] Meters CRUD with support for sum, count, max, unique_count, and latest aggregation types
- [ ] Event ingestion accepts CloudEvents-formatted batches with ce_id deduplication
- [ ] Ingestion throughput handles 1000 events/second on a single node
- [ ] Usage aggregation worker correctly computes per-meter, per-subscription, per-period totals
- [ ] Real-time usage API returns current period usage with cost estimates
- [ ] Usage events table is partitioned by ce_time
- [ ] Test coverage for metering module exceeds 90%

---

## Phase 5: Invoicing & Tax Integration

**Goal:** Build the invoice generation engine that combines recurring charges, usage charges, proration adjustments, discounts, and tax calculations into finalized invoices with PDF generation.

**Duration estimate:** 3-4 weeks

### Task 5.1: Invoice Generation Engine

**What:** Build the core invoice generation service that creates draft invoices by combining recurring plan charges with aggregated usage, applying coupons, and calculating totals.

**Design:**

```sql
-- migrations/006_invoicing.sql

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','finalized','paid','partially_paid','past_due','void','uncollectible')),
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
    tax_breakdown   JSONB,
    e_invoice_config JSONB,
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
    charge_key      TEXT,
    description     TEXT NOT NULL,
    line_type       TEXT NOT NULL CHECK (line_type IN ('charge','proration','credit','tax','discount','minimum_adjustment')),
    quantity        NUMERIC(20,8) NOT NULL DEFAULT 1,
    unit_amount     NUMERIC(20,8) NOT NULL,
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    usage_detail    JSONB,
    sort_order      INT NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);
```

```typescript
// packages/engine/src/invoicing/generator.ts

export class InvoiceGenerator {
  constructor(
    private pricingCalculator: PricingCalculator,
    private discountCalculator: DiscountCalculator,
    private usageAggregateRepo: UsageAggregateRepository,
    private invoiceRepo: InvoiceRepository,
    private invoiceNumberGenerator: InvoiceNumberGenerator,
  ) {}

  async generateForSubscription(subscriptionId: string, periodStart: Date, periodEnd: Date): Promise<Invoice> {
    const subscription = await this.subscriptionRepo.findById(subscriptionId);
    const plan = await this.planRepo.findById(subscription.plan_id);
    const customer = await this.customerRepo.findById(subscription.customer_id);

    // 1. Calculate recurring charges
    const recurringInputs = this.buildRecurringInputs(plan.pricing_config, subscription.overrides);

    // 2. Fetch usage aggregates and build usage inputs
    const usageInputs = await this.buildUsageInputs(subscription, plan, periodStart, periodEnd);

    // 3. Run pricing calculator
    const allInputs = [...recurringInputs, ...usageInputs];
    const lineItems = this.pricingCalculator.calculate(plan.pricing_config, allInputs, subscription.currency);

    // 4. Apply coupons
    const appliedCoupons = await this.appliedCouponRepo.findActiveForSubscription(subscriptionId);
    let discountTotal = Money.fromDecimal("0", subscription.currency);
    for (const ac of appliedCoupons) {
      const coupon = await this.couponRepo.findById(ac.coupon_id);
      const discount = this.discountCalculator.apply(lineItems, coupon.discount_config);
      discountTotal = discountTotal.add(discount);
    }

    // 5. Calculate subtotal
    const subtotal = lineItems.reduce(
      (sum, li) => sum.add(li.amount),
      Money.fromDecimal("0", subscription.currency),
    );

    // 6. Create draft invoice
    const invoiceNumber = await this.invoiceNumberGenerator.next(subscription.tenant_id);
    const invoice = await this.invoiceRepo.create({
      tenant_id: subscription.tenant_id,
      customer_id: subscription.customer_id,
      subscription_id: subscriptionId,
      invoice_number: invoiceNumber,
      status: "draft",
      currency: subscription.currency,
      subtotal: subtotal.toDecimal(),
      discount_amount: discountTotal.toDecimal(),
      total: subtotal.subtract(discountTotal).toDecimal(),
      amount_due: subtotal.subtract(discountTotal).toDecimal(),
      period_start: periodStart,
      period_end: periodEnd,
      due_at: this.calculateDueDate(periodStart, subscription),
    });

    // 7. Create line items
    for (const li of lineItems) {
      await this.invoiceLineItemRepo.create({
        invoice_id: invoice.id,
        charge_key: li.chargeKey,
        description: li.chargeName,
        line_type: "charge",
        quantity: li.quantity,
        unit_amount: li.unitAmount,
        amount: li.amount.toDecimal(),
        currency: subscription.currency,
        period_start: periodStart,
        period_end: periodEnd,
        usage_detail: li.tierBreakdown ? { tiers: li.tierBreakdown } : null,
      });
    }

    return invoice;
  }
}
```

**Testing:**
- Generate invoice for subscription with flat $99/mo: subtotal = $99.00, 1 line item
- Generate invoice for subscription with flat + usage: line items include both recurring and usage charges
- Generate invoice with 20% coupon: discount_amount = correct, total = subtotal - discount
- Invoice number generated in sequence: INV-2026-000001, INV-2026-000002, ...
- Invoice with net_terms_days = 30: due_at is 30 days after period_start
- Invoice with zero usage charges: only recurring line items present
- Invoice line items include usage_detail JSONB with tier breakdown for tiered usage
- Generated invoice starts in `draft` status

### Task 5.2: Invoice Finalization & PDF Generation

**What:** Implement the finalization workflow (draft -> finalized), tax calculation integration point, and PDF invoice rendering.

**Design:**

```typescript
// packages/engine/src/invoicing/finalizer.ts

export class InvoiceFinalizer {
  async finalize(invoiceId: string): Promise<Invoice> {
    const invoice = await this.invoiceRepo.findById(invoiceId);

    if (invoice.status !== "draft") {
      throw new InvalidStateTransitionError("invoice", invoice.status, "finalized");
    }

    // Calculate tax (external tax engine integration point)
    const taxResult = await this.taxCalculator.calculate(invoice);

    // Update invoice with tax
    const finalizedInvoice = await this.invoiceRepo.update(invoiceId, {
      status: "finalized",
      tax_amount: taxResult.totalTax.toDecimal(),
      tax_breakdown: taxResult.breakdown,
      total: (Money.fromDecimal(invoice.subtotal, invoice.currency)
        .subtract(Money.fromDecimal(invoice.discount_amount, invoice.currency))
        .add(taxResult.totalTax)).toDecimal(),
      amount_due: (Money.fromDecimal(invoice.subtotal, invoice.currency)
        .subtract(Money.fromDecimal(invoice.discount_amount, invoice.currency))
        .add(taxResult.totalTax)).toDecimal(),
      issued_at: new Date(),
    });

    // Generate PDF
    const pdfUrl = await this.pdfGenerator.generate(finalizedInvoice);
    await this.invoiceRepo.update(invoiceId, { pdf_url: pdfUrl });

    // Queue payment collection if auto_collection enabled
    if (invoice.auto_collection) {
      await this.paymentQueue.add("collect-payment", { invoiceId });
    }

    return finalizedInvoice;
  }
}
```

**Testing:**
- Finalizing a draft invoice transitions to `finalized` and sets `issued_at`
- Finalizing an already-finalized invoice throws InvalidStateTransitionError
- Tax calculation adds tax_amount and updates total = subtotal - discount + tax
- Tax breakdown JSONB includes jurisdiction, rate, and amount per tax line
- PDF is generated and uploaded to object storage; pdf_url is set
- Auto-collection queues a payment collection job
- Voiding a finalized invoice transitions to `void` and sets `voided_at`

### Task 5.3: Credit Notes

**What:** Implement credit note generation for refunds, order changes, and billing corrections.

**Design:**

```sql
-- migrations/007_credit_notes.sql

CREATE TABLE credit_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    credit_note_number TEXT NOT NULL,
    reason          TEXT NOT NULL CHECK (reason IN ('duplicate','fraudulent','order_change','product_unsatisfactory','other')),
    status          TEXT NOT NULL DEFAULT 'issued' CHECK (status IN ('issued', 'voided')),
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

**Testing:**
- Credit note for full invoice amount: total matches original invoice
- Partial credit note for one line item only
- Credit note with `refund_amount > 0` triggers refund through payment gateway
- Credit note with `credit_amount > 0` adds to customer's credit balance
- Credit note cannot exceed original invoice amount
- Voided credit note cannot be re-issued

### Phase 5 -- Definition of Done

- [ ] Invoice generation combines recurring charges, usage charges, and discounts correctly
- [ ] Invoice numbers are sequential and unique within a tenant
- [ ] Invoice finalization calculates tax and transitions to `finalized`
- [ ] PDF invoices are generated and stored in object storage
- [ ] Credit notes support full and partial refunds/credits
- [ ] Invoice state machine: draft -> finalized -> paid/void transitions enforced
- [ ] Integration point for external tax engines is defined and testable with a stub
- [ ] Test coverage for invoicing module exceeds 90%

---

## Phase 6: Payment Processing & Gateway Integration

**Goal:** Build the payment gateway abstraction layer, implement Stripe adapter, process payments against invoices, and handle payment lifecycle events.

**Duration estimate:** 2-3 weeks

### Task 6.1: Payment Gateway Abstraction

**What:** Define the gateway interface contract and implement the Stripe adapter. Store payment methods as gateway tokens (PCI DSS compliant -- no raw card data).

**Design:**

```typescript
// packages/gateway/src/interface.ts

export interface PaymentGateway {
  readonly name: string;

  /** Create a payment method token from a checkout session or setup intent */
  createPaymentMethod(customerId: string, params: CreatePaymentMethodParams): Promise<PaymentMethodResult>;

  /** Charge a payment method */
  charge(params: ChargeParams): Promise<ChargeResult>;

  /** Refund a previous charge */
  refund(params: RefundParams): Promise<RefundResult>;

  /** Verify a webhook signature from this gateway */
  verifyWebhookSignature(payload: string, signature: string): boolean;
}

export interface ChargeParams {
  amount: string;          // decimal string
  currency: string;        // ISO 4217
  paymentMethodToken: string;
  idempotencyKey: string;
  description?: string;
  metadata?: Record<string, string>;
}

export interface ChargeResult {
  gatewayPaymentId: string;
  status: "succeeded" | "failed" | "pending" | "requires_action";
  failureCode?: string;
  failureMessage?: string;
  receiptUrl?: string;
}

// packages/gateway/src/stripe/adapter.ts
import Stripe from "stripe";

export class StripeGateway implements PaymentGateway {
  readonly name = "stripe";
  private client: Stripe;

  constructor(apiKey: string) {
    this.client = new Stripe(apiKey, { apiVersion: "2026-04-22" });
  }

  async charge(params: ChargeParams): Promise<ChargeResult> {
    const paymentIntent = await this.client.paymentIntents.create({
      amount: this.toMinorUnits(params.amount, params.currency),
      currency: params.currency.toLowerCase(),
      payment_method: params.paymentMethodToken,
      confirm: true,
      off_session: true,     // Merchant-initiated transaction (PSD2 MIT exemption)
      description: params.description,
      metadata: params.metadata,
    }, {
      idempotencyKey: params.idempotencyKey,
    });

    return {
      gatewayPaymentId: paymentIntent.id,
      status: paymentIntent.status === "succeeded" ? "succeeded" : "failed",
      failureCode: paymentIntent.last_payment_error?.code,
      failureMessage: paymentIntent.last_payment_error?.message,
    };
  }
}
```

**Testing:**
- StripeGateway.charge with valid params returns `{ status: "succeeded", gatewayPaymentId: "pi_..." }`
- StripeGateway.charge with declined card returns `{ status: "failed", failureCode: "card_declined" }`
- Idempotency: two charges with same idempotencyKey return same gatewayPaymentId
- StripeGateway.refund with valid paymentId returns successful refund
- Webhook signature verification accepts valid signatures, rejects tampered payloads
- Gateway interface is satisfied by mock adapter for testing without Stripe credentials
- Currency conversion to minor units: $99.99 USD -> 9999, 100 JPY -> 100

### Task 6.2: Payment Processing Service

**What:** Build the service that processes payments against finalized invoices, updates invoice status, and records payment transactions.

**Design:**

```sql
-- migrations/008_payments.sql

CREATE TABLE payment_methods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    method_type     TEXT NOT NULL CHECK (method_type IN ('card','bank_account','sepa_debit','ach','paypal','wire')),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','expired','failed','revoked')),
    gateway_config  JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method_id UUID REFERENCES payment_methods(id),
    amount          NUMERIC(20,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','processing','succeeded','failed','refunded','partially_refunded')),
    gateway_response JSONB NOT NULL DEFAULT '{}',
    refunded_amount NUMERIC(20,2) NOT NULL DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_customer ON payments(customer_id);
```

```typescript
// packages/engine/src/payment/processor.ts

export class PaymentProcessor {
  async collectPayment(invoiceId: string): Promise<Payment> {
    const invoice = await this.invoiceRepo.findById(invoiceId);
    if (invoice.status !== "finalized" && invoice.status !== "past_due") {
      throw new BillingError("invalid-invoice-status", "Cannot Collect Payment", 422,
        `Invoice is in '${invoice.status}' state; must be 'finalized' or 'past_due'`);
    }

    const paymentMethod = await this.paymentMethodRepo.findDefaultForCustomer(invoice.customer_id);
    if (!paymentMethod) {
      throw new BillingError("no-payment-method", "No Payment Method", 422,
        "Customer has no default payment method configured");
    }

    const gateway = this.gatewayRegistry.get(paymentMethod.gateway_config.gateway);
    const chargeResult = await gateway.charge({
      amount: invoice.amount_due,
      currency: invoice.currency,
      paymentMethodToken: paymentMethod.gateway_config.token,
      idempotencyKey: `invoice_${invoiceId}_attempt_${Date.now()}`,
    });

    const payment = await this.paymentRepo.create({
      tenant_id: invoice.tenant_id,
      invoice_id: invoiceId,
      customer_id: invoice.customer_id,
      payment_method_id: paymentMethod.id,
      amount: invoice.amount_due,
      currency: invoice.currency,
      status: chargeResult.status,
      gateway_response: chargeResult,
      paid_at: chargeResult.status === "succeeded" ? new Date() : null,
    });

    if (chargeResult.status === "succeeded") {
      await this.invoiceRepo.update(invoiceId, {
        status: "paid",
        amount_paid: invoice.amount_due,
        amount_due: "0",
        paid_at: new Date(),
      });
    } else {
      await this.invoiceRepo.update(invoiceId, { status: "past_due" });
    }

    return payment;
  }
}
```

**Testing:**
- Successful payment transitions invoice from `finalized` to `paid`
- Failed payment transitions invoice to `past_due`
- Payment record stores gateway_response JSONB with chargeResult
- No-default-payment-method throws descriptive error
- Partial payment: amount_paid < total leaves invoice in `partially_paid`
- Payment amount matches invoice amount_due exactly
- Idempotency key prevents duplicate charges for same invoice

### Task 6.3: Refund Processing

**What:** Process refunds through the gateway and link them to credit notes.

**Design:**

```typescript
// packages/engine/src/payment/refund.ts

export class RefundService {
  async processRefund(paymentId: string, amount: string, reason: string): Promise<Refund> {
    const payment = await this.paymentRepo.findById(paymentId);
    const remainingRefundable = Number(payment.amount) - Number(payment.refunded_amount);

    if (Number(amount) > remainingRefundable) {
      throw new BillingError("refund-exceeds-payment", "Refund Exceeds Payment", 422,
        `Requested refund ${amount} exceeds refundable amount ${remainingRefundable}`);
    }

    const gateway = this.gatewayRegistry.get(payment.gateway_response.gateway);
    const refundResult = await gateway.refund({
      gatewayPaymentId: payment.gateway_response.gatewayPaymentId,
      amount,
      currency: payment.currency,
    });

    // Update payment refunded_amount
    const newRefundedAmount = (Number(payment.refunded_amount) + Number(amount)).toFixed(2);
    const newStatus = newRefundedAmount === payment.amount ? "refunded" : "partially_refunded";

    await this.paymentRepo.update(paymentId, {
      refunded_amount: newRefundedAmount,
      status: newStatus,
    });

    return refundResult;
  }
}
```

**Testing:**
- Full refund transitions payment to `refunded`
- Partial refund transitions payment to `partially_refunded`
- Refund exceeding payment amount is rejected with 422
- Second partial refund adjusting total to full amount transitions to `refunded`
- Refund on a `failed` payment is rejected
- Gateway refund failure does not update payment status

### Phase 6 -- Definition of Done

- [ ] Payment gateway abstraction supports Stripe with clean interface for adding adapters
- [ ] Payment methods stored with tokenized references only (PCI DSS compliant)
- [ ] Successful payment transitions invoice to `paid`
- [ ] Failed payment transitions invoice to `past_due`
- [ ] Refund processing supports full and partial refunds
- [ ] Idempotency keys prevent duplicate charges
- [ ] Mock gateway adapter enables full test suite without external credentials
- [ ] Test coverage for payment module exceeds 90%

---

## Phase 7: Dunning & Payment Recovery

**Goal:** Implement configurable dunning campaigns with multi-step retry schedules, notification templates, and payment method fallback. Prepare the data structures for AI dunning optimization in Phase 11.

**Duration estimate:** 2-3 weeks

### Task 7.1: Dunning Campaign Configuration

**What:** Create dunning campaigns with configurable step sequences (retry, email, SMS, pause, cancel) and delay intervals.

**Design:**

```sql
-- migrations/009_dunning.sql

CREATE TABLE dunning_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    rules           JSONB NOT NULL,
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
    result          TEXT NOT NULL CHECK (result IN ('success','failed','skipped','pending')),
    payment_id      UUID REFERENCES payments(id),
    ai_context      JSONB,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    executed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dunning_invoice ON dunning_attempts(invoice_id);
CREATE INDEX idx_dunning_scheduled ON dunning_attempts(scheduled_at) WHERE result = 'pending';
```

**Testing:**
- Create campaign with 5 steps: retry(day 1), retry(day 3), email(day 5), retry(day 10), cancel(day 30)
- Campaign validation rejects steps without delay_days
- Campaign validation rejects campaigns without at least one retry_payment step
- `GET /v1/dunning-campaigns` returns campaigns for tenant

### Task 7.2: Dunning Scheduler & Executor

**What:** Build the worker that initiates dunning sequences when invoices go past_due and executes scheduled dunning steps.

**Design:**

```typescript
// packages/workers/src/dunning-scheduler.ts

export class DunningScheduler {
  /** Called when an invoice transitions to past_due */
  async initiateDunning(invoiceId: string): Promise<void> {
    const invoice = await this.invoiceRepo.findById(invoiceId);
    const subscription = await this.subscriptionRepo.findById(invoice.subscription_id);

    // Find applicable campaign (from tenant config or subscription override)
    const campaign = await this.findCampaign(subscription);

    // Schedule first step
    const firstStep = campaign.rules.steps[0];
    const scheduledAt = new Date(Date.now() + firstStep.delay_days * 86400000);

    await this.dunningAttemptRepo.create({
      tenant_id: invoice.tenant_id,
      invoice_id: invoiceId,
      subscription_id: subscription.id,
      campaign_id: campaign.id,
      step_number: 1,
      attempt_number: 1,
      action_taken: firstStep.action,
      result: "pending",
      scheduled_at: scheduledAt,
    });
  }

  /** Worker poll: execute due dunning steps */
  async executeDueSteps(): Promise<void> {
    const dueAttempts = await this.dunningAttemptRepo.findDue(new Date());

    for (const attempt of dueAttempts) {
      await this.executeStep(attempt);
    }
  }

  private async executeStep(attempt: DunningAttempt): Promise<void> {
    switch (attempt.action_taken) {
      case "retry_payment":
        const payment = await this.paymentProcessor.collectPayment(attempt.invoice_id);
        if (payment.status === "succeeded") {
          await this.dunningAttemptRepo.update(attempt.id, { result: "success", payment_id: payment.id, executed_at: new Date() });
          // Invoice is now paid; dunning complete
          return;
        }
        await this.dunningAttemptRepo.update(attempt.id, { result: "failed", payment_id: payment.id, executed_at: new Date() });
        break;

      case "send_email":
        await this.emailService.sendDunningEmail(attempt);
        await this.dunningAttemptRepo.update(attempt.id, { result: "success", executed_at: new Date() });
        break;

      case "cancel_subscription":
        await this.subscriptionService.cancel(attempt.subscription_id);
        await this.dunningAttemptRepo.update(attempt.id, { result: "success", executed_at: new Date() });
        return; // No further steps after cancellation
    }

    // Schedule next step
    await this.scheduleNextStep(attempt);
  }
}
```

**Testing:**
- Invoice going past_due initiates dunning with first step scheduled at correct time
- Retry step: successful payment marks attempt as `success` and stops dunning
- Retry step: failed payment marks attempt as `failed` and schedules next step
- Email step: sends notification and schedules next step
- Cancel step: cancels subscription and ends dunning sequence
- Dunning with fallback payment method: tries default, then alternate
- No further steps scheduled after the last campaign step
- Concurrent dunning execution: same invoice not processed twice (locking)
- ai_context JSONB column stores empty object for non-AI attempts (prepares for Phase 11)

### Phase 7 -- Definition of Done

- [ ] Dunning campaigns configurable with multi-step sequences
- [ ] Dunning initiation triggered automatically when invoice goes past_due
- [ ] Step execution handles retry_payment, send_email, and cancel_subscription actions
- [ ] Successful payment during dunning stops the sequence
- [ ] Dunning attempts table stores ai_context for future AI integration
- [ ] Concurrent dunning execution is safe (no duplicate processing)
- [ ] Test coverage for dunning module exceeds 90%

---

## Phase 8: Revenue Recognition (ASC 606 / IFRS 15)

**Goal:** Implement ASC 606 five-step revenue recognition: contract identification, performance obligation identification, transaction price determination, price allocation, and revenue recognition scheduling.

**Duration estimate:** 4-5 weeks

### Task 8.1: Revenue Contract & Performance Obligation Management

**What:** Create revenue contracts linked to subscriptions, decompose them into performance obligations, and determine standalone selling prices.

**Design:**

```sql
-- migrations/010_revenue_recognition.sql

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
    asc606_analysis JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

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
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','posted','reversed')),
    lines           JSONB NOT NULL,
    posted_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entry_number)
);
```

```typescript
// packages/engine/src/revenue/contract-analyzer.ts

export class ContractAnalyzer {
  /**
   * ASC 606 Step 1 & 2: Identify the contract and performance obligations.
   * Decomposes a subscription into performance obligations based on charge types.
   */
  analyzeContract(subscription: Subscription, plan: Plan): ASC606Analysis {
    const obligations: PerformanceObligation[] = [];

    for (const charge of plan.pricing_config) {
      obligations.push({
        id: `po_${charge.key}`,
        description: charge.name,
        type: this.classifyObligationType(charge),
        standalone_selling_price: this.determineSSP(charge),
        charge_key: charge.key,
      });
    }

    // Step 3: Determine transaction price
    const transactionPrice = this.calculateTransactionPrice(plan.pricing_config, subscription);

    // Step 4: Allocate transaction price using relative SSP method
    const totalSSP = obligations.reduce((sum, o) => sum + o.standalone_selling_price, 0);
    for (const obligation of obligations) {
      obligation.allocated_price = (obligation.standalone_selling_price / totalSSP) * transactionPrice;
    }

    return {
      step1_contract_identified: true,
      step2_obligations: obligations,
      step3_transaction_price: transactionPrice,
      step4_allocation_method: "relative_ssp",
    };
  }

  private classifyObligationType(charge: ChargeConfig): "over_time" | "point_in_time" {
    // Recurring and usage charges are satisfied over time
    if (charge.charge_type === "recurring" || charge.charge_type === "usage") {
      return "over_time";
    }
    // Setup and one-time charges are point-in-time
    return "point_in_time";
  }
}
```

**Testing:**
- Contract with flat recurring charge creates 1 "over_time" performance obligation
- Contract with flat + setup charges creates 2 obligations: "over_time" and "point_in_time"
- SSP allocation: $1200 annual plan + $500 setup = $1700 total; SSP split applies relative method
- Point-in-time obligation is fully recognized on delivery date
- Over-time obligation generates monthly recognition schedule
- Variable consideration (usage charges) estimated using expected value method
- Contract modification (plan change) creates a new contract version

### Task 8.2: Revenue Schedule Generation & Recognition Worker

**What:** Generate time-based revenue schedules for each performance obligation and build the worker that recognizes revenue per period and generates journal entries.

**Design:**

```typescript
// packages/engine/src/revenue/schedule-generator.ts

export class RevenueScheduleGenerator {
  generateSchedule(contract: RevenueContract, obligation: PerformanceObligation): RevenueScheduleEntry[] {
    if (obligation.type === "point_in_time") {
      return [{
        obligation_key: obligation.id,
        period_start: contract.inception_date,
        period_end: contract.inception_date,
        amount: obligation.allocated_price,
        currency: contract.currency,
      }];
    }

    // Over-time: straight-line recognition across contract term
    const months = this.monthsBetween(contract.inception_date, contract.expiration_date);
    const monthlyAmount = Number((obligation.allocated_price / months).toFixed(2));
    const entries: RevenueScheduleEntry[] = [];
    let remaining = obligation.allocated_price;

    for (let i = 0; i < months; i++) {
      const periodStart = this.addMonths(contract.inception_date, i);
      const periodEnd = this.addMonths(contract.inception_date, i + 1);
      const amount = i === months - 1 ? remaining : monthlyAmount; // Last month gets remainder
      entries.push({ obligation_key: obligation.id, period_start: periodStart, period_end: periodEnd, amount, currency: contract.currency });
      remaining -= monthlyAmount;
    }

    return entries;
  }
}

// packages/workers/src/revenue-recognizer.ts

export class RevenueRecognitionWorker {
  async recognizeForPeriod(tenantId: string, periodEnd: Date): Promise<void> {
    const schedules = await this.scheduleRepo.findUnrecognizedDueBefore(tenantId, periodEnd);

    for (const schedule of schedules) {
      const journalEntry = await this.createJournalEntry(schedule);

      await this.scheduleRepo.update(schedule.id, {
        recognized: true,
        recognized_at: new Date(),
        journal_entry_ref: journalEntry.entry_number,
      });
    }
  }

  private async createJournalEntry(schedule: RevenueSchedule): Promise<JournalEntry> {
    return this.journalEntryRepo.create({
      tenant_id: schedule.tenant_id,
      entry_number: await this.entryNumberGenerator.next(schedule.tenant_id),
      entry_date: schedule.period_end,
      description: `Revenue recognition: ${schedule.obligation_key} for ${schedule.period_start} to ${schedule.period_end}`,
      source_type: "revenue_recognition",
      source_id: schedule.id,
      status: "pending",
      lines: [
        { account_code: "2500", account_name: "Deferred Revenue", debit: schedule.amount, credit: 0 },
        { account_code: "4000", account_name: "Revenue", debit: 0, credit: schedule.amount },
      ],
    });
  }
}
```

**Testing:**
- $1200 annual contract: 12 monthly schedules of $100.00 each
- $100 monthly contract: 1 schedule entry of $100.00 per month
- Remainder handling: $1000 / 12 months = 11 x $83.33 + 1 x $83.37
- Point-in-time obligation: single schedule entry on inception date
- Recognition worker: marks schedule as recognized and generates journal entry
- Journal entry lines balance (total debits == total credits)
- Unrecognized schedules query returns only past-due unrecognized entries
- Revenue report: sum of recognized amounts per period per tenant

### Phase 8 -- Definition of Done

- [ ] Revenue contracts created automatically from subscription creation/amendment
- [ ] ASC 606 five-step analysis stored in asc606_analysis JSONB
- [ ] Performance obligations classified as over_time or point_in_time
- [ ] Relative SSP allocation distributes transaction price correctly
- [ ] Revenue schedules generated with correct monthly amounts and remainder handling
- [ ] Recognition worker creates balanced journal entries
- [ ] Revenue recognition reports show deferred vs. recognized amounts per period
- [ ] Test coverage for revenue module exceeds 90%

---

## Phase 9: Credit Ledger & Commit-Based Contracts

**Goal:** Implement prepaid credit wallets, commit-based contracts with drawdowns, credit application to invoices, and credit expiration.

**Duration estimate:** 2-3 weeks

### Task 9.1: Credit Wallet Management

**What:** Build credit wallets that support prepaid commits, auto-topup, expiration, and rollover policies.

**Design:**

```sql
-- migrations/011_credits.sql

CREATE TABLE credit_wallets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(20,8) NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active',
    wallet_config   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallets_customer ON credit_wallets(customer_id);

CREATE TABLE credit_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id       UUID NOT NULL REFERENCES credit_wallets(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('credit','debit','expiration','void','topup')),
    amount          NUMERIC(20,8) NOT NULL,
    balance_after   NUMERIC(20,8) NOT NULL,
    description     TEXT,
    reference_type  TEXT,
    reference_id    UUID,
    idempotency_key TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credit_txns_wallet ON credit_transactions(wallet_id, created_at);
```

```typescript
// packages/engine/src/credits/wallet-service.ts

export class CreditWalletService {
  async credit(walletId: string, amount: string, description: string, idempotencyKey: string): Promise<CreditTransaction> {
    return this.db.withTransaction(async (client) => {
      const wallet = await this.walletRepo.findByIdForUpdate(client, walletId); // SELECT FOR UPDATE
      const amountDecimal = Number(amount);
      const newBalance = Number(wallet.balance) + amountDecimal;

      await this.walletRepo.updateBalance(client, walletId, newBalance.toString());

      return this.transactionRepo.create(client, {
        wallet_id: walletId,
        transaction_type: "credit",
        amount: amountDecimal,
        balance_after: newBalance,
        description,
        idempotency_key: idempotencyKey,
      });
    });
  }

  async debit(walletId: string, amount: string, invoiceId: string, idempotencyKey: string): Promise<CreditTransaction> {
    return this.db.withTransaction(async (client) => {
      const wallet = await this.walletRepo.findByIdForUpdate(client, walletId);
      const amountDecimal = Number(amount);

      if (Number(wallet.balance) < amountDecimal) {
        throw new BillingError("insufficient-credits", "Insufficient Credits", 422,
          `Wallet balance ${wallet.balance} is less than requested debit ${amount}`);
      }

      const newBalance = Number(wallet.balance) - amountDecimal;
      await this.walletRepo.updateBalance(client, walletId, newBalance.toString());

      return this.transactionRepo.create(client, {
        wallet_id: walletId,
        transaction_type: "debit",
        amount: -amountDecimal,
        balance_after: newBalance,
        description: `Applied to invoice`,
        reference_type: "invoice",
        reference_id: invoiceId,
        idempotency_key: idempotencyKey,
      });
    });
  }
}
```

**Testing:**
- Credit $1000 to wallet: balance increases to $1000, transaction recorded
- Debit $300 from wallet: balance decreases to $700, transaction links to invoice
- Debit exceeding balance throws InsufficientCreditsError
- Idempotency: duplicate credit with same key does not double-credit
- Concurrent debits: two $600 debits on $1000 balance -- exactly one succeeds (SELECT FOR UPDATE)
- Transaction log sum matches current wallet balance
- Credit expiration: expired credits deducted and expiration transaction recorded

### Task 9.2: Credit Application to Invoices

**What:** During invoice finalization, check for available credits and apply them before charging the payment method.

**Design:**

```typescript
// packages/engine/src/credits/invoice-credit-applicator.ts

export class InvoiceCreditApplicator {
  async applyCredits(invoice: Invoice): Promise<{ creditApplied: Money; remainingDue: Money }> {
    const wallet = await this.walletRepo.findActiveForCustomer(invoice.customer_id, invoice.currency);

    if (!wallet || Number(wallet.balance) <= 0) {
      return {
        creditApplied: Money.fromDecimal("0", invoice.currency),
        remainingDue: Money.fromDecimal(invoice.amount_due, invoice.currency),
      };
    }

    const availableCredit = Number(wallet.balance);
    const invoiceDue = Number(invoice.amount_due);
    const creditToApply = Math.min(availableCredit, invoiceDue);

    await this.creditWalletService.debit(
      wallet.id,
      creditToApply.toFixed(2),
      invoice.id,
      `credit_apply_${invoice.id}`,
    );

    await this.invoiceRepo.update(invoice.id, {
      credit_amount: creditToApply.toFixed(2),
      amount_due: (invoiceDue - creditToApply).toFixed(2),
    });

    return {
      creditApplied: Money.fromDecimal(creditToApply.toFixed(2), invoice.currency),
      remainingDue: Money.fromDecimal((invoiceDue - creditToApply).toFixed(2), invoice.currency),
    };
  }
}
```

**Testing:**
- Invoice $100, wallet $150: credit_amount = $100, amount_due = $0, no payment needed
- Invoice $100, wallet $40: credit_amount = $40, amount_due = $60, payment for $60
- Invoice $100, no wallet: credit_amount = $0, amount_due = $100
- Credit application creates debit transaction in wallet
- Invoice credit_amount field correctly reflects applied amount

### Phase 9 -- Definition of Done

- [ ] Credit wallets support credit, debit, expiration, and void operations
- [ ] Wallet operations are transactionally safe with SELECT FOR UPDATE locking
- [ ] Credit application to invoices reduces amount_due before payment processing
- [ ] Transaction log provides complete audit trail with running balance
- [ ] Idempotency keys prevent duplicate credit/debit operations
- [ ] Wallet configuration supports commit amount, expiration, and rollover policies
- [ ] Test coverage for credits module exceeds 90%

---

## Phase 10: Webhooks, Audit & API Finalization

**Goal:** Build the outbound webhook delivery system with HMAC signing and retry, the append-only audit log, and finalize the REST API with comprehensive OpenAPI 3.1 documentation.

**Duration estimate:** 2-3 weeks (runs partially in parallel with Phases 2-9)

### Task 10.1: Webhook Delivery System

**What:** Build the webhook endpoint configuration, event dispatch, HMAC-SHA256 signature generation, and exponential backoff retry.

**Design:**

```sql
-- migrations/012_webhooks.sql

CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
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
```

```typescript
// packages/webhooks/src/signer.ts
import crypto from "node:crypto";

export function signWebhookPayload(payload: string, secret: string, timestamp: number): string {
  const signedContent = `${timestamp}.${payload}`;
  const signature = crypto
    .createHmac("sha256", secret)
    .update(signedContent)
    .digest("hex");
  return `t=${timestamp},v1=${signature}`;
}

// packages/webhooks/src/dispatcher.ts
export class WebhookDispatcher {
  async dispatch(tenantId: string, eventType: string, payload: object): Promise<void> {
    const endpoints = await this.endpointRepo.findByTenantAndEvent(tenantId, eventType);

    for (const endpoint of endpoints) {
      const deliveryId = await this.deliveryRepo.create({
        endpoint_id: endpoint.id,
        event_type: eventType,
        payload,
        status: "pending",
      });

      await this.deliveryQueue.add("deliver-webhook", { deliveryId }, {
        attempts: 5,
        backoff: { type: "exponential", delay: 60000 }, // 1min, 2min, 4min, 8min, 16min
      });
    }
  }
}
```

**Testing:**
- Webhook signature matches expected HMAC-SHA256 output
- Tampered payload fails signature verification
- Endpoint subscribing to `invoice.finalized` receives webhook on invoice finalization
- Endpoint subscribing to `payment.succeeded` does NOT receive `invoice.finalized` events
- Failed delivery (HTTP 500) retries with exponential backoff
- After 5 failed attempts, delivery status transitions to `failed`
- Successful delivery (HTTP 200) sets `delivered_at` and status to `delivered`
- Webhook payload includes `event_type`, `timestamp`, and full resource object

### Task 10.2: Audit Log

**What:** Implement the append-only, partitioned audit log that records all mutations with before/after snapshots.

**Design:**

```sql
-- migrations/013_audit.sql

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

-- Create partitions for current and next month
-- (automated partition creation handled by pg_partman or application logic)

CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(tenant_id, actor_id, created_at);
```

```typescript
// packages/engine/src/audit/logger.ts
export class AuditLogger {
  async log(entry: AuditEntry): Promise<void> {
    await this.db.query(`
      INSERT INTO audit_log (tenant_id, actor_type, actor_id, action, resource_type, resource_id, changes, request_context)
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
    `, [
      entry.tenantId, entry.actorType, entry.actorId, entry.action,
      entry.resourceType, entry.resourceId, entry.changes, entry.requestContext,
    ]);
  }
}
```

**Testing:**
- Creating a subscription generates audit entry with `action: "subscription.created"`
- Updating a customer generates audit entry with `changes: { before: {...}, after: {...} }`
- Audit log query by `resource_type` and `resource_id` returns chronological history
- Audit log query by `actor_id` returns all actions by that actor
- Audit entries are immutable (no UPDATE or DELETE operations supported)
- Partition pruning: queries for recent entries do not scan old partitions

### Task 10.3: OpenAPI 3.1 Specification & API Documentation

**What:** Finalize the OpenAPI 3.1 specification covering all endpoints, generate SDK-ready documentation, and ensure all routes have request/response schemas.

**Testing:**
- OpenAPI spec validates against OpenAPI 3.1 schema
- All API routes have corresponding OpenAPI path definitions
- All request bodies have JSON Schema validation
- All response bodies have documented schemas
- Generated TypeScript types from OpenAPI match actual API responses
- `GET /openapi.json` returns the full spec

### Phase 10 -- Definition of Done

- [ ] Webhook endpoints configurable per tenant with event filtering
- [ ] Webhook payloads signed with HMAC-SHA256 and timestamp
- [ ] Failed deliveries retry with exponential backoff (5 attempts)
- [ ] Audit log records all mutations with before/after snapshots
- [ ] Audit log is partitioned by month for efficient retention management
- [ ] OpenAPI 3.1 specification is complete and validates
- [ ] API documentation served via Swagger UI
- [ ] Test coverage for webhooks and audit modules exceeds 90%

---

## Phase 11: AI-Native Features

**Goal:** Implement the AI differentiators: intelligent dunning optimization, ASC 606 contract classification, churn prediction from billing signals, and pricing recommendation.

**Duration estimate:** 5-6 weeks

### Task 11.1: AI Dunning Optimization

**What:** Build a machine learning model that predicts optimal retry timing per customer based on payment history, card type, time of day, and day of week. Integrate with the dunning scheduler to override static retry schedules when confidence exceeds threshold.

**Design:**

```typescript
// packages/ai/src/dunning/feature-extractor.ts

export interface DunningFeatures {
  customer_lifetime_days: number;
  payment_method_type: string;
  card_brand: string | null;
  amount: number;
  currency: string;
  previous_failure_count: number;
  days_past_due: number;
  day_of_week: number;        // 0-6
  hour_of_day: number;        // 0-23
  historical_success_rate: number;  // 0-1
  avg_days_to_pay: number;
}

export class DunningFeatureExtractor {
  async extract(invoiceId: string): Promise<DunningFeatures> {
    const invoice = await this.invoiceRepo.findById(invoiceId);
    const customer = await this.customerRepo.findById(invoice.customer_id);
    const paymentHistory = await this.paymentRepo.findByCustomer(customer.id, { limit: 50 });

    const successCount = paymentHistory.filter(p => p.status === "succeeded").length;
    const totalAttempts = paymentHistory.length;

    return {
      customer_lifetime_days: this.daysSince(customer.created_at),
      payment_method_type: await this.getDefaultPaymentMethodType(customer.id),
      card_brand: await this.getDefaultCardBrand(customer.id),
      amount: Number(invoice.amount_due),
      currency: invoice.currency,
      previous_failure_count: paymentHistory.filter(p => p.status === "failed").length,
      days_past_due: this.daysSince(invoice.due_at),
      day_of_week: new Date().getDay(),
      hour_of_day: new Date().getHours(),
      historical_success_rate: totalAttempts > 0 ? successCount / totalAttempts : 0.5,
      avg_days_to_pay: this.calculateAvgDaysToPay(paymentHistory),
    };
  }
}

// packages/ai/src/dunning/predictor.ts

export class DunningPredictor {
  /**
   * Predict the probability of successful payment if retried now,
   * and recommend optimal retry time.
   */
  async predict(features: DunningFeatures): Promise<DunningPrediction> {
    // Initial implementation: heuristic model
    // Later: trained ML model (gradient boosted trees)
    let confidence = features.historical_success_rate;

    // Adjust for day of week (midweek typically better)
    if (features.day_of_week >= 1 && features.day_of_week <= 4) confidence *= 1.1;

    // Adjust for time of day (morning better for card retries)
    if (features.hour_of_day >= 9 && features.hour_of_day <= 11) confidence *= 1.05;

    // Adjust for days past due (longer = worse)
    confidence *= Math.max(0.3, 1 - features.days_past_due * 0.03);

    confidence = Math.min(1, Math.max(0, confidence));

    return {
      retry_now_confidence: confidence,
      optimal_retry_hour: this.findOptimalHour(features),
      optimal_retry_day: this.findOptimalDay(features),
      model_version: "dunning-heuristic-v1",
    };
  }
}
```

**Testing:**
- Feature extraction produces all required fields from payment history
- Heuristic model: customer with 95% historical success rate gets high confidence
- Heuristic model: customer with 30% historical success rate gets low confidence
- AI dunning integration: when confidence > threshold, scheduler uses AI-predicted time instead of campaign delay
- When confidence < threshold, falls back to static campaign schedule
- ai_context JSONB stored on dunning_attempts captures model version and features
- A/B comparison: track recovery rate for AI-timed vs. static-timed retries

### Task 11.2: Churn Prediction from Billing Signals

**What:** Build a churn risk scoring model that uses usage decline patterns, payment failure frequency, support interactions, and subscription downgrade signals to predict churn probability.

**Design:**

```typescript
// packages/ai/src/churn/scorer.ts

export interface ChurnSignals {
  usage_trend_30d: number;         // % change in usage over last 30 days
  usage_trend_90d: number;         // % change over 90 days
  payment_failure_rate_90d: number;
  days_since_last_login: number;
  plan_downgrade_count: number;
  support_ticket_count_30d: number;
  contract_renewal_days: number;   // days until contract renewal
  mrr: number;
}

export class ChurnScorer {
  score(signals: ChurnSignals): ChurnScore {
    let risk = 0;

    // Usage decline is the strongest signal
    if (signals.usage_trend_30d < -0.3) risk += 30;
    else if (signals.usage_trend_30d < -0.1) risk += 15;

    if (signals.usage_trend_90d < -0.5) risk += 25;
    else if (signals.usage_trend_90d < -0.2) risk += 10;

    // Payment failures
    if (signals.payment_failure_rate_90d > 0.3) risk += 20;
    else if (signals.payment_failure_rate_90d > 0.1) risk += 10;

    // Engagement
    if (signals.days_since_last_login > 30) risk += 10;
    if (signals.plan_downgrade_count > 0) risk += 15;

    return {
      risk_score: Math.min(100, risk),
      risk_level: risk >= 70 ? "high" : risk >= 40 ? "medium" : "low",
      top_signals: this.identifyTopSignals(signals),
      recommended_actions: this.recommendActions(risk, signals),
    };
  }
}
```

**Testing:**
- Customer with 50% usage decline over 30d scores >= 70 (high risk)
- Customer with stable usage and zero payment failures scores < 30 (low risk)
- Top signals correctly identify the strongest contributing factors
- Recommended actions include specific suggestions (e.g., "reach out to customer", "offer retention discount")
- `GET /v1/customers/:id/churn-risk` returns current churn score
- Batch scoring: `GET /v1/churn-risk?risk_level=high` returns all high-risk customers

### Task 11.3: ASC 606 AI Contract Classification

**What:** Build an AI classifier that analyzes contract terms to identify performance obligations and recommend allocation methods, reducing manual finance team effort.

**Testing:**
- Contract with "platform access + onboarding" correctly identifies 2 obligations
- Contract with "annual license + quarterly support" correctly classifies satisfaction timing
- AI confidence score indicates model certainty
- Low-confidence classifications are flagged for manual review
- Classification results stored in asc606_analysis JSONB

### Task 11.4: Pricing Recommendation Engine

**What:** Analyze usage patterns across the customer base to recommend pricing tier structures and identify revenue leakage opportunities.

**Testing:**
- Analysis of 1000 customers' usage data produces tier boundary recommendations
- Revenue leakage report identifies customers on plans misaligned with their usage
- Pricing simulation shows projected revenue impact of recommended changes
- A/B experiment framework supports controlled pricing tests with statistical significance

### Phase 11 -- Definition of Done

- [ ] AI dunning optimizer predicts retry timing with measurable improvement over static schedules
- [ ] Churn prediction scores customers with explainable risk factors
- [ ] ASC 606 classifier reduces manual obligation identification effort
- [ ] Pricing recommendation engine produces actionable tier suggestions
- [ ] All AI decisions logged with model version and feature context for auditability
- [ ] AI features are behind feature flags and can be disabled per tenant
- [ ] Heuristic models in place with clear upgrade path to trained ML models

---

## Phase 12: Self-Service Portal & Integrations

**Goal:** Build the customer-facing self-service portal and pre-built integrations with CRM, ERP, and accounting systems.

**Duration estimate:** 4-5 weeks

### Task 12.1: Customer Self-Service Portal

**What:** Build a white-labelable customer portal where end customers can manage their subscription, view usage, update payment methods, and download invoices.

**Design:**

```
Portal Features:
- View current subscription plan and status
- View real-time usage dashboards per meter
- Upgrade/downgrade plan (triggers amendment flow)
- Update payment method (Stripe Elements for PCI-compliant card entry)
- View invoice history and download PDFs
- View credit balance and transaction history
- Manage billing contact information
```

```typescript
// Portal is a separate Next.js application that consumes the billing API
// Authentication via customer portal tokens (separate from API keys)

// packages/api/src/routes/portal.ts

export async function portalRoutes(app: FastifyInstance) {
  // Customer-facing endpoints (authenticated via portal token, not API key)

  app.get("/portal/subscription", { schema: portalSubscriptionSchema }, async (req) => {
    const customerId = req.portalCustomerId;
    const subscriptions = await subscriptionRepo.findByCustomer(customerId);
    return subscriptions.map(formatPortalSubscription);
  });

  app.get("/portal/usage", { schema: portalUsageSchema }, async (req) => {
    const customerId = req.portalCustomerId;
    return usageService.getCurrentUsageForCustomer(customerId);
  });

  app.get("/portal/invoices", { schema: portalInvoicesSchema }, async (req) => {
    const customerId = req.portalCustomerId;
    return invoiceRepo.findByCustomer(customerId);
  });

  app.post("/portal/payment-method", { schema: updatePaymentMethodSchema }, async (req) => {
    const customerId = req.portalCustomerId;
    // Uses Stripe SetupIntent to securely collect card details
    return paymentMethodService.createSetupIntent(customerId);
  });
}
```

**Testing:**
- Portal login with valid token shows subscription details
- Usage dashboard displays current-period consumption per meter
- Plan upgrade triggers amendment flow and generates prorated invoice
- Payment method update via Stripe Elements stores tokenized card
- Invoice history shows all invoices with PDF download links
- Credit balance shows current balance and recent transactions
- Portal respects tenant white-label settings (logo, colors)

### Task 12.2: CRM & ERP Integrations

**What:** Build pre-built connectors for Salesforce, HubSpot, NetSuite, and QuickBooks.

**Design:**

```typescript
// packages/engine/src/integrations/interface.ts

export interface CRMIntegration {
  syncCustomer(customer: Customer): Promise<void>;
  syncSubscription(subscription: Subscription): Promise<void>;
  syncInvoice(invoice: Invoice): Promise<void>;
}

export interface AccountingIntegration {
  syncInvoice(invoice: Invoice): Promise<void>;
  syncPayment(payment: Payment): Promise<void>;
  syncJournalEntry(entry: JournalEntry): Promise<void>;
  syncCreditNote(creditNote: CreditNote): Promise<void>;
}

// Implementation: webhook-driven sync
// When billing events occur, integration workers push data to connected systems
```

**Testing:**
- Salesforce sync: new customer creates/updates Salesforce Account
- Salesforce sync: new subscription creates Salesforce Opportunity
- QuickBooks sync: finalized invoice creates QuickBooks Invoice
- QuickBooks sync: payment creates QuickBooks Payment
- NetSuite sync: journal entries sync to NetSuite GL
- Integration failure retries with exponential backoff
- Integration credentials stored encrypted at rest

### Task 12.3: Helm Chart & Production Deployment Guide

**What:** Finalize Kubernetes Helm chart with production-ready configuration, including horizontal pod autoscaling, PodDisruptionBudgets, resource limits, and observability.

**Testing:**
- `helm install` deploys all components to a Kubernetes cluster
- Health checks and readiness probes function correctly
- Horizontal pod autoscaler scales API pods under load
- Database migrations run as a Helm pre-install/pre-upgrade hook
- Secrets managed via Kubernetes Secrets or external secret manager
- Prometheus metrics endpoint exposes billing-specific metrics (invoices/second, payment success rate)

### Phase 12 -- Definition of Done

- [ ] Self-service portal allows customers to manage subscriptions, view usage, and update payment methods
- [ ] Portal is white-labelable (custom branding per tenant)
- [ ] Salesforce and HubSpot CRM integrations sync customers, subscriptions, and invoices
- [ ] QuickBooks and NetSuite accounting integrations sync invoices, payments, and journal entries
- [ ] Helm chart deploys full production stack to Kubernetes
- [ ] Production deployment guide covers TLS, database setup, secret management, and monitoring
- [ ] End-to-end test: create customer -> subscribe -> ingest usage -> generate invoice -> collect payment -> recognize revenue

---

## Definition of Done (Global)

Every phase must meet these criteria before being considered complete:

### Code Quality
- [ ] All code passes TypeScript strict mode with zero errors
- [ ] ESLint reports zero warnings or errors
- [ ] No `any` types except in explicitly justified gateway response handling
- [ ] All public functions and interfaces have JSDoc documentation

### Testing
- [ ] Unit test coverage exceeds 90% for business logic (pricing, invoicing, revenue recognition)
- [ ] Integration tests run against real PostgreSQL via Testcontainers
- [ ] All API endpoints have request/response validation tests
- [ ] Financial calculation tests include edge cases (zero amounts, maximum precision, multi-currency)
- [ ] No flaky tests in CI (all tests deterministic)

### Security
- [ ] No raw card data stored anywhere in the system (PCI DSS)
- [ ] All API endpoints require authentication
- [ ] Row-level security enforces tenant isolation at the database level
- [ ] API keys are bcrypt-hashed at rest
- [ ] Webhook signatures use HMAC-SHA256 with per-endpoint secrets
- [ ] All secrets managed via environment variables, never committed to source

### Documentation
- [ ] OpenAPI 3.1 spec covers all endpoints and is validated
- [ ] Architecture decision records (ADRs) document all major technology choices
- [ ] Database schema is documented with entity-relationship diagrams

### Operational Readiness
- [ ] All background workers have dead-letter queues for failed jobs
- [ ] Health check endpoints verify all dependencies (database, Redis, object storage)
- [ ] Structured JSON logging with correlation IDs across requests
- [ ] Prometheus metrics for key billing operations (invoice generation rate, payment success rate, dunning recovery rate)
- [ ] Docker Compose for development; Helm chart for production Kubernetes deployment

### Standards Compliance
- [ ] ISO 4217 currency codes used for all monetary values
- [ ] ISO 3166-1 country codes used for all jurisdiction references
- [ ] CloudEvents v1.0 format for usage event ingestion
- [ ] RFC 7807 Problem Details for all error responses
- [ ] OpenAPI 3.1 for API specification
- [ ] ASC 606 / IFRS 15 five-step model implemented for revenue recognition
