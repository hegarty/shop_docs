# SESSION.md — running context for Claude

Read this first if picking up this project cold (new terminal session, context loss, or a
different Claude session/agent). It's a living document — update it at the end of any
session that changes the project's state, and especially whenever a "known gap" below gets
closed or a new one is found. It is **public** (this repo is public): keep it to this
platform's own architecture and operating status, not anything a future paying customer
shouldn't see (there is currently no customer data in scope, so this is a low-stakes rule
today, but keep it in mind as tenants are onboarded).

## What this project is

A Shopify sales analytics and monitoring platform (`commerce-intel`), launching with one
tenant (a real Shopify business, referred to internally as `devmoto`) and designed from
day one to grow into a multi-tenant commerce-intelligence product. Full architecture:
[`docs/architecture.md`](docs/architecture.md). Why things are built the way they are:
[`docs/adr/`](docs/adr/README.md).

## Repositories — this spans six, not one

| Repo | Contains | Status |
|---|---|---|
| [`hegarty/shop_platform`](https://github.com/hegarty/shop_platform) | Shared Go library | ✅ implemented, pushed, CI green |
| [`hegarty/shop_ingestor`](https://github.com/hegarty/shop_ingestor) | Webhook receiver, normalization, reconciliation | ✅ implemented, pushed, CI green |
| [`hegarty/shop_analytics`](https://github.com/hegarty/shop_analytics) | Scheduler + analytics worker framework | ✅ implemented, pushed, CI green |
| [`hegarty/shop_notifier`](https://github.com/hegarty/shop_notifier) | Notification delivery | ✅ implemented, pushed, CI green (log-only Sender — see Known gaps) |
| [`hegarty/shop_docs`](https://github.com/hegarty/shop_docs) | This repo — architecture, ADRs, runbook, roadmap | ✅ pushed |
| [`hegarty/terraform`](https://github.com/hegarty/terraform) (existing personal repo, not `shop_`-prefixed) | Reusable Terraform modules | ✅ pushed, open PR [#4](https://github.com/hegarty/terraform/pull/4) awaiting review/merge |
| [`hegarty/eks`](https://github.com/hegarty/eks) (existing personal repo, `eks-dev`) | Terragrunt for the personal-lab cluster | ✅ pushed, open PR [#5](https://github.com/hegarty/eks/pull/5) awaiting review/merge |
| [`hegarty/eks-prod`](https://github.com/hegarty/eks-prod) | Terragrunt for the new shared `eks-prod` cluster | ✅ pushed to `main`, governed |

As of 2026-09-20, everything above is committed and pushed. The `terraform` and `eks` PRs
are deliberately left unmerged — they touch real infrastructure definitions and are
waiting on the user's review, not on any more work.

## Operating model

```
Application code (shop_platform, shop_ingestor, shop_analytics, shop_notifier, shop_docs):
  PR → CI (trufflehog + build required; lint/govulncheck/docker/CodeQL visible, not required)
  → merge (0 required reviews, solo maintainer) → main

Infrastructure (terraform, eks, eks-prod):
  Claude generates/validates locally (fmt, validate, hclfmt) — NEVER applies.
  All terraform/terragrunt apply, module tag pushes, and AWS resource creation
  are run BY THE USER, reviewed first. See docs/deployment.md.
```

Repository governance (branch protection, required checks, secret scanning, Dependabot,
CodeQL) is applied via
[`github_policy-as-code`](https://github.com/hegarty/github_policy-as-code)'s
`plan`/`apply`/`verify` cycle against a local policy override (not committed to any
`shop_*` repo — it sets `enforcement: active` instead of that tool's own `evaluate`
default, since this account isn't on GitHub Enterprise). See
[ADR-0011](docs/adr/0011-github-repo-governance-via-policy-as-code.md).

## Current state (as of this file's creation, 2026-09-20)

- All five `shop_*` repos are implemented, pushed, and passing full CI (not just required
  checks) on `main`. Zero open PRs, zero open Dependabot alerts, across all five, as of
  this writing.
- The full Dependabot backlog (31 PRs across the four Go repos) was merged in one pass on
  2026-09-20. Doing that surfaced two real, previously-invisible CI bugs (see Known gaps
  §"Closed this session" for what they were and how they were found) — both are fixed and
  merged now.
- `sales.channel.breakdown` (the platform's first, launch-blocking analytics job) is fully
  implemented and its output is verified byte-for-byte against the original product
  brief's worked example (Shop $6,842 / Collective $3,412 across 4 partners / 45 orders /
  AOV $227.87) — see `shop_analytics/internal/jobs/sales_channel_breakdown_test.go`.
- Shopify Collective channel/partner classification (`shop_ingestor/internal/normalize`)
  and the money-field derivation (gross/discounts/returns/net/shipping/tax/total) are
  implemented against Shopify's **current, live-verified** GraphQL Admin API schema — see
  [`docs/event-model.md`](docs/event-model.md)'s citations. This was explicitly researched
  during the build, not assumed.
- `shop_notifier` ships with only a log `Sender` wired up — no real SMS vendor is
  integrated (deliberate; see that repo's README).
- The `eks-prod` Terragrunt tree (35 units: networking, EKS, Karpenter, Cilium, Redpanda,
  self-hosted observability, plus commerce-intel's own RDS/S3/ECR/secrets/Pod Identity) is
  fully designed, locally `terragrunt hclfmt`-clean, targeting its own AWS account
  (868150784168, separate from `eks-dev`'s 891377023413), **and now pushed** to a new,
  governed `hegarty/eks-prod` repo.
- `hegarty/terraform`'s new/modified modules (`eks/pod_identity`, `eks/karpenter`,
  `rds-postgres`, `secrets-manager`, `budgets`, plus small additive changes to `ecr`, `s3`,
  `eks/cluster`, `eks/addons/helm`, `eks/storage_class`, `networking/security_groups`) are
  written, pass `terraform validate`, and are pushed — stacked onto that repo's existing
  open PR #4 (see Known gaps for why), not yet merged.
- No terraform module has ever been tagged (`git tag -l` in that repo is empty) — the
  `v1.0.0` bootstrap tagging described in
  [`docs/terraform-versioning.md`](docs/terraform-versioning.md) has not been run yet. Every
  `eks-prod` Terragrunt unit's `?ref=<module>/v1.0.0` will fail to resolve until this
  happens, in addition to the module code itself needing to be pushed first.
- Helm charts/Kubernetes manifests for the three application services, Grafana dashboards,
  and `CiliumNetworkPolicy` definitions do not exist yet — explicitly deferred ("back
  burner") at the user's request on 2026-09-20, not forgotten.

## Known gaps (real, not hypothetical)

1. **Resolved 2026-09-20 (previously the top gap here): the entire infrastructure layer
   was uncommitted and, for `eks-prod`, not even on GitHub.** For the record, since this
   was a real risk worth learning from: `terraform`'s new module code and `eks`'s two
   intentional edits had been sitting as uncommitted working-tree changes since the session
   that wrote them; `eks-prod` was `git init`'d locally with all 45 files staged but zero
   commits and no GitHub remote at all — the platform's entire cluster design existed
   nowhere but one machine's working directory. None of this was a mistake in isolation
   (Terraform/Terragrunt work is generated and validated locally by design — the user
   performs all applies), but it went unnoticed for longer than it should have because
   nothing was tracking cross-repo state until this file started existing. Fixed by
   committing and pushing all three: `terraform`'s work landed on its existing open PR #4
   (it modifies the `s3` module that PR itself introduces, so it couldn't cleanly go
   anywhere else); `eks`'s two fixes got a fresh PR #5 off `main`, kept separate from an
   unrelated open PR on that repo; `eks-prod` got a new GitHub repo, a bootstrap commit,
   and the same governance as every other repo here. Both `terraform`#4 and `eks`#5 are
   still open, deliberately — merging real infrastructure definitions is the user's call.

2. **Go directive inconsistency across the four Go repos.** `shop_platform`,
   `shop_ingestor`, and `shop_analytics` have `go 1.25.0` in `go.mod`; `shop_notifier` has
   `go 1.25.14`. Harmless (all four build with whatever 1.25.x toolchain is installed,
   currently 1.25.14 via asdf), but worth converging next time any of them gets a
   dependency bump, for consistency.

3. **Closed this session, recorded for the trail**: merging the Dependabot backlog
   surfaced two real bugs that had been silently broken since the Go 1.23→1.25 bump
   earlier in the project (for a pgx security fix): (a) `golangci-lint-action` was pinned
   to a version whose installed binary couldn't parse a `go 1.25` `go.mod`, so `lint` had
   been failing (uninformatively) on every PR for days; a Dependabot PR bumping that action
   to v9 fixed it outright. (b) `shop_ingestor` and `shop_analytics`'s Dockerfiles were
   still pinned to `golang:1.24.13-bookworm`, so their `docker` CI job's `go mod download`
   had been failing since the same bump. Both fixed via shop_platform#11,
   shop_ingestor#12, shop_analytics#10 — see [ADR-0011](docs/adr/0011-github-repo-governance-via-policy-as-code.md)
   for why neither of these blocked merges the whole time they were broken (by design —
   `lint`/`docker` are visible, not required, checks).

4. **No real SMS vendor.** `shop_notifier` logs instead of sending. See that repo's
   README/CONTRIBUTING for how to add one behind `vendor.Sender`.

5. **`eks-prod`'s admin access-entry ARN is a placeholder.** `us-east-1/eks/access_entries/users/terragrunt.hcl`
   has `principal_arn = "arn:aws:iam::868150784168:role/REPLACE_ME_ADMIN_ROLE"` — must be
   replaced with a real role in that account before `terragrunt apply` there.

6. **No CiliumNetworkPolicy manifests, no Grafana dashboards, no Helm charts for the three
   application services.** Deferred by explicit user request (2026-09-20), tracked in
   `docs/deployment.md` step 5's "not yet implemented" note, not lost track of.

## Roadmap / next steps

See [`roadmap/milestones.md`](roadmap/milestones.md) for the full, checkbox-tracked
breakdown. Immediate next steps:

1. Review and merge `terraform` PR #4 and `eks` PR #5 (both open, CI green).
2. Bootstrap-tag every `terraform` module at `v1.0.0` (`make release MODULE=... VERSION=1.0.0`,
   prints the tag/push commands for review — see `docs/terraform-versioning.md`).
3. Bootstrap `eks-prod`'s remote state in AWS account 868150784168 (S3 bucket + DynamoDB
   lock table — exact commands in `eks-prod/README.md`), then work through
   `docs/deployment.md`'s Terragrunt apply order.
4. Build Helm charts/Kubernetes manifests for `shop_ingestor`/`shop_analytics`/
   `shop_notifier`, `CiliumNetworkPolicy` definitions, and Grafana dashboards (the "back
   burner" work).
5. Pick and integrate a real SMS vendor behind `shop_notifier`'s `vendor.Sender`.
6. Register the real Shopify webhook against the `devmoto` store once `shop_ingestor` is
   actually deployed and reachable — never before then, and never without the user running
   that command themselves (see `docs/deployment.md` step 6).

## Where to look for more

- [`roadmap/`](roadmap/README.md) — the original brief this platform was commissioned
  from (`roadmap/initial-prompt.md`), and its milestone breakdown (`roadmap/milestones.md`).
  This file (`SESSION.md`) is the volatile day-to-day status; the roadmap is the longer arc.
- `docs/architecture.md`, `docs/event-model.md`, `docs/database-model.md`,
  `docs/security.md`, `docs/observability.md`, `docs/cost-model.md` — the "why" behind
  everything.
- `docs/adr/` — one-decision-at-a-time rationale, including the two (0011, 0012) added
  alongside this file.
- `docs/deployment.md` — the ordered runbook.
- Each `shop_*` repo's own `README.md` — service-specific detail this file doesn't
  duplicate.
