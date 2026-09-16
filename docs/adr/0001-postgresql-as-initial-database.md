# ADR-0001: PostgreSQL as the initial database

## Status
Accepted

## Context
The platform needs one store that serves as both the operational database (orders, jobs,
schedules) and the initial analytical database (the `sales.channel.breakdown` query and
whatever follows it).

## Decision
PostgreSQL, used for both roles. No DynamoDB, MongoDB, OpenSearch, or ClickHouse at this
stage.

## Alternatives considered
- **DynamoDB**: no free-form aggregation (`GROUP BY channel, collective_partner_id`) without
  either scanning or maintaining bespoke aggregate tables by hand. Wrong shape for the
  actual query this platform exists to answer.
- **ClickHouse**: excellent for large-scale analytical aggregation, but adds a second
  database to operate for a single-tenant MVP with a modest order volume. Real candidate
  once analytics moves to large-scale, high-cardinality queries (documented, not deployed —
  see cost-model.md).
- **MongoDB / OpenSearch**: neither offers a compelling advantage over Postgres for
  relational, tenant-scoped, exact-arithmetic financial data.

## Consequences
Postgres's `NUMERIC` type gives exact decimal arithmetic for money columns (paired with
`shop_platform/money`'s integer-cents representation at the application layer). Standard
SQL keeps the door open for a future LLM/text-to-SQL query layer without a translation
step. Revisit if/when a dedicated analytical engine becomes necessary at scale — the
event-sourced design (raw archive + normalized events) makes that migration additive, not
a rewrite.
