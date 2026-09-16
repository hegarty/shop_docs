# Database Model

PostgreSQL is both the operational store and the initial analytical database — see
[ADR-001](adr/0001-postgresql-as-initial-database.md). Every table below carries
`tenant_id`; every query is tenant-scoped. No table has a bare, ambiguous `sales` column —
see [event-model.md](event-model.md#money-fields--no-ambiguous-sales-field) for why.

Actual migration files live in each owning service's `migrations/` directory (see
[terraform-versioning.md](terraform-versioning.md)'s sibling convention — this repo
documents the schema, it doesn't own the `.sql` files).

## Core commerce schema (owned by `shop_ingestor`)

```sql
CREATE TABLE tenants (
    id          TEXT PRIMARY KEY,      -- e.g. 'devmoto'
    name        TEXT NOT NULL,
    timezone    TEXT NOT NULL,         -- IANA name, e.g. 'America/New_York'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE shopify_stores (
    id           BIGSERIAL PRIMARY KEY,
    tenant_id    TEXT NOT NULL REFERENCES tenants(id),
    shop_domain  TEXT NOT NULL UNIQUE,  -- e.g. 'devmoto.myshopify.com'
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE collective_partners (
    id            BIGSERIAL PRIMARY KEY,
    tenant_id     TEXT NOT NULL REFERENCES tenants(id),
    name          TEXT NOT NULL,        -- the tag value, e.g. 'ABC Bikes'
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE orders (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               TEXT NOT NULL REFERENCES tenants(id),
    shopify_order_id        TEXT NOT NULL,
    order_number            TEXT NOT NULL,

    created_at              TIMESTAMPTZ NOT NULL,  -- Shopify's created_at — see event-model.md for why this, not processed_at
    updated_at              TIMESTAMPTZ NOT NULL,
    processed_at            TIMESTAMPTZ,

    currency                TEXT NOT NULL,

    -- All money columns are NUMERIC(12,2), never FLOAT — exact decimal
    -- arithmetic matters for financial totals. shop_platform/money's
    -- Amount (int64 cents) is what the application layer uses; the DB
    -- stores the same value as a decimal for direct SQL aggregation.
    gross_sales             NUMERIC(12,2) NOT NULL,
    discounts               NUMERIC(12,2) NOT NULL DEFAULT 0,
    returns                 NUMERIC(12,2) NOT NULL DEFAULT 0,
    net_sales               NUMERIC(12,2) NOT NULL,
    shipping                NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax                     NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_sales             NUMERIC(12,2) NOT NULL,

    channel                 TEXT NOT NULL CHECK (channel IN ('shop', 'collective', 'unknown')),
    collective_partner_id   BIGINT REFERENCES collective_partners(id),

    financial_status        TEXT NOT NULL,
    fulfillment_status      TEXT NOT NULL,

    -- Data quality / replay support
    classification_version  INT NOT NULL DEFAULT 1,
    normalization_version   INT NOT NULL DEFAULT 1,
    source_event_id         TEXT NOT NULL,   -- the RawEnvelope.event_id that produced this row
    reconciled_at           TIMESTAMPTZ,     -- last time reconciliation confirmed this row against Shopify
    raw_metadata            JSONB NOT NULL,  -- enough of the source payload to investigate classification errors

    UNIQUE (tenant_id, shopify_order_id)
);

CREATE INDEX idx_orders_tenant_created ON orders (tenant_id, created_at);
CREATE INDEX idx_orders_tenant_channel ON orders (tenant_id, channel, created_at);

CREATE TABLE order_line_items (
    id             BIGSERIAL PRIMARY KEY,
    order_id       BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    tenant_id      TEXT NOT NULL REFERENCES tenants(id),
    shopify_line_item_id TEXT NOT NULL,
    title          TEXT NOT NULL,
    quantity       INT NOT NULL,
    price          NUMERIC(12,2) NOT NULL,

    UNIQUE (order_id, shopify_line_item_id)
);
```

Query shape this schema is built for (the actual `sales.channel.breakdown` query):

```sql
SELECT
    channel,
    collective_partner_id,
    SUM(net_sales)  AS net_sales,
    SUM(total_sales) AS total_sales,
    COUNT(*)        AS order_count
FROM orders
WHERE tenant_id = $1
  AND created_at >= $2
  AND created_at <  $3
GROUP BY channel, collective_partner_id;
```

`idx_orders_tenant_channel` covers this directly. No further indexes added speculatively —
add them when a real query pattern demands it.

## Analytics schema (owned by `shop_analytics`)

```sql
CREATE TABLE job_schedules (
    id             BIGSERIAL PRIMARY KEY,
    tenant_id      TEXT NOT NULL REFERENCES tenants(id),
    job_name       TEXT NOT NULL,          -- e.g. 'sales.channel.breakdown'
    frequency      TEXT NOT NULL,          -- cron expression
    timezone       TEXT NOT NULL,
    enabled        BOOLEAN NOT NULL DEFAULT true,
    configuration  JSONB NOT NULL DEFAULT '{}',
    next_run_at    TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE analytics_jobs (
    id            BIGSERIAL PRIMARY KEY,
    tenant_id     TEXT NOT NULL REFERENCES tenants(id),
    job_name      TEXT NOT NULL,
    period_start  TIMESTAMPTZ NOT NULL,
    period_end    TIMESTAMPTZ NOT NULL,
    status        TEXT NOT NULL DEFAULT 'pending', -- pending | running | completed | failed
    requested_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ,
    error         TEXT
);

CREATE TABLE analytics_results (
    id            BIGSERIAL PRIMARY KEY,
    job_id        BIGINT NOT NULL REFERENCES analytics_jobs(id),
    tenant_id     TEXT NOT NULL REFERENCES tenants(id),
    job_name      TEXT NOT NULL,
    period_start  TIMESTAMPTZ NOT NULL,
    period_end    TIMESTAMPTZ NOT NULL,
    result        JSONB NOT NULL,     -- the structured Result (see architecture.md example)
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analytics_results_tenant_job ON analytics_results (tenant_id, job_name, period_start);
```

## Notification schema (owned by `shop_notifier`)

```sql
CREATE TABLE notification_channels (
    id            BIGSERIAL PRIMARY KEY,
    tenant_id     TEXT NOT NULL REFERENCES tenants(id),
    channel_type  TEXT NOT NULL,   -- 'sms' | 'email' | 'slack' | ...
    configuration JSONB NOT NULL,  -- phone number, webhook URL, etc. — never a raw secret; references a Secrets Manager entry instead
    enabled       BOOLEAN NOT NULL DEFAULT true
);
```

## Multi-tenancy

Every table above is tenant-scoped by a plain `tenant_id` foreign key column and row-level
filtering in application queries — not Postgres row-level security (RLS). RLS was
considered and deliberately skipped for the MVP: with a single Go codebase and a single
tenant today, the extra operational complexity (policies, `SET ROLE` per request, testing
the policies themselves) isn't earning its cost yet. Revisit before this platform has
external customers with direct DB access of any kind (there is currently none) — see
[ADR-007](adr/0007-multi-tenancy-from-day-one.md).
