# Milestones

The [initial prompt](initial-prompt.md) broken into trackable milestones and tasks. Update
this file as work completes or scope changes — check a box only once it's actually true
(verified, not "should be done by now"). For day-to-day operating status (what's pushed vs.
local-only, what's currently broken), see [`../SESSION.md`](../SESSION.md) instead.

Status legend: ✅ done · 🟡 in progress / partially done · ⬜ not started

## Product vision (from the initial prompt — the north star, not a milestone itself)

```
Phase 1: "What happened?"     <- current focus, all milestones below serve this
Phase 2: "Why did it happen?"
Phase 3: "What should I do?"
Phase 4: "Do it for me."
```

---

## Milestone 0 — Foundation & governance ✅

- [x] Inspect existing `terraform` and `eks` conventions before creating anything
- [x] Design the implementation plan (repos, modules, Terragrunt hierarchy, cost estimate,
      MVP compromises) and get it approved before writing code
- [x] Design tag-based Terraform module versioning (`<module-path>/vMAJOR.MINOR.PATCH`,
      `make release`) — see [`docs/terraform-versioning.md`](../docs/terraform-versioning.md),
      [ADR-0010](../docs/adr/0010-terraform-module-versioning.md)
- [x] Decide repo boundaries: 5 repos (`shop_platform`, `shop_ingestor`, `shop_analytics`,
      `shop_notifier`, `shop_docs`) — not 6+ per-service repos, not 1 monorepo
- [x] Create the 5 `shop_*` GitHub repos, public, governed by
      [`github_policy-as-code`](https://github.com/hegarty/github_policy-as-code)
      (branch protection, TruffleHog, Dependabot, CodeQL) — see
      [ADR-0011](../docs/adr/0011-github-repo-governance-via-policy-as-code.md)
- [x] Create `shop_platform`: event envelopes, money (integer cents), timezone-aware period
      math, Redpanda/Postgres wrappers, OTel bootstrap, structured logging, fail-fast config
- [x] Correct two early decisions from the initial prompt, in conversation, before
      implementation: repo prefix `commerce-intel-` → `shop_`; repos private → public
- [x] Decide the new shared cluster (`eks-prod`) is separate from the personal-lab
      `eks-dev` cluster, but explicitly *not* commerce-intel-scoped — see
      [ADR-0004](../docs/adr/0004-single-eks-cluster.md)
- [x] Write the initial architecture doc set (`architecture.md`, `event-model.md`,
      `database-model.md`, `security.md`, `observability.md`, `cost-model.md`,
      `disaster-recovery.md`) and ADR-0001 through ADR-0010

## Milestone 1 — Data pipeline (code) ✅

- [x] Confirm, via live Shopify documentation (not assumption), exactly how to distinguish
      a Shopify Collective order from a direct order and identify the partner (tags —
      `"Shopify Collective"` + one partner-name tag) — see `event-model.md`'s citations
- [x] Confirm the money-field derivation against Shopify's current GraphQL Admin API
      schema (no single "pre-discount total" field exists; gross sales is summed from line
      items) — corrected once already after the initial pass got this wrong
- [x] `shop_ingestor`: webhook receiver — HMAC verification, idempotency (webhook-delivery-ID
      based), raw S3 archival, publish to `shopify.orders.raw`
- [x] `shop_ingestor`: normalization — Collective channel/partner classification, money-field
      derivation, upsert to Postgres, publish `commerce.order.*` to `shopify.orders.normalized`
- [x] Decide webhooks are treated as change notifications (re-fetch via GraphQL) rather than
      parsing the webhook body directly — see
      [ADR-0012](../docs/adr/0012-webhook-as-change-notification.md)
- [x] `shop_ingestor`: reconciliation job against Shopify's Admin GraphQL API, with overlap
      window, intended as a Kubernetes CronJob
- [x] `shop_analytics`: generic `Job`/`Registry` worker framework — new analytics capability
      is a new `Job` implementation, not a new service — see
      [ADR-0008](../docs/adr/0008-generic-analytics-worker-framework.md)
- [x] `shop_analytics`: scheduler (cron-driven, tenant-timezone-aware, emits
      `analytics.jobs`, never computes analytics itself)
- [x] `shop_analytics`: **`sales.channel.breakdown`**, the platform's first required
      analytics job — output verified byte-for-byte against the original brief's worked
      example (Shop $6,842 / Collective $3,412 across 4 partners / 45 orders / AOV $227.87)
- [x] `shop_notifier`: consumes `notifications.requested` (published alongside, not instead
      of, `analytics.results`), formats via `internal/format` (SMS-style output matches the
      brief's example exactly), delivers via a vendor-agnostic `Sender` interface
- [x] All four Go repos: unit tests (table-driven where the logic branches on input shape),
      `golangci-lint`, `govulncheck`, Docker images, CI wired to each repo's governance

## Milestone 2 — Infrastructure realized 🟡

- [x] Design the `eks-prod` Terragrunt tree: networking (single NAT), EKS (pure API auth
      mode), Karpenter (Spot-first), Cilium, cert-manager, Redpanda (single broker),
      self-hosted kube-prometheus-stack/Loki/Tempo/OTel Collector, plus
      `workloads/commerce-intel/` for this project's own RDS/S3/ECR/secrets/Pod Identity
- [x] Design new generic Terraform modules: `eks/pod_identity`, `eks/karpenter`,
      `rds-postgres`, `secrets-manager`, `budgets`; generalize `eks/addons/helm`; fix a
      pre-existing bug in `eks/storage_class`; add lifecycle/tagging support to `s3`,
      `ecr`, `networking/security_groups`
- [x] Commit and push all three infrastructure repos (`terraform` PR #4, `eks` PR #5,
      new `eks-prod` repo + bootstrap commit + governance) — done 2026-09-20, after these
      sat local-only long enough to be the platform's biggest risk (see `SESSION.md`)
- [ ] Merge `terraform` PR #4 and `eks` PR #5 (open, awaiting review — not merged
      automatically since they touch real infrastructure definitions)
- [ ] Bootstrap-tag every `terraform` module referenced by `eks-prod` at `v1.0.0`
      (`make release MODULE=... VERSION=1.0.0` — prints commands, requires explicit
      confirmation per module)
- [ ] Bootstrap `eks-prod`'s remote state (S3 bucket + DynamoDB lock table) in AWS account
      868150784168 — commands in `eks-prod/README.md`
- [ ] Replace the placeholder admin access-entry ARN in
      `eks-prod/us-east-1/eks/access_entries/users/terragrunt.hcl`
- [ ] Work through `docs/deployment.md`'s Terragrunt apply order (networking → EKS →
      Karpenter/Cilium/addons → commerce-intel workload resources)
- [ ] Populate Shopify + SMS provider Secrets Manager shells out-of-band; create the
      Grafana admin credential Kubernetes Secret

## Milestone 3 — Kubernetes packaging & observability ⬜

Explicitly deferred ("back burner") at the user's request on 2026-09-20 — not forgotten,
just sequenced after infrastructure lands.

- [ ] Helm charts or manifests for `shop_ingestor`, `shop_analytics`, `shop_notifier`
      (ServiceAccounts, EKS Pod Identity annotations, health probes, resource
      requests/limits)
- [ ] `CiliumNetworkPolicy` definitions matching the documented communication model
      (default-deny + explicit allow-list per service)
- [ ] Cilium Gateway API objects for the Shopify webhook's external endpoint
- [ ] Grafana dashboards: platform overview, Shopify ingestion, Redpanda health, analytics
      workers, PostgreSQL health, notifications, Cilium/Hubble network visibility, and the
      business dashboard (sales today, shop/Collective split, partner breakdown, AOV)
- [ ] Verify OTel instrumentation actually reaches Prometheus/Tempo/Loki once the
      collector and the three services are both deployed (code exists on both sides;
      never exercised end-to-end yet)

## Milestone 4 — Production cutover ⬜

- [ ] Pick and integrate a real SMS vendor behind `shop_notifier`'s `vendor.Sender`
      (ships today with only a log-and-don't-send default — deliberate, not an oversight)
- [ ] Deploy all three application services to `eks-prod`
- [ ] Register the real Shopify webhook against the `devmoto` store — only after
      `shop_ingestor` is deployed and verified healthy, never before, never automated
- [ ] End-to-end verification with a real order: webhook → normalize → store → scheduled
      analytics run → notification delivered → dashboards show it
- [ ] First real `sales.channel.breakdown` delivered to the actual business owner

## Milestone 5+ — Phase 2 and beyond ⬜

Not started, intentionally not detailed yet — revisit once Milestone 4 is live and stable.
Directional, from the initial prompt's future-scale section:

- [ ] Additional analytics jobs (`inventory.low_stock`, `orders.unfulfilled`,
      `customer.repeat_rate`, `profit.margin`, `product.performance`, ...)
- [ ] Additional notification channels (email, Slack) behind the existing `Sender`
      interface
- [ ] Real-time streaming consumers (large-order alerts, live revenue) alongside the
      scheduled path
- [ ] Second tenant onboarding — the point at which single-node Redpanda, single-AZ RDS,
      and the single NAT Gateway (ADR-0005, ADR-0006, and the cost-model's NAT tradeoff)
      should be revisited, not before
- [ ] Non-Shopify integrations (QuickBooks, Stripe, Meta/Google Ads, Klaviyo, ShipStation,
      GA4, ...) normalized into the same canonical `commerce.*`/future event families
- [ ] Web dashboard and public API surface consuming `analytics.results` directly
- [ ] Phase 2/3/4 of the product vision: "why," "what should I do," "do it for me"
