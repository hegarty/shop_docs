# ADR-0011: GitHub repository governance via policy-as-code, not manual settings

## Status
Accepted

## Context
Every `shop_*` repository is public from creation (see the platform's security posture —
assume all code may eventually be public). Public repos need consistent branch protection,
secret scanning, and required checks from their first commit, applied the same way across
five (and growing) repositories, without hand-clicking GitHub settings per repo.

## Decision
Repository governance is applied and continuously verified by
[`github_policy-as-code`](https://github.com/hegarty/github_policy-as-code)'s
`audit → plan → apply → verify` cycle, driven by a policy file (a local override of that
tool's own `security-policy.yaml`, not committed to any `shop_*` repo) rather than by hand
in the GitHub UI. Key policy choices for this platform specifically:

- **Ruleset enforcement: `active`, not the tool's own default of `evaluate`.** Confirmed via
  a live `422` that `evaluate` (dry-run) enforcement requires GitHub Enterprise, which this
  personal account is not on. `active` is enforced from the first commit; the retained
  `repository_admin` bypass actor is the safety net instead of a dry-run period.
- **Only `trufflehog` and `build` are required status checks.** `lint` (golangci-lint),
  `govulncheck`, `docker`, and `CodeQL` all run on every PR and are visible in the check
  list, but are deliberately *not* required — a failing lint run or an unrelated Docker
  build issue should not block a secret-scan-clean, compiling, tested change from merging.
  Their results are still meant to be looked at, just not gates.
- **0 required approving reviews** (solo-maintainer policy) — there is exactly one
  maintainer; a mandatory self-approval step would be theater, not review.
- **GitHub-verified signed commits required** on every repo, from the first commit.

## Alternatives considered
- **Manual branch protection per repo via the GitHub UI**: rejected — doesn't scale past a
  couple of repos without drift, and isn't itself auditable or reviewable the way a policy
  file + `plan` diff is.
- **Requiring `lint`/`docker`/`govulncheck` as blocking checks**: rejected for the MVP.
  `docker` in particular failed for real (see below) for days without blocking any merge,
  which is the intended behavior — a broken non-required job is visible and fixable on its
  own PR, not a platform-wide merge freeze.

## Consequences
A real gap surfaced under this policy on 2026-09-20: `docker` had been failing on
`shop_ingestor`/`shop_analytics` since Go was bumped to 1.25 (their Dockerfiles' base image
was still pinned to `golang:1.24`), and `lint` had been silently non-functional platform-
wide since the same bump (the pinned `golangci-lint-action` version installed a binary too
old to parse a `go 1.25` `go.mod`). Neither blocked any of that period's merges — which is
this policy working as designed, not a failure of it — but it meant real, visible-in-CI
breakage went unfixed for days simply because nothing forced attention to it. Non-required
checks still need someone to actually look at them periodically; this policy trades
merge-blocking safety for velocity, not for the checks becoming unnecessary. See
`SESSION.md` for when/how that gap was found and closed.
