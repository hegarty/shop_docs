# ADR-0005: Single-node Redpanda for the MVP

## Status
Accepted (explicit, temporary tradeoff)

## Context
A production Kafka/Redpanda deployment normally runs 3+ brokers for availability. That
costs meaningfully more in compute and EBS than this single-tenant MVP's traffic justifies.

## Decision
One Redpanda broker, on a persistent EBS volume, for the MVP.

## Consequences
If the broker (or its node) is lost, ingestion and analytics stall until it's redeployed,
then reconciliation (Shopify Admin API polling with an overlap window) catches Postgres
back up to current. No permanent data loss, because Redpanda is deliberately not the
source of truth (see ADR-0002 and disaster-recovery.md) — but real downtime during the
outage window.

## Revisit when
A tenant's business can't tolerate that downtime window, or a second tenant onboards with
different availability expectations. Upgrade path: 3+ broker Redpanda cluster, or migrate
to Redpanda Cloud — both are additive infrastructure changes, not an application rewrite,
since `shop_platform/redpanda` already talks to Redpanda over the standard Kafka protocol.
