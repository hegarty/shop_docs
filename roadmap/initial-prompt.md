# Initial Prompt

The original brief this platform was commissioned from, reproduced verbatim (2026-09-14).
Every ADR, module, and repo boundary in this platform traces back to a constraint stated
here — when a decision looks arbitrary, check here before assuming it needs revisiting.

See [`milestones.md`](milestones.md) for how this brief was broken into trackable work, and
[`../SESSION.md`](../SESSION.md) for current status against it.

---

You are helping me design and implement the initial version of a serious e-commerce analytics and monitoring platform.

This is NOT intended to be a disposable prototype. The first use case is deliberately narrow, but I intend for this platform to evolve into a commercial SaaS product and potentially an agentic e-commerce intelligence framework.

The initial customer is my son's Shopify business, but the architecture must be designed so additional Shopify stores, integrations, tenants, analytics jobs, notification channels, and eventually AI agents can be added cleanly.

I am an experienced infrastructure/DevOps/cloud engineer and will personally review and perform all deployments. You should produce the code, infrastructure definitions, documentation, tests, repository structures, commands, and implementation artifacts. DO NOT deploy infrastructure or make production changes on my behalf.

## PRIMARY INITIAL BUSINESS REQUIREMENT

The first analytics problem we want to solve is:

For a configurable date range, determine sales grouped by:

1. Direct Shopify/shop sales
2. Shopify Collective sales
3. Within Shopify Collective, further group sales by Collective retailer/partner

The system must support these reporting periods initially:

* Daily
* Weekly
* Monthly
* Quarterly

It should be easy to add:

* Today
* Yesterday
* Week-to-date
* Previous week
* Month-to-date
* Previous month
* Quarter-to-date
* Previous quarter
* Year-to-date
* Arbitrary start/end dates

The initial output can be delivered as a text/SMS-style business summary.

Example:

```
DevMoto Daily — Sep 11

Total Sales: $10,254

Shop:
$6,842
66.7%

Collective:
$3,412
33.3%

Collective Breakdown:
E-Moto Co        $1,282
Ride Electric      $934
XYZ Motors         $721
ABC Bikes          $475

Orders: 45
AOV: $227.87
```

Eventually this same analytics result should be consumable by:

* SMS
* Web dashboard
* Email
* Slack
* APIs
* AI agents
* Other downstream services

Do not tightly couple analytics jobs to any notification mechanism.

---

## IMPORTANT PRODUCT VISION

Design this as an event-driven e-commerce intelligence platform.

The long-term progression is:

Phase 1:
"What happened?"

Phase 2:
"Why did it happen?"

Phase 3:
"What should I do?"

Phase 4:
"Do it for me."

Eventually I expect integrations beyond Shopify, including systems such as:

* QuickBooks
* Stripe
* Meta Ads
* Google Ads
* Klaviyo
* ShipStation
* Amazon
* TikTok
* GA4
* Other commerce platforms

Therefore Shopify-specific objects should be normalized into canonical platform events wherever practical.

For example:

```json
{
  "id": "0199...",
  "tenant_id": "devmoto",
  "type": "commerce.order.created",
  "source": "shopify",
  "timestamp": "2026-09-11T15:31:12Z",
  "version": 1,
  "attributes": {
    "order_id": "...",
    "channel": "collective",
    "partner": "ABC Bikes",
    "currency": "USD",
    "gross_sales": 699.00,
    "net_sales": 649.00
  }
}
```

Downstream consumers should eventually care about:

* commerce.order.created
* commerce.order.updated
* commerce.order.refunded
* inventory.changed
* customer.created
* shipment.created
* shipment.delivered
* ad.spend.recorded

rather than Shopify's native schema.

---

## LANGUAGE AND APPLICATION PLATFORM

Use:

* Golang for application services
* Kubernetes
* AWS EKS
* Cilium
* Cilium Gateway API
* Hubble
* Redpanda
* PostgreSQL
* Amazon S3
* Terraform
* Terragrunt
* Helm where appropriate

I want the platform to remain as cloud-portable as reasonably possible even though the initial deployment is AWS.

Avoid unnecessary AWS-proprietary application dependencies when an open-source/cloud-neutral abstraction makes sense.

---

## COST IS A FIRST-CLASS ARCHITECTURAL REQUIREMENT

AWS cost is extremely important.

Treat cost optimization as a primary non-functional requirement.

I want this platform to be technically serious without creating an enterprise-sized AWS bill before it has customers.

Target the initial environment toward approximately:

$150-$250/month if reasonably achievable.

Do NOT blindly implement standard highly available enterprise patterns for the MVP.

For example, do NOT automatically create:

* Three EKS clusters
* Dev/stage/prod clusters
* Three NAT Gateways
* Three Redpanda brokers
* Multi-AZ RDS
* Large managed observability stacks
* Unnecessary ALBs
* Excessive interface VPC endpoints
* Oversized nodes
* Large retention periods

Prefer:

* One EKS cluster
* Small worker footprint
* Single-AZ RDS PostgreSQL initially
* Single-node Redpanda initially
* Small EBS volumes
* S3 for inexpensive durable/archive storage
* Sensible autoscaling
* Self-hosted observability initially
* One external load balancer where possible
* Cilium Gateway API
* Explicit egress architecture
* Tight telemetry retention

However:

Cost optimization must NOT result in poor security practices, hard-coded credentials, irreversible data loss, or an architecture that cannot evolve.

Document the compromises being made for the MVP and what should change before supporting paying customers at larger scale.

---

## AWS REGION

Default the initial environment to:

us-east-1

unless there is a strong technical reason not to.

Make region configurable through Terragrunt.

---

## SECURITY — EXTREMELY IMPORTANT

ALL CODE WILL BE PUBLICLY AVAILABLE ON GITHUB.

Assume every repository may eventually become public.

Therefore:

NEVER commit:

* Shopify access tokens
* Shopify app secrets
* Shopify webhook secrets
* AWS credentials
* Database passwords
* API keys
* SMS provider credentials
* Grafana admin passwords
* Redpanda credentials
* TLS private keys
* OAuth client secrets
* JWT signing keys
* customer data
* personally identifiable information
* production configuration containing sensitive information
* kubeconfigs
* .env files containing secrets
* tfvars files containing secret values
* Terraform state
* Terragrunt cache files

Sensitive values MUST be stored in:

AWS Secrets Manager

Applications running inside EKS should retrieve secrets securely.

Prefer:

* EKS Pod Identity where appropriate
* narrowly scoped IAM roles
* Kubernetes ServiceAccounts tied to workload IAM identity
* AWS Secrets Manager
* optionally External Secrets Operator if justified

Do not copy AWS secrets into Git.

If Kubernetes Secrets are necessary, their authoritative source should still be Secrets Manager.

Do not store secrets in Helm values committed to Git.

Every repository must contain a strong `.gitignore`.

Where appropriate add automated secret detection to CI, preferably using something such as:

* TruffleHog
* Gitleaks

Fail CI if secrets are detected.

Do not log secrets.

Do not log raw authentication headers.

Be mindful that raw Shopify webhook payloads may eventually contain customer PII.

Implement structured logging and provide facilities for redaction.

Encrypt:

* data at rest
* EBS volumes
* RDS
* S3
* Secrets Manager
* transport connections where practical

Use least-privilege IAM.

Do not grant application workloads AdministratorAccess.

---

## GITHUB REPOSITORY STRATEGY

Create individual GitHub repositories for the platform components.

Use a COMMON PREFIX for all repositories so they remain grouped alphabetically in GitHub.

Use this prefix initially:

commerce-intel-

Examples could include:

commerce-intel-shopify-ingestor
commerce-intel-normalizer
commerce-intel-analytics
commerce-intel-notifier
commerce-intel-platform
commerce-intel-docs

Do not blindly create dozens of repositories.

Start with a practical decomposition.

I expect approximately these application responsibilities:

1. Shopify ingestion
2. Event normalization
3. Analytics/scheduling
4. Notification delivery

Some responsibilities may reasonably share repositories initially.

Avoid one repository per analytics job.

Before creating repositories:

1. Propose the repository structure.
2. Explain why each repository exists.
3. Avoid excessive fragmentation.
4. Then create them.

If authenticated GitHub CLI access is available, use `gh` to create the repositories.

If GitHub authentication is not available:

* Do not stop the implementation.
* Produce the exact `gh repo create` commands I should run.

Do not put secrets into GitHub repository configuration.

Provide sensible:

* README.md
* LICENSE recommendations
* CODEOWNERS if appropriate
* CONTRIBUTING.md
* SECURITY.md
* Makefile
* GitHub Actions
* Go linting
* unit tests
* vulnerability scanning
* secret scanning

This platform may eventually be open-source or source-available, so repository hygiene matters.

---

## EXISTING TERRAFORM MODULE REPOSITORY

Generic reusable Terraform modules belong here:

/Users/terencehegarty/projects/hegarty/terraform

IMPORTANT:

Inspect the existing repository before creating anything.

Do not recreate modules that already exist and satisfy the requirement.

If an appropriate module does not exist:

Create a new reusable generic module in this repository.

Do NOT create application-specific Terraform modules when the capability could reasonably be generic.

Potential module areas may include:

* VPC
* EKS
* EKS managed node groups
* EKS Pod Identity/IAM
* RDS PostgreSQL
* S3
* ECR
* Secrets Manager
* Route53
* load balancing
* AWS Budgets
* IAM
* KMS

But inspect the repository first.

Follow the conventions already used in the repository unless they conflict with a major architectural requirement.

---

## TERRAFORM MODULE VERSIONING

The Terraform repository contains multiple reusable modules in a single Git repository.

I want to move this repository to TAG-BASED MODULE VERSIONING.

Do not rely on branch references such as:

?ref=main

or:

?ref=master

Terragrunt should reference immutable version tags.

Because multiple modules share the same repository, use a clear module-specific semantic versioning convention.

Preferred pattern:

vpc/v1.0.0
eks/v1.0.0
rds-postgres/v1.0.0
s3/v1.0.0
secrets-manager/v1.0.0

or another well-supported module-scoped tagging convention if you believe a different approach is materially better.

The important properties are:

* immutable releases
* semantic versioning
* module-specific releases
* one monorepo containing multiple reusable modules
* Terragrunt can pin an individual module version
* upgrading one module does not imply upgrading all others

Document:

* tagging convention
* release workflow
* how to create a release
* how Terragrunt references tags
* how breaking changes are handled
* how existing unversioned modules should transition

If useful, provide scripts or Make targets for releasing modules.

Example desired workflow:

```bash
make release MODULE=eks VERSION=1.2.0
```

which safely produces something equivalent to:

```text
eks/v1.2.0
```

Do not automatically push tags without clearly showing me what will happen.

I will perform final releases/deployments.

---

## TERRAGRUNT STRUCTURE

Terragrunt environments should follow the convention already established under:

/Users/terencehegarty/projects/hegarty/eks

Inspect this directory before creating the new project structure.

Follow the established patterns where appropriate.

Create a logically named hierarchy for this project beneath the existing convention.

For example, something conceptually like:

```text
/Users/terencehegarty/projects/hegarty/eks/
  commerce-intel/
    account.hcl
    region.hcl
    env.hcl

    prod/
      us-east-1/
        vpc/
        eks/
        rds/
        s3/
        secrets/
        observability/
```

Do NOT use this exact structure blindly.

Inspect my existing Terragrunt layout and conform to its conventions.

The important requirement is:

Reusable resources:
`/Users/terencehegarty/projects/hegarty/terraform`

Deployable Terragrunt configuration:
`/Users/terencehegarty/projects/hegarty/eks`

Maintain strong separation between reusable Terraform modules and environment configuration.

Use dependency outputs rather than duplicating resource identifiers.

Prefer DRY Terragrunt configuration.

Use remote Terraform state if already established by my environment.

Never commit Terraform state.

---

## DEPLOYMENTS

I WILL PERFORM ALL DEPLOYMENTS.

Claude should:

* write code
* generate Terraform
* generate Terragrunt
* create Helm values/manifests
* create GitHub repositories where permitted
* create CI
* generate documentation
* generate tests
* generate commands
* perform local static validation where possible
* run unit tests locally where possible
* run `terraform fmt`
* run `terraform validate` where possible
* run `terragrunt hclfmt`
* run Go tests
* run linters

Claude should NOT:

* run `terraform apply`
* run `terragrunt apply`
* deploy Kubernetes resources to a real cluster
* modify production AWS resources
* create production secrets
* create real Shopify webhook subscriptions against production
* change DNS
* alter live infrastructure

Plan/generate/validate only.

If a command could create or modify real infrastructure, show it to me rather than executing it.

---

## INITIAL AWS ARCHITECTURE

Use one EKS cluster.

Conceptually:

```text
AWS
└── us-east-1
    └── VPC
        ├── EKS
        │   ├── Cilium
        │   ├── Hubble
        │   ├── Redpanda
        │   ├── Shopify ingestor
        │   ├── Normalizer
        │   ├── Scheduler
        │   ├── Analytics workers
        │   ├── Notifier
        │   └── Observability stack
        │
        ├── RDS PostgreSQL
        ├── S3
        ├── Secrets Manager
        └── ECR
```

Avoid separate Kubernetes clusters for development/staging/production at this phase unless there is an exceptionally strong reason.

Namespaces can provide initial separation.

Potential namespaces:

```text
platform
redpanda
observability
commerce-intel
```

or a better structure if justified.

---

## CILIUM

Use Cilium for Kubernetes networking/security.

Use Cilium intentionally, not merely as the installed CNI.

I want:

* Cilium NetworkPolicy
* default-deny where reasonable
* explicit service-to-service policy
* Cilium Gateway API
* Hubble
* network observability
* future support for service/network security controls

Example desired communication model:

```text
shopify-ingestor
    -> Redpanda

normalizer
    -> Redpanda
    -> PostgreSQL

analytics-worker
    -> PostgreSQL
    -> Redpanda

notifier
    -> Redpanda
    -> approved external notification APIs

scheduler
    -> Redpanda
    -> PostgreSQL if required
```

Other unnecessary east/west paths should be denied.

Be especially careful about unrestricted pod egress.

Document the Cilium policies.

---

## INGRESS

Prefer:

Cilium Gateway API

rather than deploying a separate ingress controller unless there is a compelling reason.

Shopify webhooks should reach something conceptually like:

```text
Internet
    |
AWS NLB
    |
Cilium Gateway API
    |
shopify-ingestor
```

Keep the number of AWS load balancers small because they have ongoing cost.

---

## REDPANDA

Use Redpanda as the event bus.

Initially, prioritize cost over high availability.

For the MVP, a SINGLE REDPANDA BROKER is acceptable.

Use persistent EBS storage.

Document clearly that this is an MVP availability tradeoff.

Redpanda is NOT the source of truth.

Redpanda should be replaceable/rebuildable from:

* Shopify reconciliation
* archived raw events in S3
* normalized data in PostgreSQL where appropriate

Initial topic concepts:

```text
shopify.orders.raw
shopify.orders.normalized

analytics.jobs
analytics.results

notifications.requested
notifications.delivered
```

Potential future topics:

```text
shopify.products.raw
shopify.inventory.raw
shopify.customers.raw
commerce.events
```

Use sensible:

* partition counts
* retention
* message keys
* schemas
* compression
* resource requests

Do not overprovision.

Tenant ID should generally be part of events.

---

## EVENT ENVELOPE

Do not publish raw Shopify JSON without context.

Use a platform envelope.

Example:

```json
{
  "event_id": "uuid",
  "tenant_id": "devmoto",
  "source": "shopify",
  "event_type": "orders/create",
  "shop_domain": "example.myshopify.com",
  "received_at": "2026-09-11T19:00:00Z",
  "schema_version": 1,
  "payload": {}
}
```

Design this with schema evolution in mind.

Do not unnecessarily expose customer PII to consumers that do not need it.

---

## SHOPIFY INGESTION

Initial Shopify webhook support should include at minimum the order lifecycle needed to accurately report sales.

Likely:

* orders/create
* orders/updated
* refund-related events
* cancellation-related changes where required

Verify the correct current Shopify APIs and webhook topics before implementation.

Prefer Shopify GraphQL Admin API for reconciliation and historical retrieval where appropriate.

The webhook receiver must:

1. accept webhook
2. verify Shopify HMAC
3. identify tenant/store
4. establish idempotency/event identifier
5. attach metadata
6. publish event
7. return HTTP success quickly

Do not perform expensive analytics synchronously in the webhook path.

Handle duplicate webhook delivery safely.

---

## SHOPIFY COLLECTIVE

The initial analytics requirement depends heavily on properly identifying Shopify Collective transactions.

Determine the most reliable current method for distinguishing:

* normal/direct Shopify orders
* Shopify Collective orders
* Collective partner/retailer

Do not hard-code assumptions without checking current Shopify documentation.

Normalize the result to fields resembling:

```text
channel = shop
channel = collective

collective_partner_id
collective_partner_name
```

Support cases where the partner cannot initially be determined.

Preserve raw metadata sufficient to investigate classification errors.

---

## RECONCILIATION

Webhooks must NOT be the only source of ingestion.

Implement a reconciliation process using Shopify APIs.

Conceptually:

```text
webhooks = low latency
Shopify API = authoritative reconciliation
```

Create a scheduled reconciliation process.

For example:

```text
updated_at > last_successful_sync - overlap_window
```

The overlap window protects against timing issues.

The reconciliation process should repair:

* missed webhooks
* duplicate events
* application downtime
* Redpanda downtime
* refunds
* edits
* cancellations
* eventual consistency issues

Also design for deeper periodic reconciliation.

Make reconciliation idempotent.

---

## DATABASE

Use:

Amazon RDS PostgreSQL

Initially:

* Single-AZ
* small instance class
* encrypted
* automated backups
* sensible retention
* Secrets Manager credentials
* private networking

Do NOT introduce DynamoDB, MongoDB, OpenSearch, or ClickHouse unless there is an actual immediate need.

PostgreSQL is both operational storage and the initial analytical database.

Potential future use of ClickHouse can be documented, but do not deploy it now.

---

## DATABASE MODEL

Everything should be multi-tenant from the beginning.

Every relevant business record should have:

tenant_id

Potential starting entities:

```text
tenants
shopify_stores
orders
order_line_items
collective_partners
analytics_jobs
analytics_results
notification_channels
job_schedules
```

Possible orders fields:

```text
id
tenant_id
shopify_order_id
order_number
created_at
updated_at
processed_at

currency

gross_sales
discounts
returns
net_sales
shipping
tax
total_sales

channel
collective_partner_id

financial_status
fulfillment_status

raw_metadata JSONB
```

Do not create a single ambiguous numeric field called only `sales`.

Clearly model:

* gross_sales
* discounts
* returns
* net_sales
* shipping
* tax
* total_sales

Provide explicit metric definitions.

This is important because Shopify, accounting systems, and business users can define "sales" differently.

---

## TENANCY

Even though the first tenant is one business, treat multi-tenancy as foundational.

Every event should include tenant identity.

Every query should be tenant-scoped.

Design toward:

```text
Tenant
├── Shopify stores
├── Users
├── Integrations
├── Analytics jobs
├── Schedules
├── Notification channels
├── Alert rules
└── Future agents
```

Do not unnecessarily implement a complete user identity system yet.

But avoid architectural decisions that would prevent it later.

---

## ANALYTICS EXECUTION MODEL

Do NOT create one microservice per analytics job.

Instead create a generic analytics worker framework.

Conceptual Go interface:

```go
type Job interface {
    Name() string
    Execute(ctx context.Context, tenantID string, period Period) (*Result, error)
}
```

Example jobs:

```text
sales.channel.breakdown
sales.collective.breakdown
sales.daily.summary

future:
inventory.low_stock
orders.unfulfilled
orders.velocity
customer.repeat_rate
profit.margin
product.performance
```

Different analytics should primarily be plugins/modules/jobs within the worker framework rather than separate deployable applications.

Separate ANALYTIC LOGIC from DEPLOYMENT UNITS.

---

## BATCH VS REAL-TIME

Support two models.

REAL-TIME / STREAMING:

Long-running Redpanda consumers.

Examples:

```text
new order
-> update realtime revenue

refund
-> update net revenue

inventory changed
-> evaluate low-stock condition

large Collective order
-> generate alert
```

SCHEDULED:

Periodic jobs.

Initially Kubernetes CronJobs are acceptable where appropriate.

However, design toward a scheduler service.

Future schedule model might include:

```text
tenant_id
job_name
frequency
timezone
next_run_at
enabled
configuration
```

The scheduler should emit work onto:

```text
analytics.jobs
```

rather than executing business analytics itself.

Example:

```json
{
  "tenant_id": "devmoto",
  "job": "sales.channel.breakdown",
  "period": {
    "start": "...",
    "end": "..."
  }
}
```

Workers consume jobs and produce results.

---

## ANALYTICS RESULTS

Analytics output should be represented as a structured result.

Example:

```json
{
  "tenant_id": "devmoto",
  "job": "sales.channel.breakdown",
  "period": {
    "start": "...",
    "end": "..."
  },
  "sales": {
    "total": 10254,
    "channels": {
      "shop": 6842,
      "collective": {
        "total": 3412,
        "partners": [
          {
            "name": "E-Moto Co",
            "sales": 1282
          }
        ]
      }
    }
  }
}
```

Then publish something conceptually like:

```text
analytics.result.generated
```

or to:

```text
analytics.results
```

Notification systems should consume those results.

---

## NOTIFICATIONS

Do not have analytics workers send SMS directly.

Use:

```text
analytics
    |
notifications.requested
    |
notifier
    |
SMS / email / Slack / other channels
```

This keeps analytics independent from presentation/delivery.

Initially support a simple text formatter.

Do not tightly couple the notifier to one SMS vendor.

Create an interface so vendors can be swapped.

Credentials must live in AWS Secrets Manager.

---

## RAW ARCHIVE

Archive raw source events inexpensively to S3.

S3 should provide a durable/replayable history where practical.

Consider partitioning keys such as:

```text
source=shopify/
tenant=devmoto/
event_type=orders_create/
year=2026/
month=09/
day=11/
```

or a better design if justified.

Use lifecycle rules to reduce storage cost.

Do not store unnecessary duplicate data forever.

Document retention strategy.

Use encryption.

---

## OBSERVABILITY

I want a complete open-source observability stack.

Use:

* Prometheus
* Loki
* OpenTelemetry
* Tempo
* Grafana
* Hubble

I want metrics, logs, traces, and network telemetry.

Architecture should conceptually support:

```text
Applications
    |
OpenTelemetry
    |
OTel Collector
    |-------- metrics -> Prometheus
    |-------- traces  -> Tempo
    |-------- logs    -> Loki where appropriate

Grafana
    |
    ├── Prometheus
    ├── Loki
    └── Tempo

Cilium
    |
Hubble
```

Prefer OpenTelemetry instrumentation in Go applications.

Applications should expose:

* request metrics
* webhook counts
* webhook failures
* HMAC verification failures
* Redpanda publish latency
* consumer lag where practical
* reconciliation counts
* reconciliation failures
* analytics execution duration
* analytics errors
* DB query latency
* notification delivery success/failure

Use distributed tracing where useful.

Include trace propagation through Redpanda events where practical.

Use structured logging.

Correlate:

* trace ID
* event ID
* tenant ID
* job ID

Be careful not to expose sensitive customer data or credentials.

---

## OBSERVABILITY COST CONTROL

Do not configure huge retention periods.

This is a small initial environment.

Keep resource requests and retention intentionally small.

Provide configurable retention.

For example, initial thinking may be:

Prometheus:
short local retention

Loki:
short searchable retention with possible S3-backed longer-term storage later

Tempo:
short trace retention

Hubble:
appropriate flow visibility without attempting to persist everything forever

Do not send all telemetry to expensive AWS managed services by default.

Do not deploy Amazon Managed Prometheus or Amazon Managed Grafana unless there is a compelling reason.

Self-hosting inside the existing EKS cluster is preferred initially for cost reasons.

---

## GRAFANA

Provide useful initial dashboards.

At minimum:

1. Platform overview
2. Shopify ingestion
3. Redpanda health
4. Analytics workers
5. PostgreSQL health
6. Notifications
7. Cilium/Hubble network visibility

Examples of useful panels:

* webhook rate
* webhook failures
* event ingestion rate
* Redpanda lag
* analytics jobs completed
* analytics failures
* analytics duration
* notification failures
* reconciliation mismatches
* pod CPU/memory
* Postgres connections
* Postgres latency
* network denied flows

Also create a simple BUSINESS dashboard eventually showing:

* sales today
* shop sales
* Collective sales
* Collective partner breakdown
* orders
* AOV

The dashboard should consume application/database metrics or APIs rather than replace the application's analytics layer.

---

## SLO/HEALTH THINKING

Even in the MVP, define useful health signals.

Examples:

```text
Shopify webhook accepted
Shopify webhook rejected
Redpanda publish success
Consumer lag
Last successful reconciliation
Last successful analytics run
Notification delivery result
Database connectivity
```

Provide Kubernetes:

* readiness probes
* liveness probes
* startup probes where necessary

Avoid probes that create load or false positives.

---

## APPLICATION REPOSITORY STRUCTURE

A Go service should use clean, conventional structure.

Conceptually:

```text
cmd/
internal/
pkg/        # only if genuinely reusable externally
migrations/
deploy/
docs/
```

Do not over-engineer Go package hierarchy.

Use:

* Go modules
* context.Context
* structured logging
* dependency injection through constructors
* interfaces around external systems
* database migrations
* unit tests
* integration tests where practical
* table-driven tests

Avoid gratuitous frameworks.

Prefer the Go standard library and mature lightweight libraries.

---

## DATABASE MIGRATIONS

Use a proper migration framework.

Migration files must be version-controlled.

Do not mutate schemas manually.

Provide commands like:

```text
make migrate-up
make migrate-down
```

or equivalents.

Production DB credentials must never be embedded in those commands.

---

## CONFIGURATION

Use environment variables or mounted configuration for non-sensitive config.

Examples:

```text
AWS_REGION
REDPANDA_BROKERS
DATABASE_HOST
DATABASE_PORT
DATABASE_NAME
OTEL_EXPORTER_OTLP_ENDPOINT
LOG_LEVEL
```

Secrets should be retrieved from AWS Secrets Manager.

Avoid giant configuration frameworks.

Validate configuration at startup and fail clearly if required config is missing.

---

## CONTAINERS

Create efficient multi-stage Dockerfiles.

Prefer:

* small runtime images
* non-root user
* read-only filesystem where practical
* minimal package surface
* explicit version pinning
* health endpoints
* graceful shutdown

Generate SBOM/security scanning where reasonable.

Do not embed credentials in container images.

---

## ECR

Use Amazon ECR for built application images.

Create repositories through Terraform.

Enable image scanning.

Use lifecycle policies so old images do not accumulate forever.

---

## CI

GitHub Actions should perform:

```text
go fmt verification
go vet
go test
golangci-lint
secret scanning
dependency/security scanning
Docker build
container scanning
Terraform formatting
Terraform validation where relevant
Terragrunt formatting
```

Do not automatically deploy to AWS.

CI builds and validates.

Deployment workflows can be added later but should require explicit authorization.

Avoid long-lived AWS credentials in GitHub.

If AWS access is added later, prefer GitHub OIDC federation.

---

## INFRASTRUCTURE COST CONTROLS

Implement AWS Budgets using Terraform if practical.

Potential thresholds:

```text
$100
$150
$200
$250
```

Document estimated monthly costs for each major infrastructure component.

At minimum discuss:

* EKS control plane
* EKS workers
* EBS
* RDS
* NLB
* NAT/egress
* S3
* CloudWatch baseline usage
* Secrets Manager
* ECR
* Route53
* data transfer

Pay particular attention to NAT Gateway cost.

Do NOT automatically create one NAT Gateway per AZ.

Evaluate cheaper alternatives appropriate to an MVP.

Document availability tradeoffs.

Avoid accidental cross-AZ traffic where practical.

---

## INITIAL SERVICE DECOMPOSITION

Start small.

A reasonable initial logical set is:

```text
shopify-ingestor
normalizer
reconciler
scheduler
analytics-worker
notifier
```

These do NOT necessarily need six separate GitHub repositories.

Choose repository boundaries based on:

* independent lifecycle
* clear ownership
* deployability
* code sharing
* avoiding excessive fragmentation

Explain the decision.

---

## FIRST ANALYTICS JOB

Implement:

```text
sales.channel.breakdown
```

Input:

```text
tenant
start timestamp
end timestamp
timezone
```

Output:

```text
total sales
shop sales
Collective sales
Collective partner breakdown
order count
AOV
```

Support:

```text
daily
weekly
monthly
quarterly
```

Use exact date ranges internally.

Period logic must respect the tenant's timezone.

Store timestamps in UTC.

Explicitly define whether the report uses:

* order created time
* processed time
* paid time

Choose a sensible initial definition and document it.

---

## SQL

The normalized schema should make queries straightforward.

Conceptually:

```sql
SELECT
    channel,
    collective_partner_id,
    SUM(net_sales),
    COUNT(*)
FROM orders
WHERE tenant_id = $1
  AND created_at >= $2
  AND created_at < $3
GROUP BY
    channel,
    collective_partner_id;
```

Add indexes based on actual query patterns.

Do not prematurely create dozens of indexes.

---

## DATA QUALITY

The system must make it easy to understand discrepancies.

Track things such as:

```text
source event ID
Shopify order ID
event ingestion timestamp
source updated_at
last reconciled_at
classification version
normalization version
```

Design for replay.

If normalization logic changes, historical events should be capable of reprocessing where practical.

---

## IDEMPOTENCY

This is extremely important.

Shopify webhooks can be duplicated.

Redpanda consumers may replay messages.

Jobs may retry.

Therefore:

* ingestion must be idempotent
* normalization must be idempotent
* database upserts must be safe
* reconciliation must be idempotent
* notification duplication should be prevented where practical

Design explicit idempotency keys.

---

## ERROR HANDLING

Do not discard failed events silently.

Design dead-letter/error handling.

Potential model:

```text
shopify.orders.raw
shopify.orders.failed

analytics.jobs
analytics.jobs.failed

notifications.requested
notifications.failed
```

Do not build an unnecessarily elaborate DLQ framework initially, but failures must be inspectable and replayable.

---

## DOCUMENTATION

Create architectural documentation.

At minimum provide:

```text
docs/
  architecture.md
  event-model.md
  database-model.md
  security.md
  observability.md
  cost-model.md
  terraform-versioning.md
  local-development.md
  deployment.md
  disaster-recovery.md
```

Use Mermaid diagrams where appropriate.

Explain:

* current architecture
* MVP compromises
* future scale path
* security boundaries
* event flows
* failure modes
* recovery process
* cost drivers

---

## ADRs

Use lightweight Architecture Decision Records for important choices.

Examples:

```text
ADR-001 PostgreSQL as initial database
ADR-002 Redpanda as event bus
ADR-003 Cilium networking
ADR-004 Single EKS cluster
ADR-005 Single-node Redpanda MVP
ADR-006 Single-AZ RDS MVP
ADR-007 Multi-tenancy from day one
ADR-008 Generic analytics worker framework
ADR-009 OpenTelemetry observability
ADR-010 Terraform module semantic versioning
```

Document alternatives and reasoning.

---

## DISASTER RECOVERY

The MVP should be inexpensive but recoverable.

Think of data ownership as:

```text
Shopify
    authoritative upstream commerce source

S3
    durable raw event archive

PostgreSQL
    normalized current state / analytics state

Redpanda
    transport/event bus
```

If the single Redpanda broker is lost, we should be able to reconstruct/replay data.

Document the recovery process.

Back up PostgreSQL appropriately.

---

## FUTURE SCALE PATH

Do not implement all of this yet, but design so we can later move toward:

```text
Redpanda:
1 broker -> 3+ broker HA cluster or Redpanda Cloud

PostgreSQL:
Single-AZ -> Multi-AZ -> replicas -> potentially specialized analytics DB

EKS:
small cluster -> larger multi-AZ cluster

Analytics:
Postgres -> potentially ClickHouse for large-scale analytics

Identity:
single internal tenant -> SaaS user/org identity

Notifications:
SMS -> SMS/email/Slack/push/webhooks

Sources:
Shopify -> multi-platform commerce ecosystem

AI:
analytics -> recommendations -> controlled autonomous agents
```

Avoid premature implementation.

---

## WHAT I WANT YOU TO DO FIRST

Do NOT start randomly creating files.

Work through this sequence.

### Step 1 — Inspect

Inspect:

```text
/Users/terencehegarty/projects/hegarty/terraform
/Users/terencehegarty/projects/hegarty/eks
```

Understand:

* current Terraform module structure
* Terragrunt conventions
* existing VPC modules
* existing EKS modules
* current tagging/versioning practices
* state configuration
* provider configuration
* naming conventions
* reusable modules already available

Also inspect the current Git environment.

Do not modify anything yet.

### Step 2 — Produce an implementation plan

Before coding, summarize:

* proposed GitHub repositories
* local directory layout
* Terraform modules that already exist
* Terraform modules that need to be created
* proposed Terragrunt hierarchy
* EKS architecture
* Redpanda deployment strategy
* Postgres strategy
* secrets strategy
* observability strategy
* Cilium strategy
* expected AWS resources
* estimated monthly AWS costs
* major MVP compromises

Also identify any assumptions.

Do not ask unnecessary questions if a reasonable default exists.

### Step 3 — Establish Terraform versioning

Design and implement the module-specific semantic tag strategy for:

```text
/Users/terencehegarty/projects/hegarty/terraform
```

Do not push tags without my approval.

Update documentation/examples so Terragrunt references immutable versions.

### Step 4 — Create missing generic Terraform modules

Only create modules actually needed.

Reuse existing modules wherever possible.

Validate them.

### Step 5 — Create Terragrunt configuration

Create environment configuration beneath:

```text
/Users/terencehegarty/projects/hegarty/eks
```

following my existing conventions.

Do not deploy it.

### Step 6 — Create application repositories

Use the prefix:

```text
commerce-intel-
```

Create only the repositories justified by the architecture.

If GitHub CLI authentication exists, create the repos.

Otherwise provide exact commands.

Clone/create them beneath a sensible local parent directory under:

```text
/Users/terencehegarty/projects/hegarty/
```

### Step 7 — Implement platform foundation

Build:

* Shopify webhook ingestion
* Redpanda event publishing
* event normalization
* PostgreSQL persistence
* S3 raw archive
* reconciliation
* analytics worker
* initial sales.channel.breakdown job
* notifier abstraction

### Step 8 — Implement Kubernetes packaging

Provide:

* Helm charts or appropriate manifests
* ServiceAccounts
* Pod Identity/IAM mapping
* Secrets Manager integration
* Cilium policies
* Gateway API objects
* health probes
* resource requests/limits
* autoscaling only where justified

### Step 9 — Implement observability

Deploy/configure via code:

* Prometheus
* Loki
* OpenTelemetry Collector
* Tempo
* Grafana
* Hubble

Instrument Go applications.

Create useful dashboards.

Keep resource usage and retention inexpensive.

### Step 10 — Validate

Run all safe local validation possible.

Examples:

```text
go test ./...
go vet ./...
golangci-lint run
terraform fmt -recursive
terraform validate
terragrunt hclfmt
helm lint
docker build
```

Do not run deployment commands.

### Step 11 — Give me the deployment/runbook

At the end, provide the ordered commands I should execute myself.

Clearly identify commands that:

* create AWS resources
* push GitHub changes
* create Git tags
* install Kubernetes software
* create Shopify resources

Do not execute those commands yourself.

---

## WORKING STYLE

Take this project seriously.

Do not optimize for producing the largest amount of code quickly.

Optimize for:

* maintainability
* correctness
* security
* low AWS cost
* observability
* replayability
* testability
* clear boundaries
* future commercialization
* operational simplicity

Avoid speculative infrastructure.

When a design decision creates recurring AWS cost, explicitly call it out.

When you find an existing module or implementation that can be reused, reuse it rather than duplicating it.

When modifying an existing repository, preserve existing conventions unless there is a strong technical reason not to.

Do not hide important assumptions.

Do not deploy anything.

Do not put secrets into source control.

Remember throughout the entire implementation:

**ALL APPLICATION AND INFRASTRUCTURE CODE MAY BE PUBLICLY VISIBLE ON GITHUB. NO SECRET OR SENSITIVE VALUE SHOULD EVER BE COMMITTED. AWS SECRETS MANAGER IS THE AUTHORITATIVE STORAGE LOCATION FOR APPLICATION SECRETS.**

Start with Step 1: inspect the existing Terraform and Terragrunt repositories and report what you find before modifying them.

---

*Two corrections were made after this brief, in conversation, before implementation began:*
*repository prefix changed from `commerce-intel-` to `shop_`, and repos were made public*
*from the start (governed by policy-as-code) rather than private. See*
*[`milestones.md`](milestones.md) for where those decisions are reflected.*
