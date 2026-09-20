# ADR-0012: Treat Shopify webhooks as change notifications, not payload sources

## Status
Accepted

## Context
`shop_ingestor`'s normalization logic (channel/Collective-partner classification, money-
field derivation — see `event-model.md`) was built and tested against Shopify's **GraphQL
Admin API** schema, since that's also what reconciliation needs to query. Shopify webhook
deliveries, however, use a different payload shape (REST-style, different field names —
confirmed against Shopify's current webhook documentation, not assumed) than the GraphQL
`Order` object. Maintaining two parallel derivations of the same gross/discount/refund
math against two different schemas — one exercised only by live webhook traffic, one by
reconciliation — would mean the webhook path's correctness is only as good as its own,
separately-maintained test suite.

## Decision
`shop_ingestor`'s webhook consumer does not parse the webhook body for order data at all,
beyond extracting `admin_graphql_api_id` to know *which* order changed. It then re-fetches
that order's current state via the same GraphQL query reconciliation uses, and runs it
through the one normalization path (`normalize.FromShopifyOrder`) that's actually tested
against Shopify's real schema. A webhook, in this design, means "something changed on this
order" — not "here is the order's data."

## Alternatives considered
- **Parse the REST webhook payload directly**: rejected — would require a second,
  independently-tested normalization function mapping REST field names (`subtotal_price`,
  `total_discounts`, etc.) to the same canonical `OrderAttributes`, doubling the surface
  area for the exact kind of subtle mapping bug this system's tests are built to catch (see
  `shop_ingestor/internal/normalize/normalize_test.go`'s worked examples).
- **Switch reconciliation to REST instead**: rejected — Shopify's GraphQL Admin API is the
  currently-recommended path for bulk/paginated queries, and the REST Admin API is the
  legacy surface, not the newer one.

## Consequences
Every webhook delivery costs one extra GraphQL round-trip to Shopify before an order is
normalized — not free, but each store's transaction volume at MVP scale doesn't make this
a meaningful cost or latency concern. In exchange, there is exactly one place
(`normalize.FromShopifyOrder`) where "what does this order's data mean" is decided,
covered by one test suite, used identically by both the real-time webhook path and the
reconciliation repair path. A `commerce.order.created` vs. `commerce.order.updated`
canonical event type is still derived from the webhook's topic header
(`X-Shopify-Topic`) — see `shop_ingestor/internal/normalize/consumer.go`'s
`commerceEventType` — since re-fetched current state can't distinguish "was just created"
from "was just updated" on its own.
