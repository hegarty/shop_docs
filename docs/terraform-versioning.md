# Terraform Module Versioning

`hegarty/terraform` is a monorepo of independent, reusable modules. Each is released and
versioned independently — upgrading one module must never imply upgrading any other.

## Tagging convention

```
<module-path>/vMAJOR.MINOR.PATCH
```

Examples: `eks/cluster/v1.0.0`, `rds-postgres/v1.2.0`, `eks/karpenter/v1.0.0`.

Terragrunt consumers (in `eks/` and `eks-prod/`) pin an exact tag via a `?ref=` query
parameter on the module source — never `?ref=main` or an unpinned source:

```hcl
locals {
  module_path    = "eks/karpenter"
  module_version = "v1.0.0"
}

terraform {
  source = format(include.root.locals.module_source, "${local.module_path}?ref=${local.module_path}/${local.module_version}")
}
```

`include.root.locals.module_source` resolves to either a local checkout (`$MODULE_SOURCE`
env var, for local module development/iteration — the `?ref=` suffix is meaningless
against a local path and is simply ignored by Terragrunt's source resolution in practice)
or the real git source (`git::ssh://git@github.com/hegarty/terraform.git//%s`) when
`$MODULE_SOURCE` is unset.

## Releasing a module

```bash
make release MODULE=eks/karpenter VERSION=1.0.0
```

This, in `hegarty/terraform`'s `Makefile`:

1. Runs `terraform fmt -check`, `terraform init -backend=false`, `terraform validate`
   against the module directory.
2. Refuses if the tag already exists, or if the module directory has uncommitted changes.
3. **Prints** the exact `git tag -a` / `git push origin` commands and asks for
   `y/N` confirmation.
4. Only tags and pushes on an explicit `y` — never automatically, never as a side effect of
   any other command.

## Breaking changes

Bump MAJOR for anything that changes a required input's meaning, removes an output, or
changes a resource in a way that would force replacement on existing infrastructure using
that module version. MINOR/PATCH for additive or backward-compatible changes (a new
optional variable with a default, a bug fix that doesn't change the module's contract).

## Transitioning existing (unversioned) modules

Every module that predates this convention is bootstrapped at `v1.0.0`, tagging its
current `main` state as "this is what's live today" — not a claim that the module was
designed from scratch at v1. Future changes to that module bump its tag independently from
that baseline, same as any other module.

## Known limitation

The local-development `$MODULE_SOURCE` override and `?ref=` version pinning don't compose:
during local module iteration (pointing at an uncommitted local checkout), the `?ref=`
suffix is inert since there's no git ref to resolve against a plain filesystem path. This
is fine in practice — local iteration inherently means "use whatever's on disk right now,"
not a specific tag — but is worth knowing if a `terragrunt init` behaves unexpectedly while
`$MODULE_SOURCE` is set.
