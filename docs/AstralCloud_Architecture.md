# AstralCloud High-Level Architecture

**Version:** 0.1
**Status:** Working canonical architecture
**Last updated:** September 2026

---

## 1. Purpose

This document defines the high-level technical architecture of AstralCloud.

It establishes the major system boundaries, infrastructure principles, telemetry flow, storage model and operational architecture before individual services, repositories and engineering teams are defined.

The purpose is to provide a consistent technical foundation for:

* synthetic company history generation;
* source-code repositories;
* Git history;
* CI/CD activity;
* infrastructure configuration;
* logs, metrics and traces;
* incidents;
* Slack-style engineering communication;
* Jira-style tickets;
* documentation;
* agent investigation scenarios;
* evaluation ground truth.

Individual service boundaries and implementation details remain subject to refinement.

---

# 2. Architectural principles

AstralCloud is a multi-tenant B2B observability and production-operations platform.

Its architecture should reflect a growth-stage production SaaS company rather than a simplified demonstration application.

Core principles are:

1. Heavy telemetry processing is separated from ordinary SaaS control-plane functionality.
2. The platform is multi-tenant throughout.
3. Telemetry ingestion is asynchronous wherever practical.
4. Large-volume workloads use streaming infrastructure.
5. Customer-facing query paths are separated from ingestion paths.
6. Infrastructure is containerised.
7. Production workloads run predominantly on Kubernetes.
8. Infrastructure is managed as code.
9. AstralCloud dogfoods its own observability platform.
10. Critical infrastructure retains limited out-of-band visibility.
11. Individual failures should be capable of producing partial degradation rather than total platform failure.
12. Architecture should permit realistic cross-service and cross-team incidents.

---

# 3. Three architectural planes

AstralCloud consists conceptually of three major planes.

## 3.1 Data Plane

The data plane handles high-volume customer telemetry.

Responsibilities include:

* telemetry ingestion;
* routing;
* streaming;
* enrichment;
* processing;
* storage;
* querying;
* alert evaluation.

Typical workloads include millions of telemetry events per second.

---

## 3.2 Control Plane

The control plane manages comparatively low-volume SaaS state.

Responsibilities include:

* organisations;
* users;
* projects;
* authentication;
* permissions;
* API keys;
* dashboards;
* monitors;
* integrations;
* retention configuration;
* feature configuration;
* subscriptions.

The control plane may remain healthy while parts of the data plane are degraded and vice versa.

---

## 3.3 Platform Layer

The platform layer provides the infrastructure on which both planes operate.

Responsibilities include:

* Kubernetes;
* container lifecycle;
* networking;
* CI/CD;
* secrets;
* configuration;
* databases;
* infrastructure provisioning;
* service discovery;
* internal observability;
* deployment tooling.

---

# 4. Major system domains

AstralCloud currently contains twelve high-level system domains.

These are architectural boundaries rather than final engineering teams or repositories.

## 4.1 Collection & Edge

Receives telemetry from customer environments.

Primary responsibilities:

* OTLP endpoints;
* HTTP/gRPC handling;
* TLS termination;
* API-key validation;
* request validation;
* tenant identification;
* rate limiting;
* regional routing.

Customer applications normally send telemetry through the Astral Collector.

---

## 4.2 Ingestion & Routing

Accepts validated telemetry from the edge and routes it into the internal streaming platform.

Responsibilities include:

* schema validation;
* tenant metadata enrichment;
* quotas;
* telemetry classification;
* Kafka production;
* routing rules;
* ingestion acknowledgements.

Heavy processing should not occur synchronously on the ingestion edge.

---

## 4.3 Streaming Backbone

Kafka provides the main asynchronous transport mechanism between high-volume systems.

Representative topic categories include:

* `metrics.raw`;
* `logs.raw`;
* `traces.raw`;
* processed telemetry streams;
* alert events;
* usage-metering events.

Multiple consumers may independently process the same underlying telemetry.

Consumer lag and partial consumer failure are therefore important operational concerns.

---

## 4.4 Telemetry Processing

Dedicated processing pipelines transform incoming telemetry into AstralCloud's internal representations.

### Metrics processing

Responsibilities may include:

* tag normalisation;
* aggregation;
* rollups;
* cardinality control.

### Log processing

Responsibilities may include:

* parsing;
* metadata enrichment;
* indexing preparation;
* trace correlation.

### Trace processing

Responsibilities may include:

* span validation;
* sampling;
* service discovery;
* dependency extraction;
* trace assembly.

---

## 4.5 Telemetry Storage

AstralCloud uses multiple storage technologies according to workload.

### ClickHouse

Primary hot analytical store for:

* logs;
* traces;
* metric datapoints;
* large-scale telemetry queries.

### Amazon S3

Used for:

* raw archival;
* cold telemetry;
* exports;
* backups;
* long-term objects.

### PostgreSQL

Used primarily for relational control-plane data.

### Redis

Used for selected:

* caches;
* sessions;
* rate limits;
* query-result caching;
* ephemeral coordination.

### Kafka

Used for durable event transport rather than primary query storage.

---

# 5. Query Platform

Customers do not query storage systems directly.

Telemetry queries pass through a dedicated query platform.

Conceptually:

```text
Web UI / API Client
        |
        v
Public API
        |
        v
Query Gateway
        |
        v
Query Planning / Validation
        |
        v
Telemetry Storage
        |
        v
Caching / Response Processing
        |
        v
Customer
```

The separation between storage and query layers means data may be correctly ingested and stored while still appearing incorrect or unavailable to customers.

---

# 6. Detection & Alerting

Customers configure monitors against their telemetry.

Example:

```text
p95(request.duration) > 800ms
for 5 minutes
```

The detection platform evaluates monitors using streaming and/or query-based evaluation.

Conceptually:

```text
Telemetry
    |
    v
Evaluation Engine
    |
    v
Monitor State
    |
    v
Alert Event
```

Alert evaluation operates independently enough that telemetry dashboards may remain healthy while alert processing is delayed or failing.

---

# 7. Incident & Notification Platform

Alert events may create or update incidents.

The incident platform stores information including:

* severity;
* status;
* responders;
* timelines;
* affected systems;
* related alerts;
* communications;
* postmortems.

Notifications may be delivered through:

* Slack;
* PagerDuty;
* email;
* webhooks.

Notification delivery is treated separately from detection so that a monitor can correctly fire while notification delivery fails.

---

# 8. Core SaaS Control Plane

The control plane contains ordinary SaaS functionality including:

* organisations;
* projects;
* users;
* teams;
* RBAC;
* authentication;
* SSO;
* API keys;
* dashboards;
* saved queries;
* monitor definitions;
* integrations;
* feature flags;
* retention policies.

PostgreSQL is the primary relational persistence layer for these systems.

---

# 9. Usage & Billing

AstralCloud uses usage-based pricing.

Usage events are generated from relevant platform activity.

Conceptually:

```text
Telemetry ingestion
        |
        v
Usage Events
        |
        v
Kafka
        |
        v
Metering
        |
        v
Aggregation
        |
        v
Usage Ledger
        |
        v
Billing
        |
        v
Payment Provider
```

Relevant usage categories may include:

* log ingestion volume;
* trace volume;
* metric volume;
* monitored workloads;
* retention;
* premium functionality.

Billing must support idempotency and accurate tenant attribution.

---

# 10. Product API & Web Platform

AstralCloud provides a browser-based customer application.

Current intended frontend stack:

* TypeScript;
* React.

Customer interfaces communicate with backend systems through APIs rather than directly accessing internal databases.

Conceptually:

```text
React Web Application
        |
        v
API Layer
        |
        +--> Query Platform
        +--> Control Plane
        +--> Incident Platform
        +--> Billing
        +--> Alerting
```

---

# 11. Platform Engineering

AstralCloud primarily operates on AWS.

Current intended platform technologies include:

* AWS;
* Amazon EKS;
* Kubernetes;
* Docker;
* Amazon ECR or equivalent container registry;
* Terraform;
* Helm;
* GitHub;
* GitHub Actions;
* Argo CD;
* AWS IAM;
* AWS networking;
* managed and/or Kubernetes-hosted data infrastructure.

## Containerisation

Application services are packaged using Docker.

A typical delivery path is:

```text
Source Code
    |
    v
Docker Build
    |
    v
Automated Tests
    |
    v
Container Registry
    |
    v
Deployment Configuration
    |
    v
Argo CD
    |
    v
EKS
```

Local development may use Docker Compose where appropriate.

Container configuration, Dockerfiles and image changes are legitimate sources of production incidents and should therefore be represented in the synthetic company history.

---

# 12. Customer telemetry flow

A typical customer telemetry path is:

```text
Customer Application
        |
        v
Astral Collector
        |
        | OTLP
        v
Global / Regional Edge
        |
        v
Telemetry Gateway
        |
        v
Validation & Tenant Routing
        |
        v
Kafka
   _____|_____
  |     |     |
  v     v     v
Metrics Logs Traces
  |     |     |
  v     v     v
Telemetry Processing
       |
       v
ClickHouse / S3
       |
       v
Query Platform
       |
       v
AstralCloud APIs
       |
       v
Web Application
```

Parallel consumers may use the same incoming telemetry for:

```text
Alerting
Usage metering
Service-map generation
Internal analytics
```

---

# 13. Multi-tenancy

Multi-tenancy is a foundational property of AstralCloud.

Telemetry should carry canonical tenant context such as:

```json
{
  "organisation_id": "org_7hd82",
  "project_id": "project_prod",
  "environment": "production",
  "service": "checkout-api",
  "region": "eu"
}
```

Tenant identity affects:

* routing;
* querying;
* access control;
* quotas;
* billing;
* storage;
* retention;
* aggregation.

This provides realistic failure modes including:

* incorrect tenant attribution;
* noisy neighbours;
* quota failures;
* billing errors;
* sharding problems;
* permission issues;
* potential tenant-isolation incidents.

---

# 14. Regional architecture

Heavy data-plane processing is regional.

The control plane is logically global.

Conceptually:

```text
              Global Control Plane
                    /       \
                   /         \
                  v           v
          EU Data Plane    US Data Plane
```

Exact AWS regions and failover behaviour remain undecided.

Regionalisation allows future scenarios involving:

* regional outages;
* data residency;
* configuration propagation;
* Kafka failures;
* regional deployments;
* asymmetric degradation;
* cross-region failover.

---

# 15. AstralCloud internal observability

AstralCloud dogfoods its own product.

The majority of internal application telemetry flows into a dedicated internal AstralCloud organisation.

```text
AstralCloud Production Services
              |
              v
      Internal AstralCloud Tenant
```

This allows engineers to use the same monitoring capabilities sold to customers.

However, AstralCloud must not depend entirely on itself to determine whether AstralCloud is healthy.

A minimal out-of-band observability path therefore exists for critical infrastructure.

Potential sources include AWS-native telemetry covering:

* EKS health;
* load balancers;
* critical Kafka infrastructure;
* network health;
* essential infrastructure logs.

This makes it possible to distinguish:

```text
"The production system failed"
```

from:

```text
"The system observing production failed."
```

---

# 16. Agent-investigation implications

The investigation agent must interact with evidence generated by these systems rather than accessing hidden canonical ground truth.

Potential evidence sources include:

* application source code;
* infrastructure code;
* Dockerfiles;
* Kubernetes manifests;
* Helm charts;
* Terraform;
* Git commits;
* pull requests;
* CI/CD runs;
* deployment records;
* logs;
* metrics;
* traces;
* alerts;
* incidents;
* engineering chat;
* tickets;
* runbooks;
* architecture documentation;
* AWS out-of-band telemetry.

Different evidence sources may be incomplete, outdated, incorrect or contradictory.

The agent must therefore perform investigation rather than merely summarising pre-linked information.

---

# 17. Current architectural decisions

The following are considered sufficiently established to use as canonical assumptions:

* three-plane architecture;
* multi-tenancy;
* AWS as primary cloud;
* Docker containerisation;
* Kubernetes/EKS;
* Terraform infrastructure-as-code;
* Helm;
* GitHub;
* GitHub Actions;
* Argo CD;
* Kafka streaming;
* ClickHouse telemetry storage;
* S3 archival;
* PostgreSQL relational persistence;
* Redis caching/ephemeral state;
* React/TypeScript web frontend;
* OpenTelemetry-compatible ingestion;
* regional data planes;
* logically global control plane;
* AstralCloud dogfooding;
* minimal out-of-band monitoring.

---

# 18. Deliberately undecided

The following should not yet be treated as canonical:

* exact microservice count;
* exact service names;
* exact repository boundaries;
* exact engineering teams;
* individual engineers;
* exact AWS regions;
* exact EKS topology;
* exact Kafka cluster topology;
* exact ClickHouse deployment topology;
* exact database schemas;
* exact deployment frequency;
* exact availability targets;
* exact disaster-recovery model.

These should emerge from later design stages rather than being invented prematurely.
