# ADR-0007: Multi-tenancy from day one

## Status
Accepted

## Context
The first customer is one Shopify business. The stated intent is a commercial SaaS
product with more tenants later. Retrofitting tenant scoping onto a single-tenant schema
and event model after the fact is a much larger, riskier change than building it in from
the start.

## Decision
Every event carries `tenant_id`. Every table carries `tenant_id`. Every query is
tenant-scoped in application code. No user/identity system is built yet — that's a
separate, larger concern deliberately deferred (see below) — but no architectural decision
here should make adding one later harder.

## Alternatives considered
- **Single-tenant now, multi-tenant later**: rejected — the migration cost (backfilling
  `tenant_id` everywhere, auditing every query for tenant scoping after the fact) is larger
  and riskier than the small amount of extra work multi-tenancy costs today.
- **Postgres Row-Level Security (RLS) for tenant isolation**: considered and deferred, not
  rejected outright. With one Go codebase as the only DB client today, RLS's operational
  complexity (policies, `SET ROLE` per request, testing the policies themselves) isn't
  earning its cost yet. Revisit before any external party gets direct database access of
  any kind — there is currently none.

## Consequences
Multi-tenancy is foundational, not bolted on: tenant + Shopify stores + integrations +
analytics jobs + schedules + notification channels + alert rules are all designed as
per-tenant concerns from the start. A full user/identity/auth system (multiple human users
per tenant, roles, permissions) is explicitly **not** implemented yet — deliberately kept
separate from tenant *data* scoping, which is the part that's expensive to retrofit.
