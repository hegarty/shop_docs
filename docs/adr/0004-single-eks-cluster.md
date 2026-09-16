# ADR-0004: Single, shared EKS cluster

## Status
Accepted

## Context
commerce-intel needs Kubernetes. The account also already has a personal-lab cluster
(`eks-dev`) with a default VPC CNI, running unrelated experiments. The person operating
this platform also expects to run other, unrelated projects on whatever cluster gets built
for this one.

## Decision
One new, dedicated EKS cluster (`eks-prod`, its own AWS account, its own VPC), separate
from `eks-dev`, but explicitly **not** scoped to commerce-intel alone — Cilium/Karpenter/
observability are installed once at the cluster level, and each project (starting with
commerce-intel) gets its own `workloads/<project>/` directory in the `eks-prod` Terragrunt
repo for its project-specific AWS resources (RDS, S3, ECR, secrets, Pod Identity roles).

## Alternatives considered
- **Reuse `eks-dev`**: would require migrating its CNI to Cilium on a live cluster
  (disruptive, risky) and would mix a "serious" workload's blast radius with an
  experimental lab cluster's lifecycle. Rejected.
- **A dedicated cluster scoped only to commerce-intel**: simpler to reason about in
  isolation, but means the ~$73/mo control-plane cost and NAT Gateway cost are commerce-
  intel's alone, and a second unrelated project later would either share this cluster
  anyway (making the "scoped" naming misleading) or pay for a third cluster's fixed costs.
  Rejected in favor of designing for shared use from the start.
- **Namespace-per-project on one cluster with no `workloads/` separation in code**: works
  operationally, but blurs "cluster platform" and "project-specific" infrastructure in the
  same Terragrunt directories, making it harder to reason about what a new project actually
  needs to add. Rejected in favor of the explicit `workloads/` split.

## Consequences
The cluster's fixed costs are honestly attributed as shared infrastructure, not charged
entirely to commerce-intel (see cost-model.md). Adding a second, unrelated project later
means a new `workloads/<project>/` directory and its own Pod Identity roles/namespace —
not a new cluster, new Cilium install, or new observability stack.
