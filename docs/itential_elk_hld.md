# Itential + Elastic Stack Integration
## High Level Design

---

## 1. Executive Summary

As network automation becomes central to enterprise operations, the ability to
observe, audit, and troubleshoot automation workflows in real time becomes equally
critical. Itential's automation platform — comprising the Itential Automation
Platform (IAP) and Automation Gateway 5 (IAG5) — generates rich operational data
across every workflow execution, device interaction, and system event. Without a
centralized observability strategy, this data is fragmented across individual log
files on multiple servers, making it difficult to diagnose failures, demonstrate
compliance, or measure automation effectiveness.

This document proposes integrating Itential's platform with the Elastic Stack
(Elasticsearch, Logstash, Kibana, and Filebeat) to provide a unified observability
plane for network automation operations. The integration surfaces workflow health,
gateway activity, southbound device interactions, and platform-level events in a
single, searchable, and alertable interface.

---

## 2. Business Drivers

### 2.1 Operational Visibility

In a distributed automation environment, a single workflow execution touches
multiple components — IAP orchestrates the job, GatewayManager dispatches it to
IAG5, and IAG5 executes the automation against one or more network devices.
Without centralized log aggregation, diagnosing a failure requires logging into
multiple servers, correlating timestamps manually, and piecing together a sequence
of events spread across different log files. This is slow, error-prone, and
unscalable as automation footprint grows.

Centralizing logs in Elasticsearch allows operations teams to reconstruct the
complete execution trail of any workflow — from job submission through gateway
dispatch to device-level execution — in a single query, reducing mean time to
resolution (MTTR) dramatically.

### 2.2 Proactive Alerting

Reactive troubleshooting is insufficient for production network automation
environments. A failed automation that goes unnoticed for hours can leave network
devices in a partially configured state, creating outage risk. Kibana's alerting
engine enables proactive notification — teams can be alerted via PagerDuty, Slack,
or email the moment automation failure rates spike, gateway connectivity is lost,
or platform authentication anomalies are detected.

### 2.3 Audit & Compliance

Enterprise customers operating under regulatory frameworks (SOX, PCI-DSS, HIPAA,
or internal change management policies) require a complete, tamper-evident audit
trail of who initiated which automation, when, against which devices, and what the
outcome was. The Elastic Stack provides centralized, indexed, and queryable audit
data drawn from IAP's webserver access logs and sub-process event logs, supporting
both internal audit requirements and external compliance reporting.

### 2.4 Operational Metrics & Continuous Improvement

Beyond troubleshooting, centralized observability enables automation teams to
measure and improve their operations over time. Kibana dashboards can surface
workflow success rates, average job execution times, most frequently failing
automations, and gateway utilization trends — data that is otherwise invisible
when logs remain siloed on individual servers. This supports data-driven
prioritization of automation improvements and capacity planning.

### 2.5 Alignment with Existing Enterprise Tooling

The Elastic Stack is widely deployed in enterprise environments, often already
present as part of security (SIEM), application performance monitoring (APM), or
IT operations (ITOM) initiatives. Integrating Itential into an existing Elastic
deployment adds value to both investments — extending Elastic's observability
coverage to network automation, and giving Itential customers a path to correlate
automation activity with broader infrastructure events without adopting a net-new
tooling stack.

For customers already running Grafana for infrastructure visualization, the Elastic
Stack serves as a shared data backend — Itential automation data becomes available
alongside network device metrics in existing Grafana dashboards.

---

## 3. Itential Platform Overview

The integration covers the following Itential components:

**IAP (Itential Automation Platform)** is the workflow orchestration engine. It is
responsible for defining, scheduling, and executing automation workflows. IAP is
composed of a main platform process and multiple sub-processes — including the
WorkFlow Engine, Operations Manager, and Gateway Manager — each of which generates
structured log data relevant to operational monitoring. IAP exposes a REST API,
and all API activity is captured in a structured webserver access log.

**IAG5 (Automation Gateway 5)** is the southbound automation gateway. It receives
dispatch requests from IAP's Gateway Manager and executes automation tasks against
network devices using protocols such as gRPC, NETCONF, RESTCONF, and CLI. IAG5
emits structured JSON logs capturing every service invocation, command execution,
and device interaction.

Supporting infrastructure components — MongoDB (IAP's primary datastore), Redis
(session and queue management), and BullMQ (job queue, built on Redis) — also
contribute operational data to the observability picture.

---

## 4. Integration Overview

The integration follows a standard log aggregation and observability pattern:

**Collection** — Lightweight Filebeat agents run alongside Itential components on
each server, continuously monitoring log files and forwarding new entries to the
central processing pipeline. Filebeat is designed to be minimally invasive — it
adds no measurable overhead to the Itential processes it monitors.

**Processing** — A Logstash pipeline receives log events from Filebeat, enriches
them with structured fields (component identity, severity tagging, correlation
IDs), and routes them to the appropriate Elasticsearch index. This processing layer
normalizes the different log formats produced by IAP and IAG5 into a consistent
schema.

**Storage & Search** — Elasticsearch stores all log events in time-based indices,
providing fast full-text search and aggregation across the entire log corpus.
Index lifecycle management (ILM) policies automatically age and expire old data,
keeping storage costs predictable.

**Visualization & Alerting** — Kibana provides the operational interface:
pre-built dashboards for workflow health, gateway operations, infrastructure
status, and security auditing; saved searches for common troubleshooting queries;
and alerting rules that notify the right teams when defined thresholds are breached.

### 4.1 Cross-Component Correlation

A key capability enabled by this integration is end-to-end tracing of an
automation execution across IAP and IAG5. Two correlation identifiers link the
components together:

The **IAP Job ID** (a MongoDB ObjectID) is assigned when a workflow is submitted
and appears throughout IAP's webserver logs in all subsequent job status queries.
It represents the workflow-level identity of an automation execution.

The **Gateway Dispatch UUID** is assigned by IAP's Gateway Manager when a job is
dispatched to IAG5 via the BullMQ queue. This UUID is passed to IAG5 as the
JSON-RPC call identifier and appears in both the IAP GatewayManager log and IAG5's
gateway log — making it the true cross-boundary correlation key. A single search
on this UUID in Kibana surfaces the complete gateway-level execution trail across
both components.

Together, these two identifiers allow operators to trace any automation execution
from the initial API call in IAP's webserver log, through the gateway dispatch in
GatewayManager, to the device-level execution result in IAG5 — without leaving
the Kibana interface.

### 4.2 Architecture Diagram

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│       Itential Platform     │         │        Elastic Stack        │
│                             │         │                             │
│  IAP ──► Filebeat ──────────┼────────►│ Logstash ──► Elasticsearch  │
│  IAG5 ──► Filebeat ─────────┼────────►│                    │        │
│  MongoDB ──► Filebeat ──────┼────────►│             Kibana ◄────────┤
│  Redis ──► Filebeat ────────┼────────►│             (Dashboards,    │
│                             │         │              Alerting)      │
│  Redis ──► redis_exporter ──┼────────►│                             │
│           (BullMQ metrics)  │         │  [Grafana - optional]       │
└─────────────────────────────┘         └─────────────────────────────┘
```

---

## 5. Integration Patterns

Three integration patterns are available depending on customer environment maturity
and requirements:

**Pattern 1 — Direct (Filebeat → Elasticsearch)** is the simplest deployment,
suitable for customers new to the Elastic Stack who want quick time-to-value.
Filebeat ships logs directly to Elasticsearch with no intermediate processing.

**Pattern 2 — Enriched Pipeline (Filebeat → Logstash → Elasticsearch)** is the
recommended pattern for production deployments. Logstash provides field
normalization, failure tagging, and index routing, producing a richer and more
consistent dataset for Kibana dashboards and alerting.

**Pattern 3 — Metrics Integration (Prometheus → Elasticsearch)** complements log
shipping with structured time-series metrics for Redis and BullMQ queue depth,
enabling unified dashboards that combine logs and metrics in a single view.

---

## 6. Key Capabilities Delivered

| Capability | Description |
|---|---|
| Centralized log search | Full-text search across all IAP and IAG5 log data from a single Kibana interface |
| Workflow health dashboards | Real-time and historical view of job execution rates, success/failure ratios, and duration trends |
| Gateway operations visibility | IAG5 service invocation tracking, device interaction logs, and connectivity status |
| End-to-end job tracing | Correlation of a single automation execution across IAP and IAG5 using shared identifiers |
| Proactive alerting | Notification via PagerDuty, Slack, or email on automation failures, gateway connectivity loss, and auth anomalies |
| Audit trail | Complete record of who ran what automation, when, and with what outcome |
| Infrastructure health | MongoDB, Redis, and BullMQ queue depth monitoring alongside application logs |
| Air-gapped deployment | Full support for offline enterprise environments via Nexus package mirroring |
| Ansible-automated deployment | All components deployable via the `itential.monitoring` Ansible collection |

---

## 7. Deployment Considerations

### 7.1 Air-Gapped Environments

Many Itential customers operate in network-isolated environments. The integration
is fully supported in air-gapped deployments — all Elastic Stack packages are
distributed via the customer's existing Nexus repository infrastructure, consistent
with how Itential's own packages are managed. No internet connectivity is required
at runtime.

### 7.2 Security

All communication between Itential components, Filebeat, Logstash, and
Elasticsearch is encrypted using TLS, with certificates issued from the customer's
internal certificate authority — the same CA used for Itential's own TLS
configuration. Role-based access control (RBAC) in Elasticsearch ensures that each
component has only the permissions it requires.

### 7.3 Deployment Automation

All components of this integration — Elasticsearch, Kibana, Logstash, Filebeat,
and the redis_exporter — are deployable via Ansible roles included in the
`itential.monitoring` collection. Customers already using Ansible for Itential
platform deployment can extend their existing playbooks to include the full
observability stack.

### 7.4 Alternative Visualization

Customers with existing Grafana deployments can connect Grafana directly to
Elasticsearch as an alternative or complementary visualization layer. This enables
Itential operational data to appear alongside network device metrics in existing
Grafana dashboards. See the accompanying Low Level Design document for
configuration details.

---

## 8. Licensing & Cost Considerations

### Open Source Status

The Elastic Stack has a notable licensing history that customers may be aware of.
Elasticsearch and Kibana were originally licensed under Apache 2.0. In 2021, Elastic
moved to a dual license under the Server Side Public License (SSPL) and Elastic
License v2 (ELv2) — neither of which are OSI-approved open source licenses — in
response to cloud providers offering Elasticsearch as a managed service without
contributing back to the project. In August 2024, Elastic reversed course and added
the AGPLv3 license as an option alongside SSPL and ELv2, making Elasticsearch and
Kibana officially open source again under an OSI-approved license. Logstash, Filebeat,
and Elastic's client libraries remain licensed under Apache 2.0 and were unaffected
by these changes.

For practical purposes, customers deploying the Elastic Stack self-managed in an
enterprise environment — as described in this guide — are not offering it as a
commercial service and are unaffected by the licensing history. The stack is free
to install and use.

### Free vs. Paid Tiers

Elastic offers a tiered subscription model. The core stack is available under a
free Basic tier, with paid tiers (Standard, Gold, Platinum, Enterprise) unlocking
advanced machine learning, enhanced security features, and support SLAs.

**All features described in this integration fall within the free Basic tier,**
including Elasticsearch storage and search, Kibana dashboards, index lifecycle
management, Logstash pipelines, Filebeat agents, and standard alerting rules.

The one area to verify before deployment is third-party alerting connectors.
Email and webhook connectors are available in the free tier. Connectors for
PagerDuty and Slack — referenced in the alerting section of the companion Low
Level Design — may require a paid Standard subscription for self-managed
deployments. Customers should confirm connector availability against their
intended subscription tier before relying on those integrations for production
alerting.

Paid subscriptions become relevant when customers require:

- Formal support SLAs from Elastic (24/7 coverage, guaranteed response times)
- Advanced machine learning features (anomaly detection, forecasting)
- Cross-cluster search and replication at scale
- Elastic's SIEM and threat detection capabilities

For the observability use case described in this guide, the free Basic tier is
the recommended starting point. Customers can upgrade to a paid tier later if
support or advanced feature requirements emerge.

### OpenSearch — An Alternative to Consider

Elastic's 2021 license change prompted AWS to fork Elasticsearch into
**OpenSearch**, which is fully licensed under Apache 2.0 and maintained as a
community project. OpenSearch is API-compatible with Elasticsearch, meaning the
Logstash pipelines, Filebeat configurations, and index management described in
the companion Low Level Design document are directly applicable with minimal
changes.

OpenSearch is worth considering for customers who:

- Operate primarily in AWS environments and prefer AWS-managed tooling
- Have strict internal policies requiring Apache 2.0 or OSI-approved licensing
  for all infrastructure components
- Are already running OpenSearch as part of an existing AWS observability stack

For customers with no existing preference or AWS dependency, Elasticsearch
remains the more mature option with a wider ecosystem of integrations,
documentation, and community support.

---

## 9. Relationship to Prometheus & Grafana

Many Itential customers already operate a Prometheus and Grafana stack for
infrastructure monitoring. This section clarifies how the Elastic Stack relates
to that investment and whether both are warranted.

### Different Tools, Different Problems

Prometheus and Grafana are purpose-built for **metrics** — numeric, time-series
data sampled at regular intervals. They excel at answering questions like "is Redis
memory trending toward capacity?", "how many BullMQ jobs are queued right now?",
or "is CPU elevated on my IAP node?" Grafana visualizes this data as graphs,
gauges, and trend panels. This is a fundamentally different data type from logs.

The Elastic Stack is purpose-built for **logs and events** — structured or
semi-structured text records generated when something happens. It excels at
answering questions like "what exactly happened during this failed workflow?",
"which user submitted the job that caused the error?", or "show me every
GatewayManager connectivity failure in the last 24 hours." Kibana provides
full-text search, event exploration, and dashboards built on discrete log events
rather than sampled numeric values.

While both stacks have some overlap — Elasticsearch can store metrics, Prometheus
can derive counts from logs — using either tool for the other's primary use case
means working against the grain of the technology. Prometheus is not a log search
engine. Elasticsearch is not an efficient high-frequency metrics store.

### Complementary, Not Competing

For Itential deployments, the two stacks address different observability needs and
serve different operational workflows:

| Observability Need | Best Tool |
|---|---|
| IAP / IAG5 workflow execution logs | Elastic Stack |
| End-to-end job tracing (IAP → IAG5) | Elastic Stack |
| Audit trail — who ran what, when | Elastic Stack |
| Full-text search across log events | Elastic Stack |
| Log-based alerting (job failure spike) | Elastic Stack |
| Redis / BullMQ queue depth over time | Prometheus + Grafana |
| IAP node CPU, memory, disk trends | Prometheus + Grafana |
| MongoDB performance metrics | Prometheus + Grafana |
| Infrastructure health dashboards | Prometheus + Grafana |
| Numeric threshold alerting | Prometheus + Grafana |
| Correlating automation with device metrics | Grafana (querying both) |

The most compelling case for running both stacks is the last row. Grafana supports
multiple datasources simultaneously, meaning a single Grafana dashboard can display
a BullMQ queue depth spike from Prometheus alongside the corresponding job failure
log events from Elasticsearch — in the same view, at the same time. This
cross-datasource correlation is difficult to achieve with either stack alone and
represents a meaningful operational advantage for teams troubleshooting automation
issues in production.

### Recommendation

Customers already running Prometheus and Grafana should not view the Elastic Stack
as a replacement. They are being asked to add a log observability and audit layer
that fills a gap their existing stack cannot address. The `itential.monitoring`
Ansible collection manages the `redis_exporter` role that feeds Redis and BullMQ
metrics into Prometheus, ensuring both stacks are deployed and maintained
consistently. Grafana remains relevant as a visualization layer regardless of
whether Elastic is present — see Appendix A of the companion Low Level Design
document for Grafana + Elasticsearch datasource configuration.

Customers who have neither stack and are choosing one as a starting point should
begin with Prometheus and Grafana for infrastructure health monitoring, and plan
to add the Elastic Stack as their automation footprint grows and log observability
and audit requirements emerge.

---

## 10. Out of Scope

The following are not covered by this integration and are candidates for future work:

- IAG4 (Automation Gateway 4) — being retired; not included
- Elastic APM distributed tracing — a future candidate for deeper request-level instrumentation within IAP
- Elastic SIEM integration — security event correlation beyond authentication auditing
- Full Grafana dashboard set and Ansible deployment role for Grafana

---

## 11. Related Documents

- Itential + Elastic Stack — Low Level Design (companion document)
- `itential.monitoring` Ansible Collection Reference
- Elastic Stack 8.x Documentation
