# Cost Model

`eks-prod` is a **shared** cluster — built once, intended to host unrelated future
projects alongside commerce-intel, so its largest fixed costs (EKS control plane, NAT
Gateway) amortize across more than one project over time. The breakdown below separates
"the whole cluster's run-rate" from "commerce-intel's own incremental cost" for that
reason — collapsing them into one number would misrepresent what commerce-intel itself
actually costs.

## Straight talk

The EKS control plane (~$73/mo) and a single NAT Gateway (~$33/mo) are **not** discountable
by Reserved Instances or a Compute Savings Plan — those only discount EC2/Fargate
compute-hours. $106/mo of fixed cost exists before a single application pod runs. There is
no way to run a real EKS cluster with external ingress for meaningfully under $100/mo
total; getting there would mean dropping the NAT Gateway or the load balancer, both of
which this platform's security posture depends on.

## Breakdown

| Item | Monthly (USD) | Notes |
|---|---|---|
| EKS control plane | $73 | Fixed. Shared across commerce-intel + future projects |
| NAT Gateway (single) | ~$33 + ~$1-5 data | Single-AZ egress — see ADR and disaster-recovery.md for the tradeoff |
| System node group (1 static node) | ~$8-12 | Hosts Karpenter, Cilium operator, CoreDNS, cert-manager before Karpenter can run itself |
| Karpenter-managed workload nodes | ~$10-25 | Spot-first with on-demand fallback, consolidation on; a 1-yr no-upfront Compute Savings Plan (~28-31% off) further reduces this — a billing commitment, purchased by hand, never automated |
| EBS (Redpanda PV + node roots) | ~$5-10 | gp3, small volumes |
| RDS PostgreSQL (db.t4g.micro, single-AZ, 20GB gp3) | ~$13-15 | |
| S3 raw archive | ~$1-3 | Lifecycle: Standard -> IA at 30 days -> Glacier at 90 days |
| ECR (3 repos, lifecycle policy) | <$1 | Keeps last 10 tagged images, expires untagged after 7 days |
| Secrets Manager (~5 secrets) | ~$2 | |
| CloudWatch (reduced log types: `api`, `audit` only) | ~$2-5 | |
| NLB (Cilium Gateway API, Shopify webhook ingress) | ~$16-18 | |
| **Total shared-cluster run-rate** | **~$165-190/mo** | |
| **commerce-intel's incremental share** (excludes control plane + NAT, which exist regardless of this project) | **~$45-65/mo** | |

## Where the MVP deliberately under-spends (and the upgrade trigger)

| Choice | Saves | Revisit when |
|---|---|---|
| Single Redpanda broker | Avoids a 3-broker cluster | A tenant can't tolerate ingestion downtime |
| Single-AZ RDS | Avoids Multi-AZ ~2x cost | Real uptime SLA needed, or a second tenant onboards |
| Single NAT Gateway | ~$33/mo vs ~$65+/mo for 2 AZs | The shared cluster's other workloads need multi-AZ egress resilience |
| AWS-managed KMS keys, no CMKs | Each CMK is a fixed monthly cost | A specific compliance requirement demands customer-controlled key rotation |
| Self-hosted observability, short retention | Avoids AMP/AMG + long retention storage | Query volume/retention needs outgrow what self-hosting comfortably handles |

## Budget alerts

`hegarty/terraform`'s `budgets` module (instantiated in `eks-prod/us-east-1/workloads/commerce-intel/budgets`)
sends email alerts at $100/$150/$200/$250 actual monthly spend, against a $250 overall
ceiling.
