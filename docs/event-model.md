# Event Model

## Two envelope shapes, on purpose

**`RawEnvelope`** (`shop_platform/event`) wraps a source system's native payload,
unmodified, immediately at ingestion:

```json
{
  "event_id": "uuid",
  "tenant_id": "devmoto",
  "source": "shopify",
  "event_type": "orders/create",
  "shop_domain": "example.myshopify.com",
  "received_at": "2026-09-11T19:00:00Z",
  "schema_version": 1,
  "payload": { "...": "raw Shopify JSON, unmodified" }
}
```

**`CommerceEvent`** is the canonical, source-agnostic event produced by normalizing a
`RawEnvelope`. Downstream consumers (analytics, notifier, future integrations) depend on
this shape, never on Shopify's native schema:

```json
{
  "id": "uuid",
  "tenant_id": "devmoto",
  "type": "commerce.order.created",
  "source": "shopify",
  "timestamp": "2026-09-11T15:31:12Z",
  "version": 1,
  "attributes": {
    "shopify_order_id": "gid://shopify/Order/123",
    "channel": "collective",
    "collective_partner_name": "ABC Bikes",
    "currency": "USD",
    "gross_sales": "699.00",
    "net_sales": "649.00"
  }
}
```

Why two shapes instead of normalizing at the door: the raw payload is the thing you replay
from when normalization logic changes or turns out to be wrong. Losing it at ingestion time
means losing the ability to reprocess history.

## Topics

```
shopify.orders.raw          RawEnvelope, keyed by tenant_id
shopify.orders.normalized   CommerceEvent, keyed by tenant_id
shopify.orders.failed       RawEnvelope + error, for anything normalization couldn't process

analytics.jobs              scheduler -> worker
analytics.results           worker -> notifier / API / dashboard
analytics.jobs.failed

notifications.requested     analytics/other -> notifier
notifications.failed
```

Future topics (not implemented yet, reserved): `shopify.products.raw`,
`shopify.inventory.raw`, `shopify.customers.raw`, `commerce.events` (a merged stream across
all `commerce.*` types once there's more than one).

## Canonical event types

```
commerce.order.created
commerce.order.updated
commerce.order.refunded
inventory.changed        (future)
customer.created         (future)
shipment.created         (future)
shipment.delivered       (future)
ad.spend.recorded        (future)
```

## Money fields — no ambiguous "sales" field

Every money field on `OrderAttributes` is explicit (see `shop_platform/money` for the
integer-cents representation that keeps these exact):

| Field | Definition |
|---|---|
| `gross_sales` | Sum of line item prices before discounts |
| `discounts` | Total discounts applied |
| `returns` | Total refunded amount |
| `net_sales` | `gross_sales - discounts - returns` |
| `shipping` | Shipping charged to the customer |
| `tax` | Tax collected |
| `total_sales` | `net_sales + shipping + tax` — what the customer actually paid |

## Which timestamp is "when" an order happened

The platform's period boundaries (see `shop_platform/period`) use **`created_at`** — when
Shopify created the order — not `processed_at` or a payment-captured timestamp. This is a
deliberate, documented choice: `created_at` is stable and always present, while
`processed_at` can be null for unpaid/pending orders and would make an order silently
disappear from a report until payment clears. Revisit if a future job needs
payment-timeline reporting specifically — that's a different question ("when did we get
paid") than "when did the sale happen."

## Classifying Shopify Collective orders

Confirmed against Shopify's current developer documentation (not assumed):

- Every order placed through Shopify Collective carries a **`"Shopify Collective"` tag**
  in the order's `tags` field — this is the channel signal.
  ([Collective orders guide](https://shopify.dev/docs/apps/build/collective/orders))
- The specific partner is a **second tag**: the retailer's name (from a supplier's
  perspective) or the supplier's name (from a retailer's perspective) — e.g. an order might
  carry `["Shopify Collective", "ABC Bikes"]`.
  ([ERP integration tutorial](https://shopify.dev/docs/apps/build/collective/erp-integration-tutorial))
- Additional context (linked partner order number, shipping cost notes) lives in the
  order's `customAttributes` ("Additional Notes") — captured into `raw_metadata` for
  investigation, not parsed into a typed field yet.
- No separate webhook topic exists for Collective orders — they arrive on the same
  `orders/create`/`orders/updated` webhooks as direct orders. Classification happens
  entirely in the normalizer, from tag content.

Classification logic (`shop_ingestor/internal/normalize`):

1. No `"Shopify Collective"` tag -> `channel = shop`, no partner.
2. `"Shopify Collective"` tag present -> `channel = collective`. Partner name = the one
   remaining tag after excluding `"Shopify Collective"` itself and any Shopify-internal
   (`_`-prefixed) tags.
3. `"Shopify Collective"` tag present but zero or **more than one** candidate partner tag
   remains -> `channel = collective`, `collective_partner_name = nil`. This is a real,
   expected case (a store owner's own manual tags can collide with the heuristic) — it is
   modeled explicitly, not guessed at, and shows up as its own queryable bucket rather than
   silently attributing sales to the wrong partner or dropping them from the Collective
   total.

`OrderAttributes.ClassificationVersion` is stamped on every event so that when this
heuristic improves, historical events tagged with an older version can be identified and
reprocessed from the S3 raw archive — see [architecture.md](architecture.md)'s data
ownership section.

## Idempotency

- **Ingestion**: idempotency key = Shopify's webhook delivery ID (`X-Shopify-Webhook-Id`)
  combined with tenant ID. Duplicate deliveries (Shopify explicitly does not guarantee
  exactly-once delivery) are detected and dropped before publishing.
- **Normalization**: upserts keyed on `(tenant_id, shopify_order_id)` — reprocessing the
  same order is always safe.
- **Redpanda production**: idempotent producer (see `shop_platform/redpanda`) prevents
  producer-to-broker duplication on retry; this is not the same guarantee as end-to-end
  exactly-once, which idempotency keys above provide.

## Error handling

Failed normalization publishes to `shopify.orders.failed` (raw envelope + error message)
rather than silently dropping the event. These aren't automatically retried — see
`shop_docs`'s deployment runbook for how they're inspected and replayed.
