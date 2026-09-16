# ADR-0010: Terraform module semantic versioning

## Status
Accepted

## Context
`hegarty/terraform` is a monorepo holding multiple independent, reusable modules,
previously referenced by Terragrunt consumers without any version pinning (an unpinned
git source, or a local-path override for development). A change to one module had no way
to avoid implicitly affecting every consumer of every module in the repo.

## Decision
Tag each module independently as `<module-path>/vMAJOR.MINOR.PATCH`. Terragrunt consumers
pin an exact tag via `?ref=`. A `make release MODULE=... VERSION=...` target validates the
module and prints (never silently runs) the exact tag/push commands. See
`terraform-versioning.md` for the full mechanics.

## Alternatives considered
- **One version number for the whole repo**: rejected — a fix to `rds-postgres` would
  force every `eks/*` consumer to bump their pinned ref too, for no reason related to their
  own module.
- **Splitting into one repo per module**: rejected as excessive fragmentation for a
  personal/small-team monorepo — module-scoped git tags achieve independent versioning
  without the overhead of dozens of repositories.

## Consequences
Upgrading `eks/karpenter` to a new version never implies upgrading `rds-postgres`, and
vice versa. Every module not yet tagged as of this decision is bootstrapped at `v1.0.0` to
establish "this is what's live today," not a claim about the module's design maturity.
