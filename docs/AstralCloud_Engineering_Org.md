# AstralCloud Engineering Organisation

**Version:** 0.1
**Status:** Working canonical organisation design
**Last updated:** September 2026

---

# 1. Engineering organisation philosophy

AstralCloud has approximately **115 people in Engineering**, including engineering managers and senior engineering leadership.

The organisation is divided into small, stable teams with clear ownership boundaries.

The operating principle is broadly:

> **You build it, you run it.**

Teams that own production software are responsible not only for feature development, but also for:

* production reliability;
* deployment;
* monitoring;
* operational documentation;
* incident response;
* technical debt;
* service-level objectives.

AstralCloud deliberately avoids a model where a central operations team owns all production problems.

SRE and platform teams provide shared infrastructure, reliability expertise and escalation support, but application teams retain operational responsibility for the systems they own.

---

# 2. High-level organisation

Engineering is organised into four major groups:

```text
VP Engineering
│
├── Telemetry Platform
│
├── Observability Products
│
├── Core Product
│
└── Platform & Reliability
```

Approximate engineering headcount:

```text
Telemetry Platform          37
Observability Products      35
Core Product                20
Platform & Reliability      19
Engineering Leadership       4
                            ───
Total                      115
```

The four central engineering leadership positions are provisionally:

```text
VP Engineering

Director of Telemetry Platform
Director of Product Engineering
Director of Platform & Reliability
```

Exact titles and individuals remain undecided.

---

# 3. Telemetry Platform

The Telemetry Platform group owns the path from customer telemetry entering AstralCloud through to the systems used to retrieve that telemetry.

It contains five teams.

## 3.1 Collectors & Edge

**Approximate headcount:** 7

Owns the boundary between customer environments and AstralCloud.

Primary responsibilities:

* Astral Collector;
* OpenTelemetry compatibility;
* OTLP/gRPC and OTLP/HTTP ingestion;
* edge endpoints;
* telemetry batching;
* retries;
* compression;
* collector configuration;
* customer-side metadata enrichment;
* edge reliability.

This team owns software that executes both within customer environments and at AstralCloud's public telemetry edge.

Typical incident classes include:

```text
Collector memory leak
OTLP compatibility regression
TLS failure
Payload-size regression
Bad retry behaviour
Regional edge outage
Collector upgrade incompatibility
```

---

## 3.2 Ingestion & Routing

**Approximate headcount:** 8

Owns the first internal stage of the telemetry data plane.

Primary responsibilities:

* request validation;
* API-key validation at ingestion;
* tenant attribution;
* telemetry routing;
* schema validation;
* quotas;
* rate limiting;
* ingestion acknowledgements;
* publishing telemetry into the streaming backbone.

This team is responsible for ensuring accepted customer telemetry enters AstralCloud correctly.

---

## 3.3 Streaming Platform

**Approximate headcount:** 7

Provides AstralCloud's Kafka-based event platform.

This is primarily an internal platform team rather than a customer-facing product team.

Responsibilities include:

* Kafka clusters;
* topic lifecycle;
* partition strategy;
* producer/consumer standards;
* schema management;
* capacity planning;
* replication;
* consumer-lag tooling;
* streaming reliability.

Other engineering teams consume the streaming platform as an internal service.

The team does **not** own every Kafka consumer merely because that consumer uses Kafka.

For example:

```text
Streaming Platform owns:

Kafka itself
topic configuration
cluster capacity
shared streaming libraries

Metrics team owns:

its metrics processor
its consumer behaviour
its consumer offsets
its processing logic
```

This distinction will matter substantially during investigations.

---

## 3.4 Storage Platform

**Approximate headcount:** 8

Owns AstralCloud's high-volume telemetry storage platform.

Primary responsibilities:

* ClickHouse;
* telemetry storage architecture;
* replication;
* partitioning;
* retention infrastructure;
* storage performance;
* capacity management;
* S3 archival pipelines;
* backup and recovery.

This is a highly specialised platform team.

Product teams should not need to understand the low-level operational details of ClickHouse to build features.

They consume storage capabilities through defined interfaces and libraries.

---

## 3.5 Query Platform

**Approximate headcount:** 7

Owns the shared infrastructure used to query AstralCloud telemetry.

Responsibilities include:

* query gateway;
* query validation;
* query planning;
* common query language;
* query execution;
* query caching;
* performance controls;
* query limits;
* aggregation infrastructure.

Product teams such as Metrics, Logs and APM build experiences on top of this platform.

This creates an important ownership distinction:

```text
Metrics Team
    ↓
uses
    ↓
Query Platform
    ↓
uses
    ↓
Storage Platform
```

A broken metrics dashboard may therefore originate in any of these three areas.

---

# 4. Observability Products

This group owns the principal customer-facing observability capabilities.

It consists of five teams.

---

## 4.1 Metrics

**Approximate headcount:** 7

Owns the metrics product.

Responsibilities include:

* metrics processing;
* metric metadata;
* aggregation;
* rollups;
* metric exploration;
* metric visualisation APIs;
* cardinality behaviour;
* customer-facing metrics functionality.

The team owns both relevant backend processing and the product behaviour of metrics.

---

## 4.2 Logs

**Approximate headcount:** 7

Owns the logging product.

Responsibilities include:

* log processing;
* parsing;
* enrichment;
* log metadata;
* log search behaviour;
* indexes;
* customer-facing log exploration;
* trace/log correlation.

---

## 4.3 APM & Tracing

**Approximate headcount:** 8

Owns distributed tracing and application-performance monitoring.

Responsibilities include:

* trace processing;
* span processing;
* sampling;
* service discovery;
* service maps;
* trace search;
* latency analysis;
* APM service views;
* trace correlation.

This team contains some of AstralCloud's strongest distributed-systems expertise.

---

## 4.4 Detection & Alerting

**Approximate headcount:** 7

Owns monitor evaluation and alert generation.

Responsibilities include:

* monitor configuration semantics;
* streaming evaluation;
* scheduled/query evaluation;
* monitor state;
* alert state transitions;
* deduplication;
* silence/muting behaviour;
* anomaly-detection infrastructure where applicable.

This team determines **whether an alert should fire**.

It does not necessarily own how that alert is communicated externally.

---

## 4.5 Incidents & Notifications

**Approximate headcount:** 6

Owns AstralCloud's incident-management functionality and outbound notification platform.

Responsibilities include:

* incident creation;
* incident lifecycle;
* responder assignment;
* incident timelines;
* Slack integration;
* PagerDuty integration;
* email notifications;
* webhooks;
* notification retries;
* delivery status.

This separation means:

```text
Monitor fires correctly
        ↓
alert event exists
        ↓
notification fails
```

is a legitimate failure mode.

---

# 5. Core Product

Core Product owns the SaaS functionality surrounding the observability platform itself.

It contains three teams.

---

## 5.1 Identity & Organisations

**Approximate headcount:** 7

Owns:

* authentication;
* organisations;
* projects;
* users;
* teams;
* RBAC;
* SSO;
* API keys;
* sessions;
* organisation configuration.

This team owns a substantial portion of the AstralCloud control plane.

Because tenant identity flows throughout the platform, failures here can have wide effects.

---

## 5.2 Billing & Metering

**Approximate headcount:** 7

Owns:

* usage events;
* usage aggregation;
* usage ledger;
* pricing rules;
* subscriptions;
* quotas;
* entitlements;
* invoices;
* payment-provider integration.

This team interacts heavily with both the data plane and control plane.

For example:

```text
Ingestion
    ↓
usage event
    ↓
Kafka
    ↓
Metering
    ↓
Usage Ledger
    ↓
Billing
```

Errors here may affect customer charges without affecting telemetry availability.

---

## 5.3 Web Experience

**Approximate headcount:** 6

Owns the shared AstralCloud web application and customer-facing frontend infrastructure.

Primary technologies:

* React;
* TypeScript.

Responsibilities include:

* application shell;
* navigation;
* shared design system;
* frontend authentication integration;
* shared state;
* frontend performance;
* API integration;
* browser observability.

Domain-specific frontend functionality may be implemented collaboratively with corresponding product teams.

For example, the Metrics team may own metric-specific functionality while Web Experience owns the shared frontend foundation.

---

# 6. Platform & Reliability

This group enables the rest of Engineering to build, deploy and operate software safely.

It contains three teams.

---

## 6.1 Developer Platform

**Approximate headcount:** 7

Developer Platform treats AstralCloud engineers as its customers.

Responsibilities include:

* Docker build conventions;
* base container images;
* GitHub Actions;
* CI templates;
* deployment tooling;
* Argo CD;
* Helm standards;
* developer environments;
* secrets integration;
* service templates;
* internal developer tooling.

The intended developer experience is approximately:

```text
Engineer
   ↓
GitHub
   ↓
CI
   ↓
Docker Image
   ↓
ECR
   ↓
Argo CD
   ↓
EKS
```

Application teams own their application deployments.

Developer Platform owns the machinery that makes those deployments possible.

---

## 6.2 Site Reliability Engineering

**Approximate headcount:** 7

SRE provides reliability engineering across AstralCloud.

Responsibilities include:

* SLO framework;
* reliability standards;
* incident-management practices;
* capacity planning;
* reliability reviews;
* production readiness;
* major incident coordination;
* resilience testing;
* disaster recovery;
* critical operational tooling;
* out-of-band observability.

SRE does **not** become the default owner for application failures.

Instead:

```text
Service problem
      ↓
Service-owning team's on-call responds
      ↓
SRE assists/escalates when appropriate
```

For major cross-platform incidents, SRE may provide the incident commander.

---

## 6.3 Security Engineering

**Approximate headcount:** 5

Owns shared security capabilities and provides security expertise across engineering.

Responsibilities include:

* cloud security;
* IAM standards;
* secrets-management standards;
* vulnerability management;
* container security;
* dependency scanning;
* security monitoring;
* threat detection;
* security incident response.

Security also works closely with Identity & Organisations on customer-facing security functionality.

This team is comparatively small because security responsibility remains distributed throughout Engineering rather than being delegated completely to one team.

---

# 7. Engineering ownership model

Every production component must have a clearly identified owning team.

Ownership applies to:

```text
Source repositories
Services
Kafka consumers
Infrastructure
Databases
Runbooks
Dashboards
Alerts
SLOs
Deployments
Operational documentation
```

There should generally be one **primary owning team**, although dependencies may involve several supporting teams.

For example:

```text
Customer reports missing traces
              |
              v
       APM & Tracing
        owns symptom
              |
              v
        Investigation
          /       \
         /         \
 Ingestion         Query Platform
     |                  |
     v                  v
Streaming           Storage Platform
 Platform
```

No single team automatically owns the entire investigation simply because the incident begins in its product area.

---

# 8. On-call model

Most production-owning teams operate an on-call rotation.

Examples include:

```text
collector-edge-oncall
ingestion-oncall
streaming-oncall
storage-oncall
query-oncall
metrics-oncall
logs-oncall
apm-oncall
alerting-oncall
incident-platform-oncall
identity-oncall
billing-oncall
web-oncall
developer-platform-oncall
sre-oncall
security-oncall
```

Not every alert pages every team.

Routing is determined by system ownership and alert configuration.

For severe incidents, multiple rotations may become involved.

Example:

```text
14:04   trace ingestion SLO breaches

14:06   apm-oncall paged

14:09   APM determines incoming trace volume has fallen

14:12   ingestion-oncall engaged

14:16   ingestion identifies Kafka publish latency

14:19   streaming-oncall engaged

14:27   issue traced to Kafka broker capacity after infrastructure change

14:29   sre-oncall joins as incident commander
```

This organisational history will later be reflected in Slack, PagerDuty-style alerts and incident records.

---

# 9. Cross-team interaction

Teams communicate through several mechanisms.

Routine engineering work uses:

```text
GitHub
Jira
Slack
documentation
architecture proposals
code review
```

Operational work additionally uses:

```text
on-call systems
incident channels
alerts
dashboards
runbooks
postmortems
```

AstralCloud deliberately has imperfect organisational knowledge.

Therefore it is valid for:

* engineers to misunderstand another team's system;
* ownership documents to become stale;
* Slack speculation to be incorrect;
* old incidents to bias current investigations;
* tickets to omit important context;
* documentation to lag behind implementation.

This is intentional because the investigation agent must reason about evidence quality rather than assuming every internal statement is authoritative.

---

# 10. Relationship between teams and repositories

Teams do not necessarily have exactly one repository.

Repository boundaries will be defined later.

Valid arrangements include:

```text
One team → several repositories

Several teams → one large repository

Platform team → infrastructure repositories

Product team → application + worker repositories
```

Repository structure should emerge from technical boundaries rather than being forced to mirror the organisational chart.

Every repository will nevertheless have identifiable ownership.

This may eventually be expressed through mechanisms analogous to:

```text
CODEOWNERS
```

and repository metadata.

---

# 11. Relationship between teams and services

Likewise, teams do not map one-to-one to services.

For example:

```text
APM & Tracing

could own:

trace ingestion worker
trace processor
sampling service
service-map processor
APM API
```

while:

```text
Streaming Platform

could own:

Kafka infrastructure
schema registry
shared producer libraries
consumer-lag tooling
```

The exact service decomposition will be established later.

---

# 12. Team-size summary

| Group                  | Team                         | Approx. headcount |
| ---------------------- | ---------------------------- | ----------------: |
| Telemetry Platform     | Collectors & Edge            |                 7 |
| Telemetry Platform     | Ingestion & Routing          |                 8 |
| Telemetry Platform     | Streaming Platform           |                 7 |
| Telemetry Platform     | Storage Platform             |                 8 |
| Telemetry Platform     | Query Platform               |                 7 |
| Observability Products | Metrics                      |                 7 |
| Observability Products | Logs                         |                 7 |
| Observability Products | APM & Tracing                |                 8 |
| Observability Products | Detection & Alerting         |                 7 |
| Observability Products | Incidents & Notifications    |                 6 |
| Core Product           | Identity & Organisations     |                 7 |
| Core Product           | Billing & Metering           |                 7 |
| Core Product           | Web Experience               |                 6 |
| Platform & Reliability | Developer Platform           |                 7 |
| Platform & Reliability | Site Reliability Engineering |                 7 |
| Platform & Reliability | Security Engineering         |                 5 |
|                        | **Team total**               |           **111** |
|                        | Engineering leadership       |             **4** |
|                        | **Engineering total**        |           **115** |

Headcounts represent approximate organisational scale rather than immutable staffing numbers.

---

# 13. Current canonical decisions

The following are now treated as established assumptions:

* approximately 115 engineering staff;
* four major engineering groups;
* sixteen engineering teams;
* team sizes generally between five and eight engineers/managers;
* clear service ownership;
* application teams operate what they build;
* SRE supports rather than replacing service ownership;
* most production-owning teams maintain on-call rotations;
* Developer Platform provides shared delivery infrastructure;
* Streaming and Storage operate as internal platform capabilities;
* engineering communication and documentation are imperfect;
* ownership applies across code, infrastructure and operations.

---

# 14. Deliberately undecided

The following remain to be designed:

* individual engineers;
* team leads and engineering managers;
* exact reporting structure;
* exact repositories;
* exact services;
* CODEOWNERS configuration;
* Slack channel names;
* Jira project structure;
* exact on-call membership;
* exact rotations;
* individual employment history;
* team reorganisations;
* historical ownership changes.

These will be derived from the technical and historical requirements of AstralCloud rather than invented independently.
