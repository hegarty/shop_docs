# ADR-0008: Generic analytics worker framework, not one service per job

## Status
Accepted

## Context
The platform will accumulate analytics jobs over time (`sales.channel.breakdown` today;
`inventory.low_stock`, `orders.unfulfilled`, `customer.repeat_rate`, `profit.margin`, and
others are the stated direction). One deployable Go service per job would mean N
deployments, N sets of Kubernetes manifests, N CI pipelines, and N times the operational
overhead for what is fundamentally the same shape of work: read a period + tenant, query
Postgres, produce a structured result.

## Decision
A single `shop_analytics` service containing a generic worker framework:

```go
type Job interface {
    Name() string
    Execute(ctx context.Context, tenantID string, period period.Range) (*Result, error)
}
```

New analytics capability is a new `Job` implementation registered with the worker, not a
new deployable service. The scheduler only emits work onto `analytics.jobs` — it never
computes analytics itself, keeping "when to run" separate from "what running means."

## Alternatives considered
- **One microservice per job**: rejected as excessive fragmentation for logic that shares
  a database, a result shape, and a delivery mechanism.
- **Analytics logic embedded directly in the scheduler**: rejected — conflates scheduling
  concerns (cron timing, timezone handling) with business logic (what a sales breakdown
  actually is), making both harder to test independently.

## Consequences
Adding `inventory.low_stock` later means writing a `Job` implementation and registering it
— no new repo, no new CI pipeline, no new Kubernetes deployment. The tradeoff: all jobs
share the same deployment's resource limits and failure blast radius, which is acceptable
at this scale and revisitable (splitting out a genuinely resource-heavy job) if one job's
resource profile ever diverges sharply from the rest.
