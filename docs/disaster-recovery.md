# Disaster Recovery

## Data ownership, and what's actually irreplaceable

```
Shopify      — authoritative upstream commerce source. Always recoverable from here via
               the Admin API, as long as the order still exists on Shopify's side.
S3           — durable raw event archive. The practical replay source of truth — cheaper
               and faster to replay from than re-fetching everything from Shopify's API.
PostgreSQL   — normalized current state + analytics state. Rebuildable from S3 + Shopify.
Redpanda     — transport only. NOT a source of truth. Losing it loses in-flight messages,
               not history.
```

Nothing in this platform's design treats Redpanda as durable. If the single broker is
lost, the recovery path is: redeploy Redpanda, then re-run reconciliation (which polls
Shopify's Admin API for anything created/updated since the last successful sync, with an
overlap window) to refill it and Postgres. No data is permanently lost purely from a
Redpanda outage, because ingestion doesn't only happen at webhook-delivery time.

## Failure modes and recovery

| Failure | Impact | Recovery |
|---|---|---|
| Redpanda broker lost | Ingestion/analytics pipeline stalls; no data loss | Redeploy broker (StatefulSet + PV), run reconciliation to catch up |
| RDS instance lost (no snapshot) | Normalized data lost; raw archive intact | Restore from automated backup (7-day retention); worst case, replay from S3 + Shopify from scratch |
| Single NAT Gateway's AZ has an outage | All private-subnet egress breaks (both AZs) until the AZ recovers | Accepted MVP tradeoff — see cost-model.md. No automated failover; this is the explicit cost/availability tradeoff of one NAT Gateway instead of one per AZ |
| Shopify webhook delivery missed (Shopify-side issue, app downtime) | Order missing until next reconciliation cycle | Reconciliation (`updated_at > last_successful_sync - overlap_window`) catches it on its next run |
| Normalization logic bug produces wrong `channel`/`collective_partner` | Historical orders misclassified | `classification_version` on each row identifies which orders used the buggy logic; reprocess from S3 raw archive after fixing the classifier |
| EKS cluster itself lost | Full outage | Terragrunt/Terraform re-apply from `eks-prod` recreates the cluster; RDS/S3/Secrets Manager are independent of the cluster's lifecycle and survive a cluster recreation |

## Backups

- **RDS**: automated backups, 7-day retention (`backup_retention_period` in
  `rds-postgres` module), `deletion_protection = true`, `skip_final_snapshot = false` — a
  final snapshot is always taken before any intentional deletion.
- **S3 raw archive**: versioning enabled. Lifecycle transitions to Infrequent Access at 30
  days and Glacier at 90 days — not deleted, just tiered down in cost.
- **Terraform state**: stored in S3 (versioned bucket) with DynamoDB locking. State itself
  is backed up by S3 versioning; losing the state file doesn't lose AWS resources, just the
  ability to manage them until state is reconstructed or imported.

## What is NOT backed up (by design)

Redpanda's on-disk log is not backed up — see above for why that's an accepted, documented
gap rather than an oversight.
