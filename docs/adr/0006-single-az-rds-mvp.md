# ADR-0006: Single-AZ RDS for the MVP

## Status
Accepted (explicit, temporary tradeoff)

## Context
Multi-AZ RDS roughly doubles the instance cost for automatic failover this single-tenant
MVP's uptime requirements don't yet justify.

## Decision
Single-AZ `db.t4g.micro` Postgres, encrypted, with 7-day automated backups and
`deletion_protection = true`.

## Consequences
An AZ-level failure affecting the RDS instance causes downtime until AWS restores the
instance or a manual restore-from-backup completes. Given the raw S3 archive and Shopify
itself as replay sources, this is data-loss-resistant even though it isn't
downtime-resistant.

## Revisit when
Real uptime expectations exist (a paying customer with an SLA) or query load outgrows
`db.t4g.micro` — whichever comes first. Enabling Multi-AZ at that point is a single
Terraform variable flip (`multi_az = true`), not a schema or application change.
