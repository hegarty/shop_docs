# Deployment Runbook

Every command below that creates or modifies real AWS/GitHub/Kubernetes state is meant to
be run **by you**, reviewed before running. Nothing here is automated end-to-end on
purpose.

**Status**: infrastructure sequence below is accurate as of this writing. The application
deployment section (Helm charts / manifests for `shop_ingestor`/`shop_analytics`/
`shop_notifier`) is not yet implemented — this doc will be updated once those exist.

## 1. Bootstrap `eks-prod`'s remote state (one-time)

Neither the state bucket nor lock table exist yet, in `eks-prod`'s own AWS account
(868150784168 — separate from `eks-dev`'s account). See `eks-prod/README.md` for the exact
`aws s3api`/`aws dynamodb` commands. Verify both exist before anything else.

## 2. Tag terraform modules

```bash
cd hegarty/terraform
make release MODULE=<path> VERSION=1.0.0   # for every module eks-prod references
```

Review and confirm each tag/push individually — see `terraform-versioning.md`.

## 3. Terragrunt apply order (`eks-prod`)

Terragrunt resolves most of this automatically via `dependency` blocks, but Helm-based
addons need the cluster and a schedulable node to exist first. Rough order:

```
networking/vpc
networking/internet_gateway
networking/nat
networking/routes/{controller,public,worker}
eks/security_groups
eks/iam/{cluster_role,node_role}
eks/cluster
eks/access_entries/{nodes,users}        <- fix the placeholder admin ARN first (see eks-prod/README.md)
eks/system_node_group                    <- must exist before any Helm release below
eks/storage_class
eks/karpenter                            <- IAM/SQS
eks/addons/karpenter                     <- the actual Helm release
    kubectl apply -f eks-prod/manifests/karpenter/   <- NodePool/EC2NodeClass, once the CRDs exist
eks/addons/cilium
eks/addons/cert_manager
eks/addons/redpanda
eks/addons/kube_prometheus_stack
eks/addons/loki
eks/addons/tempo
eks/addons/otel_collector

workloads/commerce-intel/rds
workloads/commerce-intel/s3/raw_archive
workloads/commerce-intel/ecr/*
workloads/commerce-intel/secrets/*       <- then populate Shopify tokens + SMS key by hand, out-of-band
workloads/commerce-intel/pod_identity/*
workloads/commerce-intel/budgets
```

Run each with `terragrunt plan` first, review, then `terragrunt apply` — never
`run-all apply` blind on a first deployment.

## 4. Populate secrets out-of-band

After step 3's `secrets/*` units create empty Secrets Manager shells:

```bash
aws secretsmanager put-secret-value \
  --secret-id commerce-intel/shopify/devmoto \
  --secret-string '{"admin_api_token":"...","webhook_signing_secret":"..."}'

aws secretsmanager put-secret-value \
  --secret-id commerce-intel/sms-provider \
  --secret-string '{"api_key":"..."}'
```

Also create the Grafana admin credential Secret referenced by
`kube_prometheus_stack`'s `grafana.admin.existingSecret`:

```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace observability \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 24)"
```

## 5. Deploy the application services

Not yet implemented — Helm charts/manifests for `shop_ingestor`, `shop_analytics`,
`shop_notifier` are pending. This section will list `helm upgrade --install` (or
equivalent) commands once they exist, plus how images get from each repo's CI into ECR.

## 6. Register the Shopify webhook

Only after `shop_ingestor` is actually running and reachable through the Gateway API's
NLB. Do not create this against production Shopify until the receiver is deployed and
verified healthy:

```bash
# exact command TBD — depends on final ingestor endpoint path and Shopify API version in use
```

## 7. Verify

```bash
kubectl get pods -A
kubectl logs -n commerce-intel deploy/shop-ingestor
# confirm Grafana dashboards show data after a test order
```
