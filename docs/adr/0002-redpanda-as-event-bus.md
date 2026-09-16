# ADR-0002: Redpanda as the event bus

## Status
Accepted

## Context
The platform needs an event bus decoupling ingestion, normalization, analytics, and
notification, and needs it to be inexpensive to run for one tenant while remaining
Kafka-API-compatible for future scale (managed Kafka, Redpanda Cloud, etc.).

## Decision
Redpanda, self-hosted on `eks-prod`, accessed via `shop_platform/redpanda` (built on
franz-go, a Kafka-protocol client).

## Alternatives considered
- **Amazon MSK**: fully managed, but meaningfully more expensive at this scale and adds an
  AWS-proprietary dependency this platform is otherwise avoiding (see the broader
  cloud-portability goal).
- **NATS**: simpler operationally, but the Kafka-compatible ecosystem (schema tooling,
  Kafka Connect if ever needed, broad client support) is worth more here than NATS's
  simplicity, especially once integrations beyond Shopify are in scope.
- **SQS/SNS**: no consumer-group semantics or replay-by-offset, both of which this
  platform's replay/reprocessing story depends on.

## Consequences
Redpanda is explicitly **not** the source of truth (see disaster-recovery.md) — it's
transport. This lets the MVP run a single broker without treating that as a real
durability risk. The tradeoff: a lost broker stalls the pipeline until reconciliation
catches back up, which is an acceptable MVP availability compromise, not acceptable
indefinitely (see ADR-0005).
