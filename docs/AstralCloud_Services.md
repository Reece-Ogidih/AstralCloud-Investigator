# AstralCloud Production Services

**Version:** 0.1
**Status:** Working canonical service architecture
**Last updated:** September 2026

---

# 1. Purpose

This document defines the principal production services that make up AstralCloud.

The services described here sit beneath the previously established architectural domains and engineering-team structure.

A service exists as a distinct deployment unit where there is a meaningful reason for independent:

* scaling;
* deployment;
* ownership;
* failure;
* release cadence;
* operational behaviour.

AstralCloud deliberately avoids splitting functionality into unnecessarily small microservices.

The initial architecture consists of approximately **27 significant deployable software components**, including the customer-side collector and web application.

This number may evolve as implementation progresses.

---

# 2. Service architecture overview

```text
CUSTOMER ENVIRONMENT
────────────────────────────────────────────────────

                 astral-collector
                       │
                       │ OTLP
                       ▼


ASTRALCLOUD DATA PLANE
────────────────────────────────────────────────────

                telemetry-gateway
                       │
                       ▼
                ingestion-router
                       │
                       ▼
                     Kafka
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
 metrics-processor logs-processor traces-processor
        │              │              │
        │              │              ├──► service-map-worker
        │              │              │
        └──────────────┼──────────────┘
                       ▼
              ClickHouse / S3
                       │
                       ▼
                 query-gateway
                       │
                       ▼
                 query-planner
                ┌──────┼──────┐
                ▼      ▼      ▼
          metrics-api logs-api apm-api


CONTROL / PRODUCT PLANE
────────────────────────────────────────────────────

                     public-api
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
 identity-service organisation-service dashboard-service
       │                 │
       │                 └──► api-key-service
       │
       └─────────────────────────────┐
                                     ▼
                              PostgreSQL


ALERTING / INCIDENTS
────────────────────────────────────────────────────

                  monitor-service
                        │
                        ▼
                  alert-evaluator
                        │
                        ▼
                  incident-service
                        │
                        ▼
                notification-service


USAGE / BILLING
────────────────────────────────────────────────────

Telemetry ──► usage-aggregator ──► billing-service
                                       │
                                       ▼
                               entitlement-service


CUSTOMER INTERFACE
────────────────────────────────────────────────────

                    web-app
                       │
                       ▼
                   public-api
```

---

# 3. Customer-side collection

## 3.1 `astral-collector`

**Owner:** Collectors & Edge
**Primary language:** Go
**Deployment location:** Customer infrastructure

The Astral Collector is the primary telemetry collection agent installed in customer environments.

It may run as:

* Kubernetes DaemonSet;
* sidecar;
* standalone container;
* host agent.

Responsibilities include telemetry collection, batching, compression, retry handling, local buffering, Kubernetes metadata discovery and OpenTelemetry-compatible export.

The collector does not contain customer business logic.

It produces OTLP traffic directed toward AstralCloud ingestion endpoints.

Important failure modes include memory leaks, excessive CPU usage, incompatible upgrades, retry storms, dropped buffers and malformed metadata.

---

# 4. Edge and ingestion

## 4.1 `telemetry-gateway`

**Owner:** Collectors & Edge
**Primary language:** Go
**Deployment:** Regional EKS Deployment
**Characteristics:** Stateless, horizontally scalable

The telemetry gateway is the first AstralCloud application reached by customer telemetry.

It receives OTLP over HTTP and gRPC.

Responsibilities include:

* protocol termination;
* request decoding;
* basic payload validation;
* tenant authentication;
* cached API-key validation;
* request-size enforcement;
* regional rate limiting;
* assignment of canonical request metadata.

It deliberately performs minimal heavy processing.

The gateway should remain highly horizontally scalable.

---

## 4.2 `ingestion-router`

**Owner:** Ingestion & Routing
**Primary language:** Go
**Deployment:** Regional EKS Deployment
**Characteristics:** Stateless

The ingestion router receives validated telemetry from the gateway.

It performs:

```text
tenant validation
      ↓
quota enforcement
      ↓
schema classification
      ↓
canonical metadata enrichment
      ↓
topic selection
      ↓
Kafka production
```

Representative destination topics include:

```text
metrics.raw
logs.raw
traces.raw
```

The router acknowledges telemetry only according to the configured ingestion durability guarantees.

Failures here may affect all telemetry types or only specific tenants/types depending on the fault.

---

## 4.3 `tenant-config-sync`

**Owner:** Ingestion & Routing
**Primary language:** Go
**Deployment:** Regional EKS Deployment / worker

The data plane must not synchronously query the global control plane for every telemetry request.

`tenant-config-sync` therefore distributes important tenant configuration into regional caches.

Examples include:

```text
active API keys
tenant status
quotas
entitlements
retention tier
feature flags relevant to ingestion
```

The service consumes control-plane change events and maintains regional Redis-backed configuration.

This creates an intentional consistency boundary.

For example:

```text
API key revoked globally
        ↓
propagation delayed
        ↓
regional gateway temporarily accepts old key
```

This is a realistic class of distributed-system problem.

---

# 5. Telemetry processing

## 5.1 `metrics-processor`

**Owner:** Metrics
**Primary language:** Go
**Deployment:** Regional EKS Deployment
**Workload:** Kafka consumer

Consumes:

```text
metrics.raw
```

Responsibilities include:

* metric validation;
* tag normalisation;
* metadata enrichment;
* rollup preparation;
* cardinality controls;
* transformation into storage representation.

Processed metric batches are written into ClickHouse.

The service also produces relevant downstream events for alerting and usage metering.

---

## 5.2 `logs-processor`

**Owner:** Logs
**Primary language:** Go
**Deployment:** Regional EKS Deployment
**Workload:** Kafka consumer

Consumes:

```text
logs.raw
```

Responsibilities include:

* parsing;
* timestamp normalisation;
* metadata extraction;
* severity normalisation;
* trace correlation;
* enrichment;
* preparation for ClickHouse storage.

The log processor supports both structured and unstructured customer logs.

---

## 5.3 `traces-processor`

**Owner:** APM & Tracing
**Primary language:** Go
**Deployment:** Regional EKS Deployment
**Workload:** Kafka consumer

Consumes:

```text
traces.raw
```

Responsibilities include:

* span validation;
* trace assembly;
* sampling;
* metadata normalisation;
* service identification;
* latency calculation;
* dependency extraction.

Processed traces are written into ClickHouse and relevant downstream streams.

---

## 5.4 `service-map-worker`

**Owner:** APM & Tracing
**Primary language:** Go
**Deployment:** Regional EKS worker

Consumes processed trace information and constructs service dependency relationships.

For example:

```text
checkout-api
      │
      ▼
payments-api
      │
      ▼
stripe
```

The resulting information powers AstralCloud's service maps and dependency views.

Because this processing is asynchronous, traces can remain searchable even while service maps are stale.

---

## 5.5 `archive-worker`

**Owner:** Storage Platform
**Primary language:** Go
**Deployment:** EKS worker / scheduled workloads

Moves eligible telemetry into lower-cost S3-backed storage according to retention rules.

Responsibilities include:

* archival;
* retention enforcement;
* lifecycle processing;
* export preparation;
* cold-storage metadata.

Errors here generally do not immediately affect hot telemetry but can cause retention, storage-cost or historical-query problems.

---

# 6. Query infrastructure

## 6.1 `query-gateway`

**Owner:** Query Platform
**Primary language:** Go
**Deployment:** Regional EKS Deployment

Provides the common entry point for telemetry queries.

Responsibilities include:

* query authentication;
* tenant isolation enforcement;
* query limits;
* request validation;
* timeout control;
* request routing;
* cache interaction.

All product APIs use the query platform rather than directly querying ClickHouse.

---

## 6.2 `query-planner`

**Owner:** Query Platform
**Primary language:** Go
**Deployment:** Regional EKS Deployment

Translates AstralCloud query requests into efficient storage operations.

Responsibilities include:

* query planning;
* time-range optimisation;
* storage selection;
* aggregation planning;
* ClickHouse query construction;
* cost estimation;
* execution coordination.

This component is intentionally separate because query planning is complex enough to have an independent release and failure profile.

A query-planner regression can therefore cause customer-visible data errors even when storage is completely healthy.

---

# 7. Product telemetry APIs

## 7.1 `metrics-api`

**Owner:** Metrics
**Primary language:** Go
**Deployment:** Regional EKS Deployment

Provides domain-specific operations for the metrics product.

Examples include:

```text
metric discovery
metric metadata
timeseries queries
tag discovery
aggregation requests
```

It translates product concepts into requests against the shared query platform.

---

## 7.2 `logs-api`

**Owner:** Logs
**Primary language:** Go
**Deployment:** Regional EKS Deployment

Provides:

```text
log search
faceting
log metadata
saved query execution
trace/log correlation
```

The API relies on the query platform for underlying telemetry retrieval.

---

## 7.3 `apm-api`

**Owner:** APM & Tracing
**Primary language:** Go
**Deployment:** Regional EKS Deployment

Provides customer-facing APM functionality including:

```text
trace search
service performance
endpoint latency
dependency information
service maps
error traces
```

It reads from both trace storage and derived service metadata.

---

# 8. Alerting

## 8.1 `monitor-service`

**Owner:** Detection & Alerting
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns customer monitor definitions.

Responsibilities include:

* monitor creation;
* monitor updates;
* monitor validation;
* muting;
* evaluation configuration;
* threshold configuration;
* monitor metadata.

Canonical monitor configuration is stored in PostgreSQL.

Relevant configuration is distributed to alert-evaluation infrastructure asynchronously.

---

## 8.2 `alert-evaluator`

**Owner:** Detection & Alerting
**Primary language:** Go
**Deployment:** Regional EKS workers

Evaluates customer monitors.

Depending on monitor type, evaluation may consume streaming telemetry or execute queries through the query platform.

Responsibilities include:

```text
evaluation
state tracking
threshold comparison
deduplication
alert transitions
```

Possible transitions include:

```text
OK → WARNING
WARNING → ALERT
ALERT → RECOVERED
```

Alert events are then emitted for downstream incident and notification processing.

---

# 9. Incidents and notifications

## 9.1 `incident-service`

**Owner:** Incidents & Notifications
**Primary language:** Python / FastAPI
**Deployment:** Control-plane EKS

Owns AstralCloud incident objects.

Responsibilities include:

* incident creation;
* incident status;
* severity;
* responders;
* timelines;
* linked monitors;
* linked services;
* incident notes;
* postmortem references.

The incident service stores its primary data in PostgreSQL.

---

## 9.2 `notification-service`

**Owner:** Incidents & Notifications
**Primary language:** Python
**Deployment:** EKS API + workers

Responsible for outbound communication.

Supported destinations include:

```text
Slack
PagerDuty
email
webhooks
```

Responsibilities include:

* notification rendering;
* routing;
* retries;
* rate limits;
* provider authentication;
* delivery tracking;
* dead-letter handling.

A failed notification does not imply a failed alert evaluation.

---

# 10. Core control plane

## 10.1 `identity-service`

**Owner:** Identity & Organisations
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns human authentication and identity.

Responsibilities include:

* login;
* password authentication where applicable;
* SSO;
* sessions;
* identity-provider integration;
* authentication tokens.

---

## 10.2 `organisation-service`

**Owner:** Identity & Organisations
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns:

```text
organisations
projects
teams
membership
RBAC
organisation settings
```

The organisation is the principal AstralCloud tenant boundary.

---

## 10.3 `api-key-service`

**Owner:** Identity & Organisations
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns machine credentials used by collectors and API clients.

Responsibilities include:

* API-key generation;
* hashing;
* revocation;
* rotation;
* scopes;
* audit metadata.

Changes generate configuration events consumed by `tenant-config-sync`.

The ingestion path therefore never needs to synchronously call `api-key-service`.

---

## 10.4 `entitlement-service`

**Owner:** Billing & Metering
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Determines which functionality and limits an organisation is entitled to use.

Examples include:

```text
retention period
ingestion quota
premium APM features
advanced alerting
user limits
```

Entitlement changes are propagated into relevant runtime systems.

---

# 11. Usage and billing

## 11.1 `usage-aggregator`

**Owner:** Billing & Metering
**Primary language:** Go
**Deployment:** EKS workers

Consumes usage events generated throughout the platform.

Examples include:

```text
logs_ingested_bytes
trace_spans_ingested
metric_samples_ingested
active_hosts
```

It performs aggregation and idempotent processing before updating the usage ledger.

Correctness is more important than immediate latency.

---

## 11.2 `billing-service`

**Owner:** Billing & Metering
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns commercial billing logic.

Responsibilities include:

* pricing plans;
* subscriptions;
* invoice preparation;
* usage conversion;
* discounts;
* billing periods;
* payment-provider integration.

The billing service reads validated aggregated usage rather than raw telemetry.

---

# 12. Customer product layer

## 12.1 `dashboard-service`

**Owner:** Web Experience
**Primary language:** Kotlin / Spring Boot
**Deployment:** Control-plane EKS

Owns persisted customer presentation configuration.

Examples include:

```text
dashboards
widgets
saved views
saved queries
layout configuration
```

It stores configuration rather than raw telemetry.

---

## 12.2 `public-api`

**Owner:** Web Experience
**Primary language:** TypeScript / Node.js
**Deployment:** EKS

Acts as the principal backend-for-frontend and public API composition layer.

It provides a stable customer interface while delegating domain operations to underlying services.

Conceptually:

```text
web-app
   │
   ▼
public-api
   │
   ├── identity-service
   ├── organisation-service
   ├── dashboard-service
   ├── monitor-service
   ├── incident-service
   ├── billing-service
   ├── metrics-api
   ├── logs-api
   └── apm-api
```

It must not become the location where all business logic accumulates.

---

## 12.3 `web-app`

**Owner:** Web Experience
**Primary language:** TypeScript
**Framework:** React

Customer-facing AstralCloud web application.

Production artifacts are built into static frontend bundles and served through appropriate AWS edge/CDN infrastructure.

Major product areas include:

```text
Overview
Infrastructure
APM
Metrics
Logs
Alerts
Incidents
SLOs
Usage & Billing
Integrations
Organisation Settings
```

---

# 13. Shared infrastructure components

Several critical production systems are not AstralCloud application services.

These include:

| Component            | Purpose                        | Primary owner                            |
| -------------------- | ------------------------------ | ---------------------------------------- |
| Amazon EKS           | Kubernetes runtime             | Developer Platform / SRE                 |
| Docker               | Application packaging          | All teams / Developer Platform standards |
| Amazon ECR           | Container registry             | Developer Platform                       |
| Kafka                | Event streaming                | Streaming Platform                       |
| Schema Registry      | Event schema governance        | Streaming Platform                       |
| ClickHouse           | Telemetry analytics storage    | Storage Platform                         |
| PostgreSQL           | Relational state               | Platform + owning application teams      |
| Redis                | Cache / ephemeral state        | Platform + owning teams                  |
| Amazon S3            | Archive/object storage         | Storage Platform                         |
| Argo CD              | Continuous delivery            | Developer Platform                       |
| Helm                 | Kubernetes packaging           | Developer Platform / service owners      |
| Terraform            | Infrastructure as code         | Platform & Reliability                   |
| GitHub Actions       | CI                             | Developer Platform                       |
| AWS IAM              | Cloud identity/permissions     | Security / Platform                      |
| Secrets Manager      | Secrets                        | Security / Platform                      |
| CloudWatch           | Minimal out-of-band visibility | SRE                                      |
| Load Balancers / DNS | Traffic ingress                | Platform & Reliability                   |

These systems are equally valid investigation targets.

An incident does not need to originate in custom application code.

---

# 14. Container and deployment model

All server-side AstralCloud application services are containerised using Docker.

A typical application repository produces:

```text
source
   ↓
tests
   ↓
Docker build
   ↓
security / dependency checks
   ↓
ECR image
   ↓
Helm configuration
   ↓
Argo CD
   ↓
EKS
```

Individual services normally run as Kubernetes Deployments.

Background processing services typically run as long-lived Kubernetes workers.

Scheduled maintenance tasks may run as Kubernetes CronJobs or Jobs.

Every deployable service should eventually possess:

```text
Dockerfile
application configuration
health endpoint
metrics endpoint
structured logging
tests
Helm configuration
CI workflow
deployment metadata
```

---

# 15. Example: complete trace lifecycle

A customer application generates a span:

```text
checkout-api
    ↓
POST payment
```

The trace flows:

```text
Customer application
        ↓
astral-collector
        ↓
telemetry-gateway
        ↓
ingestion-router
        ↓
Kafka: traces.raw
        ↓
traces-processor
        ├──────────────► service-map-worker
        │
        ▼
ClickHouse
        ↓
query-planner
        ↓
query-gateway
        ↓
apm-api
        ↓
public-api
        ↓
web-app
```

At the same time:

```text
ingestion-router
        ↓
usage event
        ↓
usage-aggregator
        ↓
billing-service
```

and potentially:

```text
processed trace telemetry
        ↓
alert-evaluator
        ↓
incident-service
        ↓
notification-service
```

A single customer request can therefore interact with several independent systems.

---

# 16. Example investigation ambiguity

Assume a customer reports:

> Traces have disappeared from the APM interface.

This symptom alone does not identify the failing service.

Possible locations include:

```text
astral-collector
        ↓
telemetry-gateway
        ↓
ingestion-router
        ↓
Kafka
        ↓
traces-processor
        ↓
ClickHouse
        ↓
query-planner
        ↓
query-gateway
        ↓
apm-api
        ↓
public-api
        ↓
web-app
```

For example:

* the collector could have stopped exporting;
* the gateway could reject payloads;
* ingestion could route to the wrong topic;
* Kafka could have consumer lag;
* the processor could crash on a schema change;
* ClickHouse writes could fail;
* a query-planner regression could exclude valid data;
* APM filtering could be incorrect;
* the frontend could display the wrong time range.

The investigator must determine which layer actually failed.

---

# 17. Ownership summary

| Service                | Primary owner             |
| ---------------------- | ------------------------- |
| `astral-collector`     | Collectors & Edge         |
| `telemetry-gateway`    | Collectors & Edge         |
| `ingestion-router`     | Ingestion & Routing       |
| `tenant-config-sync`   | Ingestion & Routing       |
| `metrics-processor`    | Metrics                   |
| `logs-processor`       | Logs                      |
| `traces-processor`     | APM & Tracing             |
| `service-map-worker`   | APM & Tracing             |
| `archive-worker`       | Storage Platform          |
| `query-gateway`        | Query Platform            |
| `query-planner`        | Query Platform            |
| `metrics-api`          | Metrics                   |
| `logs-api`             | Logs                      |
| `apm-api`              | APM & Tracing             |
| `monitor-service`      | Detection & Alerting      |
| `alert-evaluator`      | Detection & Alerting      |
| `incident-service`     | Incidents & Notifications |
| `notification-service` | Incidents & Notifications |
| `identity-service`     | Identity & Organisations  |
| `organisation-service` | Identity & Organisations  |
| `api-key-service`      | Identity & Organisations  |
| `entitlement-service`  | Billing & Metering        |
| `usage-aggregator`     | Billing & Metering        |
| `billing-service`      | Billing & Metering        |
| `dashboard-service`    | Web Experience            |
| `public-api`           | Web Experience            |
| `web-app`              | Web Experience            |

---

# 18. Service count

The current model contains:

```text
1 customer collector
20 backend application services/workers
5 control/product services
1 frontend application

≈27 significant software components
```

This does not count infrastructure such as Kafka, ClickHouse, PostgreSQL, Redis, Kubernetes or Argo CD as application services.

The exact number is less important than preserving meaningful operational boundaries.

---

# 19. Implementation principle for the synthetic company

The eventual AstralCloud source repositories do not need to reproduce the entire feature depth of a commercial observability platform.

However, the code that exists must be **real and executable**.

Where a service is materialised it should contain genuine behaviour such as:

```text
HTTP/gRPC endpoints
Kafka producers/consumers
database access
configuration handling
business logic
error paths
tests
Docker packaging
deployment configuration
structured logging
metrics
```

Synthetic incidents should arise from real code or configuration changes wherever practical.

For example, an incident should ideally originate from an actual diff such as:

```diff
- pool_size = 100
+ pool_size = 10
```

or:

```diff
- key = tenant_id
+ key = project_id
```

rather than from a metadata file merely declaring that a failure happened.

This ensures the investigator can inspect genuine source evidence.

---

# 20. Current canonical decisions

The service set above is considered the starting canonical production model.

The architecture establishes:

* distinct ingestion and query paths;
* independent metrics, logs and tracing processors;
* shared streaming infrastructure;
* shared storage infrastructure;
* shared query infrastructure;
* asynchronous control-plane propagation;
* separate alert evaluation and notification;
* independent usage and billing processing;
* a real multi-service control plane;
* containerised Kubernetes deployment;
* realistic cross-service failure boundaries.

Service boundaries may be revised where implementation reveals unnecessary complexity or missing responsibilities.
