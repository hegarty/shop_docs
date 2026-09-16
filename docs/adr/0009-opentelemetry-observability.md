# ADR-0009: OpenTelemetry for observability instrumentation

## Status
Accepted

## Context
The platform needs metrics, logs, and traces correlated across service boundaries — a
webhook received in `shop_ingestor` should be traceable through normalization, into
`shop_analytics`, and out through `shop_notifier`, all keyed by the same trace ID.

## Decision
OpenTelemetry SDKs in every Go service (`shop_platform/otelx`), exporting via OTLP/gRPC to
an in-cluster OTel Collector, which fans out to Prometheus (metrics), Tempo (traces), and
Loki (logs) — all self-hosted, not AWS-managed (see cost-model.md). Trace context
propagates through Redpanda record headers (`shop_platform/redpanda`), so a trace survives
crossing the event bus, not just synchronous HTTP calls.

## Alternatives considered
- **Amazon CloudWatch + X-Ray**: AWS-proprietary, and meaningfully more expensive at any
  real log/metric volume than self-hosting on a cluster that already exists.
- **Vendor-specific SDKs (e.g. a specific APM vendor's agent)**: locks instrumentation to
  one backend. OpenTelemetry's vendor-neutral SDK means the backend (currently
  Prometheus/Tempo/Loki) can change without touching application code.

## Consequences
Every service pays a small dependency and setup cost (`otelx.Bootstrap` at startup) for
trace/metric export that works uniformly across all three. Structured logs
(`shop_platform/logging`) automatically attach `trace_id`/`span_id` when a context carries
an active span, so log-to-trace correlation in Grafana works without manual wiring at each
call site.
