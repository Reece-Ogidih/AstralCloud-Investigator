# AstralCloud AWS & Kubernetes Infrastructure

**Version:** 0.1
**Status:** Working canonical infrastructure topology
**Last updated:** September 2026

---

# 1. Purpose

This document defines the production infrastructure topology used by AstralCloud.

It covers:

* AWS organisation and accounts;
* production regions;
* VPC topology;
* EKS clusters;
* Kubernetes namespaces;
* compute;
* Kafka;
* ClickHouse;
* PostgreSQL;
* Redis;
* S3;
* container infrastructure;
* ingress;
* secrets and identity;
* deployment;
* staging;
* observability;
* disaster recovery.

The goal is to model a realistic growth-stage production environment without reproducing hyperscaler-level infrastructure complexity.

---

# 2. AWS organisation

AstralCloud operates under AWS Organizations and separates major security and workload boundaries into dedicated AWS accounts.

The initial account structure is:

```text
AstralCloud AWS Organization

├── management
│
├── security
│
├── log-archive
│
├── shared-services
│
├── production-control
│
├── production-eu
│
├── production-us
│
└── staging
```

## `management`

Used primarily for AWS Organizations and account governance.

Production workloads do not run here.

## `security`

Contains central security services and security-team tooling.

Examples include:

* security findings;
* organisation-wide security configuration;
* audit tooling;
* selected threat-detection systems.

## `log-archive`

Receives immutable or restricted AWS audit logs.

Examples include:

* CloudTrail;
* selected AWS access logs;
* security audit records.

This is intentionally separate from AstralCloud's own observability platform.

## `shared-services`

Contains shared engineering infrastructure.

Examples include:

* container registry;
* selected CI/CD supporting resources;
* shared DNS resources;
* artifact storage.

## `production-control`

Hosts the global AstralCloud control-plane infrastructure.

## `production-eu`

Hosts the European telemetry data plane.

## `production-us`

Hosts the US telemetry data plane.

## `staging`

Hosts the shared pre-production environment.

---

# 3. Production regions

AstralCloud initially operates two primary production regions.

```text
EU
eu-west-1
Ireland

US
us-east-1
Northern Virginia
```

These regions represent separate customer telemetry residency boundaries.

Customer organisations are assigned a home telemetry region.

For example:

```text
Acme UK Ltd
    ↓
EU
    ↓
eu-west-1
```

while:

```text
Northstar Systems Inc
    ↓
US
    ↓
us-east-1
```

Customer telemetry is not automatically redirected between regions simply because another region is healthy.

This prevents disaster-recovery behaviour from accidentally violating data-residency expectations.

---

# 4. High-level production topology

```text
                           INTERNET
                              │
             ┌────────────────┴─────────────────┐
             │                                  │
             ▼                                  ▼

       Customer telemetry                 AstralCloud users
             │                                  │
             ▼                                  ▼

       Regional ingestion                 Global web/API
             │                                  │
             ▼                                  ▼

     ┌─────────────────┐              ┌─────────────────┐
     │ EU DATA PLANE   │              │ CONTROL PLANE   │
     │                 │              │                 │
     │ eu-west-1       │              │ primary: EU     │
     └─────────────────┘              │ DR: US          │
                                      └─────────────────┘

     ┌─────────────────┐
     │ US DATA PLANE   │
     │                 │
     │ us-east-1       │
     └─────────────────┘
```

The heavy telemetry systems remain regional.

The control plane is logically global.

---

# 5. VPC topology

Each production workload account contains a dedicated VPC.

Each production VPC spans three Availability Zones.

Conceptually:

```text
VPC

AZ-A
├── public subnet
├── application subnet
└── data subnet

AZ-B
├── public subnet
├── application subnet
└── data subnet

AZ-C
├── public subnet
├── application subnet
└── data subnet
```

## Public subnets

Used only where public routing is genuinely required.

Examples include:

* load balancers;
* NAT gateways where required.

Application workloads do not normally run directly in public subnets.

## Application subnets

Private subnets containing:

* EKS worker nodes;
* application pods;
* internal load balancers.

## Data subnets

More restricted private subnets containing systems such as:

* MSK brokers;
* ClickHouse nodes;
* Aurora/RDS;
* ElastiCache.

Security groups and network policies restrict access between layers.

---

# 6. Production EKS clusters

AstralCloud initially runs four production EKS clusters.

```text
production-eu
└── ac-prod-eu-data-01

production-us
└── ac-prod-us-data-01


production-control
├── ac-prod-control-eu-01
└── ac-prod-control-us-01
```

---

# 7. Regional data-plane clusters

The European cluster:

```text
ac-prod-eu-data-01
```

runs in:

```text
eu-west-1
```

The US cluster:

```text
ac-prod-us-data-01
```

runs in:

```text
us-east-1
```

Each hosts the regional components required to ingest, process, query and alert on customer telemetry.

Representative workloads include:

```text
telemetry-gateway
ingestion-router
tenant-config-sync

metrics-processor
logs-processor
traces-processor
service-map-worker

query-gateway
query-planner

metrics-api
logs-api
apm-api

alert-evaluator
```

These clusters are independent.

A Kubernetes-level failure in the EU data-plane cluster should not automatically affect the US data plane.

---

# 8. Control-plane clusters

The principal control-plane cluster is:

```text
ac-prod-control-eu-01
```

in `eu-west-1`.

A disaster-recovery cluster exists in:

```text
ac-prod-control-us-01
```

in `us-east-1`.

Control-plane workloads include:

```text
identity-service
organisation-service
api-key-service

monitor-service

incident-service
notification-service

dashboard-service

billing-service
entitlement-service

public-api
```

The US control-plane cluster normally operates in a reduced or standby capacity for functionality requiring globally writable relational state.

This avoids pretending AstralCloud has solved transparent multi-master relational writes across continents.

---

# 9. Kubernetes namespace model

Namespaces group related workloads while ownership remains defined at the service/team level.

A regional data-plane cluster approximately contains:

```text
edge
ingestion
telemetry-metrics
telemetry-logs
telemetry-traces
query
alerting
storage-ops

platform-system
security-system
observability-system
```

The control-plane cluster approximately contains:

```text
identity
control-plane
billing
incidents
web

platform-system
security-system
observability-system
```

Namespaces are not treated as sufficient security boundaries by themselves.

IAM, network controls and workload identities are also used.

---

# 10. EKS compute

EKS clusters use multiple worker-node classes rather than placing every workload onto identical machines.

Representative node groups include:

```text
system
general
compute
memory
```

## System nodes

Reserved for important cluster-level workloads such as:

* DNS;
* controllers;
* Argo CD components;
* selected security agents.

## General nodes

Run ordinary APIs and lightweight workers.

## Compute nodes

Run CPU-heavy telemetry processors.

## Memory nodes

Support workloads with higher memory requirements.

Workloads use:

* resource requests;
* resource limits;
* affinity;
* topology-spread constraints;
* PodDisruptionBudgets;
* autoscaling.

Incorrect values in any of these become legitimate sources of production incidents.

---

# 11. Container model

Server-side application workloads are packaged with Docker.

A service build produces an immutable container image.

Images are tagged with useful human-readable metadata but deployments ultimately refer to immutable image digests.

Example:

```text
astralcloud/query-planner

git commit:
8fa42bc

container:
query-planner:2026.09.18-8fa42bc

digest:
sha256:...
```

This gives AstralCloud a deterministic relationship between:

```text
Git commit
    ↓
Docker image
    ↓
deployment
    ↓
running pod
```

This relationship will later be highly valuable to the investigator.

---

# 12. Container registry

Amazon ECR is used as AstralCloud's production container registry.

Repository access is controlled through AWS IAM.

Application teams may push images through CI but production workloads receive only pull permissions.

The registry forms part of AstralCloud's software supply chain.

Relevant failure classes include:

```text
incorrect image tag
wrong image digest
missing image
bad base image
vulnerable dependency
registry permission failure
```

---

# 13. Kafka

AstralCloud uses Amazon Managed Streaming for Apache Kafka.

Each telemetry region owns an independent production MSK cluster.

Conceptually:

```text
EU

telemetry-gateway
      ↓
ingestion-router
      ↓
EU MSK
      ↓
processors


US

telemetry-gateway
      ↓
ingestion-router
      ↓
US MSK
      ↓
processors
```

Kafka brokers span multiple Availability Zones.

The Streaming Platform team owns:

* cluster capacity;
* broker configuration;
* topic lifecycle;
* replication configuration;
* schema tooling;
* Kafka operational standards.

Application teams own the behaviour of their consumers and producers.

Representative topics include:

```text
metrics.raw
logs.raw
traces.raw

metrics.processed
logs.processed
traces.processed

usage.events
alert.events
control.tenant-config
```

Actual naming will be refined later.

---

# 14. ClickHouse

High-volume telemetry is stored primarily in dedicated regional ClickHouse clusters.

ClickHouse does **not** run inside the general-purpose application EKS cluster.

Instead, it runs on dedicated AWS compute managed by the Storage Platform team.

Conceptually:

```text
EU ClickHouse Cluster
├── nodes in AZ-A
├── nodes in AZ-B
└── nodes in AZ-C
```

with an equivalent US deployment.

The architecture uses:

* replication;
* sharding;
* EBS-backed local storage where appropriate;
* S3 for archival/object-backed workflows.

ClickHouse configuration and schema definitions primarily live in:

```text
astralcloud-data-platform
```

while AWS compute/network resources are primarily provisioned from:

```text
astralcloud-platform
```

This separation intentionally creates cross-repository infrastructure dependencies.

---

# 15. Object storage

Amazon S3 stores:

* cold telemetry;
* archived telemetry;
* exports;
* selected backups;
* processing artifacts.

Each telemetry region has separate regional buckets.

Customer telemetry is not automatically copied into the other production geography.

Example conceptual buckets:

```text
astralcloud-prod-eu-telemetry-archive
astralcloud-prod-us-telemetry-archive
```

Real generated infrastructure names may differ.

---

# 16. Relational database

The global control plane uses PostgreSQL-compatible Amazon Aurora.

The primary writer runs in:

```text
eu-west-1
```

with disaster-recovery capacity in:

```text
us-east-1
```

The relational database stores state such as:

```text
organisations
users
projects
RBAC

API-key metadata

dashboards
monitor definitions

incidents

billing configuration
subscriptions
entitlements
```

Control-plane services do not each receive an independent database merely to claim microservice purity.

Instead, database ownership is separated through schemas and application boundaries where appropriate.

This avoids excessive operational complexity while preserving service ownership.

---

# 17. Control-plane disaster recovery

Aurora Global Database provides cross-region replication for important control-plane relational state.

Normal operation uses the EU writer.

The US environment contains a secondary copy that can participate in regional disaster recovery.

Control-plane regional failover is an explicit operational event rather than an invisible assumption.

This means AstralCloud can later have incident scenarios involving:

```text
replication lag
failover
DNS propagation
read/write confusion
stale control-plane state
```

Aurora's global-database model uses a primary writable region and secondary read-only regions under normal operation, which aligns well with this design.

---

# 18. Redis

Amazon ElastiCache for Redis is used for ephemeral and performance-sensitive state.

Regional data-plane usage includes:

* tenant-configuration cache;
* API-key cache;
* query-result cache;
* rate-limiting state.

Control-plane use may include:

* sessions;
* short-lived caches.

Redis is not treated as the canonical source for important business state.

This distinction allows incidents where:

```text
PostgreSQL is correct

but

Redis contains stale state
```

---

# 19. Public telemetry ingress

Customers send telemetry to region-specific endpoints.

For example:

```text
otlp.eu.astralcloud.io

otlp.us.astralcloud.io
```

These endpoints map into their respective regional environments.

Conceptually:

```text
DNS
 ↓
AWS Network Load Balancer
 ↓
telemetry-gateway pods
```

gRPC and HTTP ingestion are both supported.

Regional endpoints intentionally avoid automatically sending European telemetry into the US during ordinary failures.

---

# 20. Web and API ingress

Customer web traffic uses:

```text
app.astralcloud.io
```

Static web assets are distributed using:

```text
CloudFront
```

with origin content stored in S3 or an equivalent build-artifact origin.

Customer API requests use:

```text
api.astralcloud.io
```

and route toward the active control-plane environment.

AWS WAF protects public HTTP application endpoints where appropriate.

---

# 21. Internal service networking

Most service-to-service communication remains private.

Services use Kubernetes-native DNS and internal load balancing where appropriate.

For example:

```text
public-api
    ↓
organisation-service

monitor-service
    ↓
PostgreSQL

apm-api
    ↓
query-gateway
```

Data services such as MSK, Redis, Aurora and ClickHouse are inaccessible directly from the public internet.

---

# 22. AWS workload identity

Kubernetes workloads use AWS workload identities rather than static AWS access keys embedded in application configuration.

Individual services receive only the AWS permissions required for their responsibilities.

Examples:

```text
archive-worker
    ↓
write selected S3 paths


notification-service
    ↓
read notification-provider secret


tenant-config-sync
    ↓
access relevant Redis resources
```

Permission configuration is therefore another legitimate incident source.

---

# 23. Secrets

Production secrets are stored outside Git.

AWS Secrets Manager stores secrets such as:

* third-party API credentials;
* database credentials where required;
* webhook credentials;
* payment-provider credentials.

Kubernetes workloads receive authorised secrets at runtime.

Git contains references and configuration, not plaintext production credentials.

---

# 24. Infrastructure as code

Terraform defines AWS infrastructure.

Representative structure:

```text
astralcloud-platform/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   ├── msk/
    │   ├── aurora/
    │   ├── redis/
    │   ├── s3/
    │   ├── ecr/
    │   └── iam/
    │
    └── environments/
        ├── production-eu/
        ├── production-us/
        ├── production-control/
        └── staging/
```

Terraform changes require reviewed pull requests.

Production infrastructure should not normally be manually changed through the AWS Console.

Emergency manual modifications must eventually be reconciled back into infrastructure as code.

Configuration drift can therefore become a legitimate historical event.

---

# 25. Kubernetes packaging

Application deployment configuration uses Helm.

Each application repository owns its base deployment configuration.

For example:

```text
astralcloud-query/
└── deploy/
    └── query-planner/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

Environment-specific configuration lives in:

```text
astralcloud-platform
```

For example:

```text
environments/
└── production-eu/
    └── query-planner.yaml
```

The effective production configuration is therefore composed from both repositories.

---

# 26. GitOps and Argo CD

AstralCloud uses Argo CD to reconcile Kubernetes production state from Git.

The central principle is:

> Git records the intended production deployment state.

A running deployment therefore has an auditable Git origin.

Conceptually:

```text
Application repository
        │
        │ merge
        ▼
GitHub Actions
        │
        ├── tests
        ├── security checks
        └── Docker build
                │
                ▼
               ECR
                │
                ▼
Deployment promotion
                │
                ▼
astralcloud-platform
environment config
                │
                ▼
              Argo CD
                │
                ▼
               EKS
```

---

# 27. Deployment promotion

Merging application code does not automatically imply immediate production deployment.

A typical change progresses:

```text
feature branch
     ↓
pull request
     ↓
CI
     ↓
merge to main
     ↓
Docker image produced
     ↓
staging
     ↓
automated/integration verification
     ↓
production promotion
     ↓
platform repo updated
     ↓
Argo CD deployment
```

The production promotion records the immutable image digest.

For example:

```yaml
queryPlanner:
  image:
    repository: ...
    digest: sha256:1af...
```

A production deployment can therefore be correlated with both:

```text
application commit

and

platform repository promotion commit
```

---

# 28. Why deployment state lives in Git

This is particularly important for AstralCloud Investigator.

Suppose:

```text
13:41 application commit merged

13:47 Docker image built

14:03 production promotion PR merged

14:05 Argo CD sync begins

14:07 new pods become ready

14:19 latency starts rising

14:26 customer alert triggered
```

The investigator can reconstruct the timeline from independent evidence.

It does not need to trust a synthetic sentence saying:

> The deployment caused latency.

---

# 29. Deployment strategies

Most ordinary services initially use rolling Kubernetes deployments.

Important customer-facing workloads may additionally support controlled canary deployment.

For example:

```text
5%
 ↓
25%
 ↓
50%
 ↓
100%
```

The exact rollout mechanism is deliberately not yet standardised across every service.

Deployment strategy may evolve historically.

This gives AstralCloud realistic differences between older and newer systems.

---

# 30. Staging environment

AstralCloud maintains a persistent staging environment in `eu-west-1`.

Staging contains a reduced-scale representation of production.

It includes:

```text
EKS
Kafka
ClickHouse
PostgreSQL
Redis
S3
```

but with substantially smaller capacity.

Staging is intended for:

* integration testing;
* release verification;
* schema migration validation;
* infrastructure testing;
* incident reproduction.

It is **not assumed to perfectly reproduce production behaviour**.

This is important.

A performance regression may pass staging because production has:

```text
far greater data volume
higher cardinality
more tenants
different traffic distribution
larger Kafka partitions
```

Therefore:

> "It worked in staging."

is evidence, not proof.

---

# 31. Local development

Developers do not reproduce the complete production platform locally.

Docker Compose provides lightweight local dependencies where practical.

A developer working on `query-planner`, for example, might run:

```text
query-planner
ClickHouse
Redis
mock control-plane API
```

rather than:

```text
27 services
Kafka production topology
four EKS clusters
Aurora Global Database
```

Integration environments provide broader validation.

---

# 32. Production observability

AstralCloud services emit:

```text
structured logs
metrics
distributed traces
deployment metadata
health information
```

primarily into AstralCloud itself.

Each workload emits standard metadata including:

```text
service
version
git_sha
environment
region
cluster
namespace
pod
```

For example:

```json
{
  "service": "query-planner",
  "version": "2026.09.18",
  "git_sha": "8fa42bc",
  "environment": "production",
  "region": "eu-west-1",
  "cluster": "ac-prod-eu-data-01"
}
```

This allows telemetry to be directly correlated with source and deployments.

---

# 33. Out-of-band observability

AstralCloud deliberately keeps a minimal independent visibility path outside its own telemetry product.

AWS-native telemetry provides essential information such as:

```text
EKS state
load-balancer health
AWS infrastructure events
critical platform logs
selected MSK health
CloudTrail
```

The intent is not to duplicate AstralCloud's entire monitoring platform.

It exists to answer:

> Is production broken, or is our ability to observe production broken?

---

# 34. Security and audit boundary

AWS CloudTrail and selected security/audit information are retained outside the normal AstralCloud telemetry pipeline.

Security-sensitive records are centralised in the dedicated audit/security accounts.

The investigation agent may eventually receive restricted tools for some of this information depending on scenario permissions.

It should not automatically have unrestricted access to every security source.

---

# 35. Failure domains

The topology intentionally creates several failure boundaries.

Examples include:

```text
single pod
single service
namespace
EKS node
Availability Zone
EKS cluster
Kafka topic
Kafka cluster
ClickHouse shard
regional data plane
control plane
AWS region
```

Symptoms should depend on which boundary fails.

For example:

```text
EU Kafka problem
```

should not automatically generate:

```text
US telemetry failure
```

unless another dependency genuinely connects them.

---

# 36. Example infrastructure investigation

Suppose customers report:

> EU APM queries are taking 15–20 seconds.

Initial telemetry shows:

```text
apm-api healthy
query-gateway healthy
ClickHouse CPU elevated
```

A recent `astralcloud-query` release contains no obvious problem.

Investigation eventually discovers a platform change:

```text
astralcloud-platform

production-eu/query-planner.yaml
```

changed:

```diff
 resources:
   limits:
-    cpu: "4"
+    cpu: "1"
```

The sequence becomes:

```text
platform Git commit
      ↓
Argo CD sync
      ↓
query-planner rollout
      ↓
CPU throttling
      ↓
query backlog
      ↓
ClickHouse query concurrency changes
      ↓
APM latency
```

The agent has to connect:

```text
Git
Kubernetes
deployment history
metrics
logs
service dependencies
```

rather than merely summarising an incident record.

---

# 37. Example regional investigation

Suppose:

```text
EU customers:
missing traces

US customers:
healthy
```

This immediately makes certain hypotheses less likely.

Shared global systems remain possible causes, but regional components become more suspicious:

```text
EU telemetry gateway
EU ingestion router
EU MSK
EU traces processor
EU ClickHouse
EU query path
```

Regional topology therefore becomes meaningful evidence for the agent.

---

# 38. Canonical production clusters

The initial canonical Kubernetes clusters are:

```text
ac-prod-eu-data-01
ac-prod-us-data-01

ac-prod-control-eu-01
ac-prod-control-us-01

ac-staging-eu-01
```

Further clusters should only be introduced when scale or isolation genuinely requires them.

---

# 39. Canonical data infrastructure

Production currently assumes:

```text
EU

MSK
ClickHouse
ElastiCache
S3


US

MSK
ClickHouse
ElastiCache
S3


GLOBAL CONTROL PLANE

Aurora PostgreSQL
ElastiCache
S3
```

Some infrastructure may later be decomposed further where implementation requires it.

---

# 40. Infrastructure repositories

Responsibility is divided primarily between:

```text
astralcloud-platform
```

for AWS, EKS and environment configuration,

and:

```text
astralcloud-data-platform
```

for Kafka/ClickHouse schemas, configuration and operational tooling.

This distinction must remain visible in ownership and Git history.

---

# 41. Current canonical decisions

The following are now treated as established:

* AWS Organizations;
* multi-account environment;
* production EU and US data regions;
* `eu-west-1` and `us-east-1`;
* three-AZ production VPC topology;
* separate regional data-plane EKS clusters;
* primary + DR control-plane EKS clusters;
* Docker application packaging;
* ECR;
* Amazon MSK;
* dedicated ClickHouse infrastructure;
* Aurora PostgreSQL for control-plane relational state;
* ElastiCache Redis;
* regional S3 telemetry storage;
* Terraform;
* Helm;
* Argo CD;
* GitHub Actions;
* Git-based production promotion;
* immutable image digests;
* persistent staging;
* limited out-of-band AWS observability.

---

# 42. Deliberately undecided

The following remain open until deeper implementation requires them:

* exact VPC CIDR ranges;
* exact EC2 instance families;
* exact node-group sizes;
* exact Kafka broker counts;
* exact Kafka partition counts;
* exact ClickHouse shard/replica count;
* exact Aurora instance classes;
* exact autoscaling thresholds;
* exact canary implementation;
* service mesh adoption;
* exact network-policy engine;
* exact secret-injection mechanism;
* exact disaster-recovery RTO/RPO;
* exact enterprise cross-region telemetry options.

These details should not be invented until they affect a meaningful implementation or scenario.
