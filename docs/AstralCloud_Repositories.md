# AstralCloud Repository Architecture

**Version:** 0.1
**Status:** Working canonical repository model
**Last updated:** September 2026

---

# 1. Purpose

This document defines how AstralCloud's production systems are divided into source-code repositories.

Repository boundaries are designed around:

* coherent technical domains;
* meaningful ownership;
* shared implementation concerns;
* independent release patterns;
* manageable Git history;
* realistic cross-team collaboration;
* practical synthetic-code generation.

AstralCloud does **not** use either extreme of:

* one repository per microservice; or
* one company-wide monorepo.

Instead, it uses a domain-oriented multi-repository model.

The initial design contains **10 principal repositories**.

---

# 2. Repository overview

| Repository                  | Primary contents                                     | Main technologies     |
| --------------------------- | ---------------------------------------------------- | --------------------- |
| `astralcloud-collector`     | Customer-side collector                              | Go                    |
| `astralcloud-ingestion`     | Telemetry edge and routing                           | Go                    |
| `astralcloud-telemetry`     | Metrics, logs and trace processing                   | Go                    |
| `astralcloud-data-platform` | Kafka/ClickHouse platform and archival               | Go, Terraform, config |
| `astralcloud-query`         | Shared query layer and telemetry APIs                | Go                    |
| `astralcloud-alerting`      | Monitors, alerting, incidents, notifications         | Go, Kotlin, Python    |
| `astralcloud-control-plane` | Identity, organisations, API keys, dashboards        | Kotlin                |
| `astralcloud-billing`       | Metering, entitlements and billing                   | Go, Kotlin            |
| `astralcloud-web`           | Public API and React application                     | TypeScript            |
| `astralcloud-platform`      | AWS, EKS, CI/CD and shared deployment infrastructure | Terraform, Helm, YAML |

These repositories represent the simulated company's production repositories.

They are distinct from the real portfolio repository:

```text
astralcloud-investigator
```

which contains the simulation, investigator, generators and evaluation system.

---

# 3. `astralcloud-collector`

**Primary owner:** Collectors & Edge

Contains:

```text
astral-collector
```

This receives its own repository because it is fundamentally different from AstralCloud's server-side software.

It executes inside customer environments and therefore has:

* independent releases;
* downloadable/containerised versions;
* backward-compatibility requirements;
* customer-facing release notes;
* its own testing requirements.

Representative structure:

```text
astralcloud-collector/
├── cmd/
│   └── collector/
├── internal/
│   ├── config/
│   ├── discovery/
│   ├── batching/
│   ├── buffering/
│   ├── export/
│   └── metadata/
├── pkg/
├── deploy/
│   ├── kubernetes/
│   └── docker/
├── tests/
├── Dockerfile
├── go.mod
└── README.md
```

This repository can contain customer-visible version tags such as:

```text
v3.18.0
v3.18.1
v3.19.0
```

---

# 4. `astralcloud-ingestion`

**Primary owners:**

* Collectors & Edge
* Ingestion & Routing

Contains:

```text
telemetry-gateway
ingestion-router
tenant-config-sync
```

These services are grouped because they form the tightly coupled telemetry ingress path and share:

* tenant metadata models;
* ingestion schemas;
* authentication primitives;
* rate-limiting libraries;
* OTLP-related utilities;
* Kafka producer infrastructure.

Representative structure:

```text
astralcloud-ingestion/
├── services/
│   ├── telemetry-gateway/
│   ├── ingestion-router/
│   └── tenant-config-sync/
│
├── internal/
│   ├── auth/
│   ├── tenancy/
│   ├── otlp/
│   ├── kafka/
│   └── ratelimit/
│
├── deploy/
├── tests/
├── go.mod
└── README.md
```

Ownership within the repository is divided using `CODEOWNERS`.

For example:

```text
/services/telemetry-gateway/     @collectors-edge
/services/ingestion-router/      @ingestion-routing
/services/tenant-config-sync/    @ingestion-routing
```

This creates realistic cross-team pull requests without forcing artificial repository separation.

---

# 5. `astralcloud-telemetry`

**Primary owners:**

* Metrics
* Logs
* APM & Tracing

Contains:

```text
metrics-processor
logs-processor
traces-processor
service-map-worker
```

These services share a common conceptual responsibility:

> Transform raw telemetry into AstralCloud's internal analytical representation.

They also share utilities for:

* metadata;
* tenant context;
* Kafka consumers;
* ClickHouse writes;
* OpenTelemetry structures;
* telemetry timestamps;
* usage-event generation.

Representative structure:

```text
astralcloud-telemetry/
├── services/
│   ├── metrics-processor/
│   ├── logs-processor/
│   ├── traces-processor/
│   └── service-map-worker/
│
├── internal/
│   ├── kafka/
│   ├── clickhouse/
│   ├── tenancy/
│   ├── telemetry/
│   └── usage/
│
├── deploy/
├── tests/
└── go.mod
```

This repository intentionally has multiple team owners.

That allows genuine situations where a change to shared telemetry code introduced by one team affects another team's processor.

---

# 6. `astralcloud-data-platform`

**Primary owners:**

* Streaming Platform
* Storage Platform

Contains:

```text
archive-worker
Kafka platform configuration
ClickHouse configuration
schemas
storage migrations
data lifecycle tooling
streaming operational tooling
```

Representative structure:

```text
astralcloud-data-platform/
├── services/
│   └── archive-worker/
│
├── kafka/
│   ├── topics/
│   ├── schemas/
│   ├── quotas/
│   └── tooling/
│
├── clickhouse/
│   ├── schemas/
│   ├── migrations/
│   ├── materialized_views/
│   └── config/
│
├── storage/
│   └── retention/
│
├── tools/
└── deploy/
```

This repository is particularly important for investigations.

A production incident may originate from:

```text
application code          ✗
Kafka topic config        ✓

or

application code          ✗
ClickHouse migration      ✓
```

The investigator therefore cannot assume every root cause lives in a conventional service repository.

---

# 7. `astralcloud-query`

**Primary owners:**

* Query Platform
* Metrics
* Logs
* APM & Tracing

Contains:

```text
query-gateway
query-planner
metrics-api
logs-api
apm-api
```

Representative structure:

```text
astralcloud-query/
├── services/
│   ├── query-gateway/
│   ├── query-planner/
│   ├── metrics-api/
│   ├── logs-api/
│   └── apm-api/
│
├── internal/
│   ├── querylang/
│   ├── planner/
│   ├── caching/
│   ├── tenancy/
│   └── clickhouse/
│
├── deploy/
├── tests/
└── go.mod
```

The shared query language and query execution libraries live alongside the services using them.

A shared query-library modification can therefore affect:

```text
Metrics
Logs
APM
```

simultaneously.

---

# 8. `astralcloud-alerting`

**Primary owners:**

* Detection & Alerting
* Incidents & Notifications

Contains:

```text
monitor-service
alert-evaluator
incident-service
notification-service
```

This repository is intentionally polyglot.

Indicative structure:

```text
astralcloud-alerting/
├── services/
│   ├── monitor-service/          # Kotlin
│   ├── alert-evaluator/          # Go
│   ├── incident-service/         # Python
│   └── notification-service/     # Python
│
├── contracts/
│   ├── alert-events/
│   └── incident-events/
│
├── deploy/
├── integration-tests/
└── README.md
```

Services retain their own language-specific build systems.

A top-level build script or CI workflow coordinates repository-level integration testing.

This is preferable to introducing a complex monorepo build system purely for the sake of the simulation.

---

# 9. `astralcloud-control-plane`

**Primary owners:**

* Identity & Organisations
* Web Experience

Contains:

```text
identity-service
organisation-service
api-key-service
dashboard-service
```

Primary language:

```text
Kotlin / Spring Boot
```

Representative structure:

```text
astralcloud-control-plane/
├── services/
│   ├── identity-service/
│   ├── organisation-service/
│   ├── api-key-service/
│   └── dashboard-service/
│
├── libraries/
│   ├── auth/
│   ├── database/
│   ├── events/
│   └── tenancy/
│
├── deploy/
├── integration-tests/
├── build.gradle.kts
└── settings.gradle.kts
```

A Gradle multi-project build makes shared control-plane libraries realistic and manageable.

---

# 10. `astralcloud-billing`

**Primary owner:** Billing & Metering

Contains:

```text
usage-aggregator
billing-service
entitlement-service
```

Indicative structure:

```text
astralcloud-billing/
├── services/
│   ├── usage-aggregator/       # Go
│   ├── billing-service/        # Kotlin
│   └── entitlement-service/    # Kotlin
│
├── contracts/
│   └── usage-events/
│
├── deploy/
├── integration-tests/
└── README.md
```

These components share a business domain even though implementation languages differ.

Keeping them together makes historical relationships between:

```text
usage collection
       ↓
entitlements
       ↓
pricing
       ↓
billing
```

visible in a single repository history.

---

# 11. `astralcloud-web`

**Primary owner:** Web Experience

Contains:

```text
public-api
web-app
```

Representative structure:

```text
astralcloud-web/
├── apps/
│   ├── web/
│   └── public-api/
│
├── packages/
│   ├── ui/
│   ├── api-client/
│   ├── auth/
│   └── observability/
│
├── tests/
├── package.json
└── README.md
```

A JavaScript/TypeScript workspace can be used.

The React frontend and backend-for-frontend share generated API types and customer-facing contracts.

This repository provides opportunities for genuine frontend, backend and API-contract incidents.

---

# 12. `astralcloud-platform`

**Primary owners:**

* Developer Platform
* Site Reliability Engineering
* Security Engineering

This repository contains shared production infrastructure rather than business application source code.

Representative structure:

```text
astralcloud-platform/
├── terraform/
│   ├── modules/
│   │   ├── eks/
│   │   ├── networking/
│   │   ├── iam/
│   │   ├── rds/
│   │   ├── redis/
│   │   └── s3/
│   │
│   └── environments/
│       ├── production-eu/
│       ├── production-us/
│       └── staging/
│
├── argocd/
│   ├── applications/
│   └── application-sets/
│
├── helm/
│   ├── base-service/
│   └── shared/
│
├── environments/
│   ├── production-eu/
│   ├── production-us/
│   └── staging/
│
├── github-actions/
├── base-images/
├── policies/
└── README.md
```

This repository is a major investigation target.

Examples of root causes include:

```text
CPU limit change
memory limit change
IAM policy regression
bad network rule
EKS configuration
secret reference
autoscaling configuration
Argo CD change
regional environment override
```

---

# 13. Application configuration ownership

AstralCloud uses a split configuration model.

Application repositories own:

```text
Dockerfile
service defaults
service Helm chart
health configuration
application-level deployment settings
```

The platform repository owns:

```text
environment-specific overrides
cluster infrastructure
Argo CD applications
networking
IAM
shared infrastructure
regional configuration
```

For example:

```text
astralcloud-query
└── deploy/
    └── query-planner/
        └── values.yaml

astralcloud-platform
└── environments/
    └── production-eu/
        └── query-planner.yaml
```

This separation creates realistic cross-repository failure scenarios.

A service's application repository might specify:

```yaml
resources:
  limits:
    cpu: "2000m"
```

while the production environment repository overrides it:

```yaml
resources:
  limits:
    cpu: "500m"
```

An investigator therefore needs to inspect both application code and deployed configuration.

---

# 14. Complete service mapping

| Production component   | Repository                  |
| ---------------------- | --------------------------- |
| `astral-collector`     | `astralcloud-collector`     |
| `telemetry-gateway`    | `astralcloud-ingestion`     |
| `ingestion-router`     | `astralcloud-ingestion`     |
| `tenant-config-sync`   | `astralcloud-ingestion`     |
| `metrics-processor`    | `astralcloud-telemetry`     |
| `logs-processor`       | `astralcloud-telemetry`     |
| `traces-processor`     | `astralcloud-telemetry`     |
| `service-map-worker`   | `astralcloud-telemetry`     |
| `archive-worker`       | `astralcloud-data-platform` |
| `query-gateway`        | `astralcloud-query`         |
| `query-planner`        | `astralcloud-query`         |
| `metrics-api`          | `astralcloud-query`         |
| `logs-api`             | `astralcloud-query`         |
| `apm-api`              | `astralcloud-query`         |
| `monitor-service`      | `astralcloud-alerting`      |
| `alert-evaluator`      | `astralcloud-alerting`      |
| `incident-service`     | `astralcloud-alerting`      |
| `notification-service` | `astralcloud-alerting`      |
| `identity-service`     | `astralcloud-control-plane` |
| `organisation-service` | `astralcloud-control-plane` |
| `api-key-service`      | `astralcloud-control-plane` |
| `dashboard-service`    | `astralcloud-control-plane` |
| `usage-aggregator`     | `astralcloud-billing`       |
| `billing-service`      | `astralcloud-billing`       |
| `entitlement-service`  | `astralcloud-billing`       |
| `public-api`           | `astralcloud-web`           |
| `web-app`              | `astralcloud-web`           |

Infrastructure components are primarily represented through:

```text
astralcloud-data-platform
astralcloud-platform
```

rather than being treated as application services.

---

# 15. GitHub organisation model

In the fictional production company, the repositories conceptually live within an AstralCloud GitHub organisation:

```text
github.com/astralcloud/
```

For example:

```text
astralcloud/astralcloud-ingestion
astralcloud/astralcloud-query
astralcloud/astralcloud-web
```

This namespace is conceptual unless the repositories are later intentionally published.

During local simulation, equivalent repositories are materialised under the investigator project.

For example:

```text
.generated/
└── repos/
    └── astralcloud/
        ├── astralcloud-collector/
        ├── astralcloud-ingestion/
        ├── astralcloud-telemetry/
        ├── astralcloud-data-platform/
        ├── astralcloud-query/
        ├── astralcloud-alerting/
        ├── astralcloud-control-plane/
        ├── astralcloud-billing/
        ├── astralcloud-web/
        └── astralcloud-platform/
```

Every directory is a genuine independent Git repository with its own `.git` history.

The outer `astralcloud-investigator` repository ignores `.generated/`.

---

# 16. Relationship with `astralcloud-investigator`

The real project repository contains the machinery required to reproduce the simulated company.

A future structure may resemble:

```text
astralcloud-investigator/
│
├── docs/
│
├── src/
│   └── astral_agent/
│
├── astralcloud/
│   ├── world/
│   ├── history/
│   │   ├── events/
│   │   └── patches/
│   │
│   ├── knowledge/
│   │
│   └── repo_seeds/
│       ├── astralcloud-collector/
│       ├── astralcloud-ingestion/
│       ├── astralcloud-telemetry/
│       ├── astralcloud-data-platform/
│       ├── astralcloud-query/
│       ├── astralcloud-alerting/
│       ├── astralcloud-control-plane/
│       ├── astralcloud-billing/
│       ├── astralcloud-web/
│       └── astralcloud-platform/
│
├── generators/
├── scenarios/
├── evals/
└── .generated/
```

`repo_seeds/` contains source material used to construct the simulated repositories.

It does **not** contain the final generated `.git` histories.

---

# 17. Git-history materialisation

The target Git histories should be deterministic and reproducible.

Conceptually:

```text
Repository seed
      │
      ▼
Initial commits
      │
      ▼
Normal development history
      │
      ▼
Feature patches
      │
      ▼
Infrastructure changes
      │
      ▼
Bug-introducing changes
      │
      ▼
Incident
      │
      ▼
Investigation
      │
      ▼
Fix / revert
```

The generator controls metadata such as:

```text
author
committer
timestamp
branch
commit message
changed files
pull request association
release
deployment
```

This ensures that Git history remains synchronised with the broader simulated company history.

---

# 18. Source-code history strategy

AstralCloud's source history should not be generated as thousands of unrelated LLM-created commits.

Instead, source generation follows a controlled approach.

The canonical company history determines important engineering events.

For an event such as:

```text
EVT-0417
query-planner optimisation merged
```

the corresponding history definition may specify:

```text
repository: astralcloud-query
branch: feature/query-cache-planning
author: emma-clarke
files:
  - internal/planner/cache.go
  - internal/planner/planner.go
tests:
  - internal/planner/cache_test.go
```

The generator applies the corresponding source patch and creates genuine Git history.

Background development commits may be generated separately to make repositories feel lived-in, but they must not contradict canonical events.

---

# 19. Why not 27 repositories?

Using one repository per service would create several problems:

* unrealistic administrative overhead for the simulated company;
* excessive CI configuration;
* difficult local generation;
* shallow histories;
* duplicated libraries;
* excessive tool calls for the investigator;
* unnecessary project complexity.

Repository boundaries therefore group services that genuinely share domain models and implementation concerns.

---

# 20. Why not one monorepo?

A single company-wide monorepo would make simulation easier but remove useful investigation complexity.

AstralCloud benefits from separate histories for:

```text
application code
data-platform configuration
cloud infrastructure
frontend code
collector releases
```

Cross-repository dependencies are also important to the investigation task.

For example:

```text
astralcloud-query
      │
      │ application healthy
      ▼
astralcloud-platform
      │
      │ production CPU override changed
      ▼
query latency incident
```

This would be substantially less interesting if every artifact existed in one repository and one change history.

---

# 21. Branch and pull-request model

AstralCloud uses a largely trunk-based workflow.

The default branch is:

```text
main
```

Engineers create short-lived feature and fix branches and merge changes through reviewed pull requests.

Representative branches include:

```text
feature/query-cache
fix/tenant-routing
chore/clickhouse-retention
hotfix/trace-sampling
```

Protected production repositories generally require:

* successful CI;
* code review;
* relevant CODEOWNER approval for sensitive areas.

Emergency changes may use expedited procedures but still leave an auditable Git history.

Exact merge strategy will be defined later when synthetic Git history generation is implemented.

---

# 22. Production realism requirements

A materialised repository should contain enough real implementation to support genuine investigation.

The investigator should eventually be able to perform operations such as:

```text
git log
git show
git diff
git blame
git branch
git tag
```

and source-level search.

It should encounter genuine:

```text
application code
tests
Dockerfiles
configuration
Helm values
Terraform
schemas
migrations
CI configuration
documentation
```

Incident-causing commits should modify real behaviour wherever practical.

---

# 23. Current canonical repository set

The initial canonical set is therefore:

```text
astralcloud-collector
astralcloud-ingestion
astralcloud-telemetry
astralcloud-data-platform
astralcloud-query
astralcloud-alerting
astralcloud-control-plane
astralcloud-billing
astralcloud-web
astralcloud-platform
```

This model may be revised where implementation reveals a clearly better boundary, but new repositories should not be created without a technical or organisational justification.
