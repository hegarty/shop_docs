# Observability

Self-hosted, not AWS-managed (no AMP/AMG) — a deliberate cost decision, see
[cost-model.md](cost-model.md). Everything below runs inside `eks-prod`.

```
Applications (shop_ingestor, shop_analytics, shop_notifier)
    | OTLP/gRPC (shop_platform/otelx)
    v
OTel Collector  ---- metrics ----> Prometheus (kube-prometheus-stack)
                ---- traces  ----> Tempo
                ---- logs    ----> Loki

Grafana <- Prometheus, Loki, Tempo
Cilium -> Hubble (network flow visibility)
```

## What every service exposes

Via `shop_platform/otelx` and `shop_platform/logging`:

- **Traces**: one span per HTTP request (ingestor), per Redpanda publish/consume
  (`shop_platform/redpanda`), per analytics job execution, per notification delivery
  attempt. Trace context propagates through Redpanda record headers, so a trace started at
  webhook receipt continues through normalization, analytics, and notification.
- **Logs**: structured JSON, `trace_id`/`span_id` attached automatically to any log line
  emitted with a context carrying an active span. `tenant_id` and `job_id`/`event_id`
  attached explicitly by each service at the call site (not automatic, since they aren't
  derivable from the span alone).
- **Metrics** (service-specific, expected but not yet all instrumented as of this writing):
  webhook accept/reject counts, HMAC verification failures, Redpanda publish latency,
  consumer lag, reconciliation run counts/failures, analytics job duration/failures,
  notification delivery success/failure, DB query latency.

## Retention (intentionally short)

| Component | Retention | Storage |
|---|---|---|
| Prometheus | 3 days | 10Gi EBS (gp3) |
| Loki | 72 hours | 10Gi EBS, filesystem chunks (not S3-backed yet) |
| Tempo | 24 hours | 10Gi EBS, local backend |
| Hubble | flow visibility only, not persisted long-term | — |

This is short by design for a single-tenant MVP with low request volume. Move Loki/Tempo to
S3-backed storage with longer retention before this becomes a real operational
bottleneck — not preemptively.

## Dashboards

Grafana ships via `kube-prometheus-stack`'s bundled release (`eks-prod`'s
`us-east-1/eks/addons/kube_prometheus_stack`). Planned dashboards (not all built as of this
writing):

1. Platform overview
2. Shopify ingestion (webhook rate/failures, HMAC failures)
3. Redpanda health (lag, throughput)
4. Analytics workers (job duration, failures)
5. PostgreSQL health (connections, query latency)
6. Notifications (delivery success/failure)
7. Cilium/Hubble network visibility (denied flows)
8. Business dashboard: sales today, shop vs. Collective split, partner breakdown, orders,
   AOV — sourced from the application/analytics-results data, not a re-implementation of
   the analytics layer in PromQL.

Grafana's admin credential is never in Helm values or Git — provisioned as a Kubernetes
Secret out-of-band (see `deployment.md`), referenced via `admin.existingSecret` in the
Helm release.

## Health signals

```
Shopify webhook accepted / rejected
Redpanda publish success / failure
Consumer lag
Last successful reconciliation run
Last successful analytics run per (tenant, job)
Notification delivery result
Database connectivity
```

Kubernetes readiness/liveness probes exist on every service; none poll anything expensive
enough to generate its own load (e.g. a readiness probe never queries Postgres with a real
query — it checks the pool can acquire a connection).
