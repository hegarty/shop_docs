# Local Development

## Prerequisites

- Go 1.25.x (each repo pins an exact patch version in `.tool-versions`; `asdf install` picks
  it up — bumped from 1.23.8 after `github.com/jackc/pgx/v5` shipped a security fix that
  required Go ≥ 1.25)
- Docker (for local Postgres + Redpanda)
- `gh` CLI authenticated, if you need to interact with GitHub from the command line

## Running dependencies locally

Each service repo's own README documents its specific `docker-compose.yml`. In general:

```bash
docker run -d --name shop-postgres \
  -e POSTGRES_PASSWORD=localdev -e POSTGRES_DB=commerce_intel \
  -p 5432:5432 postgres:16

docker run -d --name shop-redpanda \
  -p 9092:9092 \
  docker.redpanda.com/redpandadata/redpanda:latest \
  redpanda start --smp 1 --overprovisioned --node-id 0 \
  --kafka-addr PLAINTEXT://0.0.0.0:9092 \
  --advertise-kafka-addr PLAINTEXT://127.0.0.1:9092
```

## Environment variables

Every service validates its configuration at startup via `shop_platform/config` and fails
immediately, listing every missing/invalid variable at once, rather than failing on first
use of whichever variable happens to be read first. Common variables across services:

```
AWS_REGION
REDPANDA_BROKERS          # e.g. "localhost:9092" locally
DATABASE_HOST
DATABASE_PORT
DATABASE_NAME
DATABASE_USER
# DATABASE_PASSWORD is never an env var in a deployed environment — read
# from Secrets Manager. Locally, set it directly for convenience.
OTEL_EXPORTER_OTLP_ENDPOINT   # e.g. "localhost:4317" if running a local collector, or omit
LOG_LEVEL                 # debug | info | warn | error
```

Service-specific variables (Shopify credentials, SMS provider config, tenant ID) are
documented in each service's own README.

## Migrations

Each service that owns schema (`shop_ingestor` for the core commerce tables,
`shop_analytics` for job/results tables) ships its own `migrations/*.sql` and a
`cmd/migrate` binary built on `shop_platform/db`:

```bash
make migrate-up
make migrate-down
```

Never run these against a production database from a local machine — they're for local
and CI/test-database use. Production migrations happen as part of the deployment runbook
(see `deployment.md`), using credentials pulled from Secrets Manager, not a developer's
local environment.

## Tests

```bash
make test    # go test ./... -race -count=1
make lint    # golangci-lint run
```

Tests that need a real Postgres/Redpanda are marked accordingly in each repo (see that
repo's README) and skip automatically when the dependency isn't reachable, so `make test`
works without Docker running for the majority of the test suite (pure logic — `period`,
`money`, `event`, classification — needs no external dependency at all).
