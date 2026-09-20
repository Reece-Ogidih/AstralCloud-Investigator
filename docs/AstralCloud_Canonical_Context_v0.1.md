# AstralCloud — Canonical Company & Product Context

**Version:** v0.1  
**Date:** 20 September 2026  
**Project status:** Initial canonical specification

## 1. Purpose and status of this document

This document is the canonical reference for the fictional company AstralCloud and the production environment that will underpin the AstralCloud Investigator AI engineering project. It exists to prevent design drift as the project expands across multiple chats, repositories, generated artefacts, evaluation scenarios and implementation phases.

AstralCloud is intentionally designed as a realistic growth-stage software company rather than a toy demo. The environment should be large and internally coherent enough to support genuine investigation tasks involving source code, Git history, deployment records, infrastructure, metrics, logs, traces, tickets, internal documentation and engineer communications.

> **Status convention:** Facts marked LOCKED are treated as canonical unless deliberately revised. Facts marked PROVISIONAL are current design choices that may change as the architecture is elaborated. Any material change should increment the document version and be reflected in the project repository.

## 2. Executive summary

AstralCloud is a fictional multi-tenant B2B SaaS company that provides cloud observability and production-operations tooling to engineering organisations running cloud-native software. Customers connect applications, infrastructure and cloud environments to AstralCloud and use the platform to ingest, search, correlate and act on operational telemetry such as metrics, logs, traces and infrastructure events.

The company is modelled as a successful growth-stage organisation: large enough to have meaningful distributed-system complexity, multiple engineering teams, continuous deployment, on-call operations, legacy decisions and technical debt, while remaining small enough that the complete environment can be understood and simulated within a single engineering project.

The AstralCloud Investigator agent will not receive a simplified report of what happened. It will investigate the same classes of fragmented evidence that a production engineer would encounter. A hidden simulation layer will maintain canonical ground truth and generate the observable artefacts, allowing the system to be evaluated against known causes without exposing those causes to the agent.

## 3. Canonical company identity

| Field | Canonical value | Status |
| --- | --- | --- |
| Company name | AstralCloud | LOCKED |
| Legal-style name | AstralCloud Ltd | PROVISIONAL |
| Founded | 2021 | PROVISIONAL |
| Company type | B2B SaaS | LOCKED |
| Primary domain | Cloud observability and production operations | LOCKED |
| Stage | Growth-stage / approximately Series C scale | PROVISIONAL |
| Employees | Approximately 260 | PROVISIONAL |
| Engineering employees | Approximately 115 | PROVISIONAL |
| Customer organisations | Approximately 3,500 | PROVISIONAL |
| Primary infrastructure provider | AWS | PROVISIONAL |
| Primary runtime platform | Kubernetes on Amazon EKS | PROVISIONAL |

## 4. Product definition

AstralCloud gives engineering teams a unified platform for understanding the health, performance and behaviour of production systems. The product combines telemetry ingestion, storage, querying, visualisation, alerting and incident workflows.

### 4.1 Customer telemetry flow

Customers instrument applications and infrastructure using an AstralCloud collector and/or OpenTelemetry-compatible integrations. Telemetry is transmitted to AstralCloud ingestion endpoints, processed through distributed pipelines, stored in specialised backends and exposed through query APIs and the AstralCloud web application.

Conceptual flow: Customer applications / Kubernetes / databases / cloud services → Astral Collector or OpenTelemetry → AstralCloud ingestion edge → streaming and processing → specialised storage → query APIs → dashboards, alerts and incident workflows.

### 4.2 Core product areas

| Product area | Purpose |
| --- | --- |
| Metrics | Ingest, aggregate, query and visualise infrastructure, application and business metrics. |
| Logs | Ingest, index, retain and search structured and unstructured application/system logs. |
| APM / distributed tracing | Follow requests across service boundaries, analyse latency and errors, and expose service dependencies. |
| Infrastructure monitoring | Observe Kubernetes clusters, containers, nodes, cloud resources, databases and queues. |
| Alerting | Evaluate rules and service-level conditions, then notify engineers or trigger downstream workflows. |
| Incident management | Create, coordinate, record and resolve production incidents with timelines, responders and related evidence. |
| SLOs | Track service-level objectives, error budgets and reliability indicators. |
| Integrations | Connect external clouds, CI/CD systems, messaging systems, incident tooling and customer services. |
| Usage and billing | Meter customer consumption, enforce entitlements and support usage-based charging. |
| Organisation settings | Manage tenants, projects, environments, roles, permissions and configuration. |

## 5. Target customers and use cases

AstralCloud primarily serves software organisations operating production cloud infrastructure. The intended customer base includes SaaS companies, FinTech companies, e-commerce businesses, developer-tooling companies and digital marketplaces.

- SRE and platform teams monitor reliability, infrastructure health, capacity and incidents.
- Backend engineers investigate application errors, latency regressions, traces and deployment-related changes.
- Engineering managers monitor SLOs, incident frequency, operational trends and service health.
- Security and platform teams use infrastructure and audit telemetry for operational visibility.
- Finance and operations teams depend on accurate usage metering and subscription entitlement behaviour.

## 6. Business model

AstralCloud uses a usage-based SaaS model. Exact pricing is intentionally not yet defined, but billing is assumed to depend on combinations of monitored workloads, telemetry ingestion volume, retention, premium capabilities and user/organisation entitlements.

- Monitored hosts, containers or workloads.
- Log ingestion volume and retention period.
- Metric series or metric-event volume.
- Trace/span volume and retention.
- Number of seats or premium users.
- Advanced features such as longer retention, enterprise access controls or premium incident tooling.

This model is important technically because it requires reliable metering, aggregation, quota enforcement, subscription management, billing integration and entitlement checks. These systems create realistic failure modes that are distinct from the telemetry pipeline itself.

## 7. Scale assumptions

AstralCloud should feel like a real production system, but the project does not need to physically generate production-scale data. Company scale and simulation scale are deliberately separate concepts.

| Dimension | Canonical/target assumption | Materialisation strategy |
| --- | --- | --- |
| Customer organisations | ~3,500 | Represented canonically; only scenario-relevant tenants need full data. |
| Active users | ~90,000 | Represented statistically; a small subset receives explicit identities. |
| Monitored workloads | ~600,000 | Represented as environment scale rather than physically instantiated workloads. |
| Average telemetry throughput | ~1–2 million events/sec | Modelled through metadata and sampled/scaled scenario datasets. |
| Peak telemetry throughput | ~5 million events/sec | Used as an architectural constraint, not generated literally. |
| Log ingestion | ~5–10 TB/day | Scenario windows contain representative thousands/tens of thousands of events. |
| Metric series | Hundreds of millions active | Only scenario-relevant metric series are materialised. |
| Trace volume | Hundreds of millions of spans/day | Representative sampled traces are generated per scenario. |
| API traffic | Tens of millions of requests/day | Reflected in workload assumptions and sampled logs/traces. |

## 8. Multi-tenancy model

AstralCloud is a multi-tenant platform. Multi-tenancy is a foundational architectural choice because it creates realistic requirements around data isolation, permissions, noisy-neighbour effects, sharding, billing attribution and retention.

Most customer telemetry should be attributable through a common identity envelope containing at least organisation, environment and service dimensions. Additional project, region, workload or dataset identifiers may be introduced later.

| Field | Example |
| --- | --- |
| organisation_id | org_7hd82 |
| project_id | prj_checkout_prod |
| environment | production |
| service | checkout-api |
| region | eu-west-2 |

Multi-tenancy enables realistic incident classes such as incorrect tenant attribution, data isolation failures, quota-enforcement bugs, high-volume customer pressure, hot partitions, permission defects and usage-metering discrepancies.

## 9. High-level technical architecture

AstralCloud is primarily hosted on AWS and runs most application services on Kubernetes/EKS. The platform is expected to include an ingestion edge, streaming backbone, specialised processing services, multiple storage technologies, query APIs, customer-facing application services and internal platform services.

### 9.1 Core telemetry path

Customer system → collector / OpenTelemetry → ingestion edge → Kafka or equivalent streaming backbone → metrics/logs/traces processing → specialised storage → query APIs → AstralCloud web application → dashboards, alerts, SLOs and incidents.

### 9.2 Cross-cutting platform services

- Authentication and identity.
- Organisation and tenant management.
- Authorisation / RBAC.
- Usage metering and billing.
- Entitlements and quotas.
- Notifications and integrations.
- Configuration and feature flags.
- Deployment and runtime infrastructure.
- Internal observability and on-call tooling.

## 10. Technology assumptions

The exact service-by-service stack will be defined later. The following technologies form the provisional platform palette and should be used where they make architectural sense rather than forced into every component.

| Area | Provisional technology choices |
| --- | --- |
| Cloud | AWS |
| Containers / orchestration | Docker, Kubernetes, Amazon EKS |
| Infrastructure as code | Terraform |
| Primary service languages | Go, Python, Java/Kotlin, TypeScript |
| Streaming / event backbone | Kafka |
| Relational data | PostgreSQL |
| Caching / ephemeral state | Redis |
| Telemetry analytics storage | ClickHouse |
| Object storage | Amazon S3 |
| Telemetry standard | OpenTelemetry |
| Source control | GitHub |
| CI | GitHub Actions |
| Continuous delivery | Argo CD |
| Team communications | Slack-like internal workspace |
| Issue/project tracking | Jira-like system |
| Documentation | Confluence-like knowledge base / repository docs |
| On-call / paging | PagerDuty-like workflow |

## 11. Internal observability and dogfooding

AstralCloud dogfoods its own platform. Production AstralCloud services emit telemetry into an isolated internal AstralCloud organisation. Engineers use this internal tenant for service dashboards, log search, traces, SLOs, alerts and incident investigation.

This creates a valuable project property: the platform may sometimes be investigating failures in the same telemetry system it depends upon for visibility. Consequently, missing data must not automatically be interpreted as healthy behaviour. Scenarios may involve partial observability failure, delayed ingestion, dropped telemetry or misleading dashboards.

## 12. Engineering and operations model

AstralCloud is expected to operate as a modern cloud engineering organisation with continuous delivery, service ownership and on-call responsibility. Exact team topology and staffing will be defined in a later canonical file.

- Multiple specialised engineering teams own bounded service areas.
- Services are deployed continuously rather than through infrequent release trains.
- Production changes are traceable to source-control commits and deployment records.
- On-call rotations exist for operationally important domains.
- Incidents generate timelines, responder communications and follow-up work.
- Runbooks and architecture documentation exist but can become stale.
- Feature flags, configuration changes and infrastructure changes are first-class production changes.
- Not every failure is caused by the most recent application-code deployment.

## 13. The AstralCloud Investigator taskbed

The purpose of AstralCloud is not merely to provide a fictional backdrop. It is a controlled production-like taskbed in which an AI investigation system can be required to perform real evidence gathering and reasoning.

### 13.1 Hidden ground truth

A canonical world model and event history will exist outside the agent’s accessible environment. This hidden layer records what actually happened: service changes, deployments, infrastructure mutations, latent bugs, incident causes and causal relationships. It functions as both the source for synthetic artefact generation and the answer key for evaluation.

The agent must never be allowed to read the hidden world model or canonical event history during an investigation.

### 13.2 Observable evidence

The hidden history will generate realistic observable artefacts. The investigator should work from those artefacts directly rather than from prose summaries of their meaning.

| Evidence source | What the agent should actually inspect |
| --- | --- |
| Git repositories | Real source files, branches, commits, diffs, blame history, tags and configuration. |
| Code review / PR data | Change descriptions, review discussion, approvals and linked issues where materialised. |
| Deployments | Version changes, environment, timestamps, commit references, rollout state and rollback information. |
| Logs | Raw or structured log events queried by time, service, tenant, trace or message fields. |
| Metrics | Time-series values, alert thresholds and service/infrastructure measurements. |
| Traces | Spans, parent/child relationships, timing, errors and cross-service request paths. |
| Internal communications | Actual channel messages and threads, including speculation, corrections and incomplete information. |
| Tickets | Descriptions, acceptance criteria, comments, status and links to changes/incidents. |
| Documentation | Runbooks, architecture notes, operational procedures and potentially stale information. |
| Infrastructure/configuration | Terraform, Kubernetes manifests, Helm values, environment configuration and runtime metadata. |
| Incidents | Declared incidents, responder timelines, severity changes, notes and post-incident records. |

### 13.3 Investigator behaviour

The target agent is an investigator, not a synthesiser. It should form hypotheses, choose evidence sources, inspect low-level artefacts, update or reject hypotheses, correlate events and produce a conclusion supported by exact evidence.

- Search and inspect source code rather than only reading commit descriptions.
- Compare commits and configurations across versions.
- Correlate deployments with metric, log and trace changes.
- Distinguish engineer speculation from verified evidence.
- Detect stale or contradictory documentation.
- Traverse service dependencies when symptoms originate downstream.
- Consider infrastructure, configuration, data and external dependencies as well as application code.
- Recognise when evidence is insufficient and avoid fabricated certainty.
- Cite the concrete artefacts that support a conclusion.

### 13.4 Intentionally misleading evidence

Some scenarios should contain plausible but misleading signals. For example, engineers may initially blame Redis because its latency rose at the same time as an incident, while the true cause is a Kubernetes CPU limit changed in a Helm configuration several deployments earlier. This is deliberate: an investigator should test hypotheses rather than repeat the loudest available theory.

## 14. Simulation realism without unmanageable data volume

Production realism does not require physically storing every event implied by AstralCloud’s scale. The simulation should materialise bounded, internally consistent slices around scenarios while maintaining background noise and unrelated changes so the correct evidence is not trivially isolated.

A typical investigation window might contain tens of thousands of log records, minute-level metric series, thousands of sampled traces, multiple deployments, dozens of commits, several tickets and a meaningful amount of engineer discussion. These figures can be tuned as the tooling matures.

The crucial requirement is causal and temporal consistency: observable artefacts generated from the same hidden event must agree on identities, timestamps, versions and dependencies unless inconsistency itself is an intentional part of the scenario.

## 15. Canonical generation philosophy

AstralCloud data should be generated from a single canonical history rather than maintained as independent fictional datasets. A hidden event can produce multiple externally observable consequences: a Git commit, a deployment, log patterns, metric shifts, trace changes, Slack discussion, a ticket and eventually an incident record. This approach keeps artefacts synchronised and makes scenario generation reproducible.

Generated source repositories should become real Git repositories. Where practical, the agent should interact with actual files and Git history rather than JSON abstractions that merely describe code changes.

## 16. Planned engineering organisation

The detailed team topology is deliberately deferred until after this company-level context is accepted. The current high-level expectation is approximately nine major engineering domains, subject to refinement:

- Ingestion.
- Telemetry Platform.
- Query & Storage.
- APM.
- Infrastructure Monitoring.
- Identity & Access.
- Billing.
- Web Platform.
- Developer Platform / SRE.

Non-engineering functions such as Product, Design, Sales, Customer Success, Security, Finance and People exist canonically but only need deep simulation when they are relevant to a scenario.

## 17. Product surface

The customer-facing application is expected to expose the following major navigation areas. Exact naming and UI details remain provisional.

- Overview.
- Infrastructure: Kubernetes, hosts and cloud resources.
- APM: services, traces and dependency views.
- Metrics: explorer and dashboards.
- Logs: search and analytics.
- Alerts.
- Incidents.
- SLOs.
- Integrations.
- Usage & Billing.
- Organisation Settings.

## 18. Design principles that are now locked

| Principle | Meaning |
| --- | --- |
| Production-scale semantics | AstralCloud should behave like a real distributed SaaS platform, not a tutorial application. |
| Bounded materialisation | We simulate only the data needed to make scenarios realistic; company-scale numbers are architectural context. |
| Hidden truth vs visible evidence | Ground truth is inaccessible to the agent and used only for generation/evaluation. |
| Real artefacts where useful | Source code, Git history and configuration should be materialised as actual files/repositories where practical. |
| Investigation over summarisation | The agent must gather and test evidence rather than read pre-solved reports. |
| Cross-source consistency | Generated artefacts share canonical IDs, timestamps, versions and causal relationships. |
| Messy reality | Humans can speculate incorrectly, docs can be stale, alerts can be noisy and telemetry can be incomplete. |
| Evaluability | Scenarios must have known ground truth so agent performance can be measured objectively. |

## 19. Open design decisions

The following decisions are intentionally not yet fixed and should be resolved in later design phases rather than invented implicitly during implementation.

- Exact engineering team structure, team sizes, leads and individual staff identities.
- Exact service catalogue and ownership boundaries.
- Regional deployment topology and disaster-recovery model.
- Detailed data-retention tiers and customer pricing.
- Exact storage architecture for each telemetry type.
- Whether some subsystems are monoliths, modular services or independently deployed microservices.
- Exact internal developer platform and CI/CD workflow.
- Precise source-control repository boundaries.
- Detailed observability query APIs exposed to the investigation agent.
- Scenario taxonomy and evaluation metrics.
- Synthetic-history generation implementation and source-code generation strategy.

## 20. Repository role of this document

This document should live in the project repository as the top-level design reference, ideally alongside a machine-friendly Markdown version. A suggested location is docs/astralcloud_context.md. The Markdown copy should be the easiest form to version-control, diff and provide to future AI sessions, while the formatted document can be used as a human-readable reference.

As more formal canonical files are added—such as company.yaml, teams.yaml, services.yaml, infrastructure.yaml and events schemas—those files should not silently contradict this document. If a design change is accepted, update the canonical context and the affected structured files in the same change.

## 21. Change log

| Version | Date | Summary |
| --- | --- | --- |
| v0.1 | 20 September 2026 | Initial canonical company and product definition. Establishes AstralCloud as a multi-tenant B2B cloud observability and production-operations platform and defines the investigation taskbed philosophy. |
