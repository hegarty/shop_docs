# shop_docs

Architecture documentation, ADRs, and the deployment runbook for the commerce-intel
platform — a Shopify sales analytics and monitoring platform, launching with one tenant
(a real Shopify business) and built to grow into a multi-tenant commerce-intelligence
product.

**Picking this up cold?** Read [`SESSION.md`](SESSION.md) first — it tracks what's
actually pushed vs. still local-only across this platform's six repos, and the current
known gaps.

## Repositories

| Repo | Purpose |
|---|---|
| [`shop_platform`](https://github.com/hegarty/shop_platform) | Shared Go library: event envelopes, money, period math, Redpanda/Postgres wrappers, OTel, logging, config |
| [`shop_ingestor`](https://github.com/hegarty/shop_ingestor) | Shopify webhook receiver, reconciliation, normalization |
| [`shop_analytics`](https://github.com/hegarty/shop_analytics) | Scheduler + analytics worker framework + jobs |
| [`shop_notifier`](https://github.com/hegarty/shop_notifier) | Notification delivery (SMS first, vendor-agnostic) |
| `eks-prod` | Terragrunt config for the shared EKS cluster this platform (and future unrelated projects) runs on. **Not yet pushed to GitHub** — see `SESSION.md`. |
| [`terraform`](https://github.com/hegarty/terraform) | Reusable, tag-versioned Terraform modules. New modules for this platform are written but not yet committed — see `SESSION.md`. |

## Docs

- [architecture.md](docs/architecture.md) — system overview, event flow, MVP compromises
- [event-model.md](docs/event-model.md) — envelope shapes, topics, canonical event types, Shopify Collective classification
- [database-model.md](docs/database-model.md) — schema, metric definitions, indexes
- [security.md](docs/security.md) — secrets handling, network policy, IAM boundaries
- [observability.md](docs/observability.md) — metrics/logs/traces, dashboards, retention
- [cost-model.md](docs/cost-model.md) — AWS cost breakdown and drivers
- [terraform-versioning.md](docs/terraform-versioning.md) — module tagging convention
- [local-development.md](docs/local-development.md) — running services locally
- [deployment.md](docs/deployment.md) — the ordered runbook for standing up the platform
- [disaster-recovery.md](docs/disaster-recovery.md) — data ownership, recovery procedures
- [adr/](docs/adr/README.md) — architecture decision records
