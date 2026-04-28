# Itential + Elastic Stack Integration
## Low Level Design

## Introduction

This document provides detailed implementation guidance for integrating Itential's
automation platform with the Elastic Stack. It is intended for engineers responsible
for deploying and configuring the integration. For the business context, value
proposition, and high-level architecture, refer to the companion High Level Design
document.

### Itential Components Covered

| Component | Description |
|---|---|
| **IAP** (Itential Platform) | Workflow orchestration engine — jobs, tasks, workflows, and all associated sub-processes |
| **IAG5** (Automation Gateway 5) | Southbound automation gateway — interfaces with network devices via CLI, NETCONF, RESTCONF, and REST |

> **Note:** Automation Gateway 4 (IAG4) is not covered in this guide as it is being retired.
> Customers still running IAG4 should plan migration to IAG5.

### Supported Elastic Stack Version

Elastic Stack **8.x** is the target version for this guide. Elastic Stack 7.x has reached
end-of-life and is not recommended for new deployments.

---

## Section 1 — Log Sources & Data Inventory

### How Everything Fits Together

At a high level, this integration is about collecting data from Itential's components
and making it searchable, visualizable, and actionable in a central place. Here is the
flow from left to right:

Itential's applications — IAP and IAG5 — write log files to disk as they run. A
lightweight agent called **Filebeat** runs on each of those servers, watches those log
files, and ships new entries onward as they appear. In a production deployment, those
log entries flow into **Logstash**, which acts as a processing middle layer — it parses
and enriches the raw log data, adds useful fields, tags failures, and routes events to
the right destination. The processed log data is stored in **Elasticsearch**, which is
the search and storage engine at the heart of the stack. **Kibana** sits on top of
Elasticsearch and provides the user-facing interface — dashboards, charts, log search,
and alerting. Supporting infrastructure components like Redis and MongoDB also
contribute data, rounding out the full operational picture.

For customers already running **Prometheus** for metrics collection, the
`redis_exporter` component exposes Redis and BullMQ queue metrics in a format
Prometheus can scrape, which can then be forwarded into Elasticsearch alongside
the log data. Customers who already use **Grafana** for visualization can optionally
connect it directly to Elasticsearch as an alternative to Kibana.

The sections that follow describe each component in more detail, then walk through
installation, configuration, and deployment.

### Component Descriptions

**Elasticsearch** is the core storage and search engine for the integration. It receives
log events and metrics from the rest of the pipeline, indexes them for fast retrieval,
and answers queries from Kibana and Grafana. It is a distributed system, typically
deployed as a three-node cluster in production to provide redundancy and resilience.
All data in this integration ultimately lives in Elasticsearch, organized into time-based
indices (e.g. one index per day per application).

**Kibana** is the web-based user interface for Elasticsearch. It provides log search and
exploration via its Discover view, pre-built and custom dashboards, and an alerting
engine that can notify teams via Slack, PagerDuty, or email when defined conditions are
met. Kibana is the primary interface customers will use day-to-day to monitor Itential
operations.

**Logstash** is a data processing pipeline that sits between Filebeat and Elasticsearch.
It receives raw log events, applies filtering and transformation rules (written in a
simple Ruby-like configuration language), enriches events with additional fields, tags
failure conditions, and routes events to the appropriate Elasticsearch index. In simpler
deployments it can be omitted and Filebeat can ship directly to Elasticsearch, but
Logstash is recommended for production environments where enrichment and routing are
needed.

**Filebeat** is a lightweight log shipping agent that runs on each server alongside
Itential's applications. It watches configured log files for new entries and forwards
them onward — either to Logstash or directly to Elasticsearch. Filebeat is designed to
be low-overhead and resilient; it tracks its position in each log file so that no events
are lost if the network is interrupted or the destination is temporarily unavailable.

**Elastic Agent / Fleet** is an optional but recommended component that provides
centralized management of Filebeat and other Elastic data collection agents across all
servers. Fleet Server acts as a control plane — policies, configurations, and updates
can be pushed to all agents from a single Kibana interface rather than managed
individually per host.

**Prometheus** is an open-source metrics collection system. In this integration it is
used specifically to collect Redis and BullMQ queue metrics via the `redis_exporter`
agent. Prometheus scrapes metrics on a defined interval and can forward them to
Elasticsearch via a remote write adapter, making queue depth and job state data
available alongside log data in Kibana.

**redis_exporter** is a small agent that runs alongside Redis and exposes Redis
internals — memory usage, connected clients, command rates, and crucially BullMQ
queue depths and job state counts — as a Prometheus-compatible metrics endpoint.
It is deployed and managed as part of the `itential.monitoring` Ansible collection.

**Grafana** is an open-source visualization platform. While Kibana is the primary
visualization tool for this integration, customers who already have Grafana deployed
can connect it directly to Elasticsearch as a datasource. This allows Itential
operational data to be displayed alongside other monitoring sources such as network
device metrics. See Appendix A for details.

### Itential Component Log Sources

Before configuring any shipping or pipeline components, customers should understand
what each Itential component generates and where that data lives.

### IAP (Itential Platform)

IAP emits structured JSON logs. It is composed of a main platform process and multiple
sub-processes, each writing to its own log file under `/var/log/itential/platform/`.

**Main process logs:**

| File | Description |
|---|---|
| `platform.log` | Main platform process — startup, shutdown, health, core events |
| `webserver.log` | HTTP API layer — requests, responses, auth events |

**Sub-process logs:**

| Category | Files |
|---|---|
| Orchestration / execution | `WorkFlowEngine.log`, `OperationsManager.log`, `AGManager.log` |
| Gateway connectivity | `GatewayManager.log` |
| Design-time / studio | `AutomationStudio.log`, `WorkflowBuilder.log`, `FormBuilder.log`, `TemplateBuilder.log`, `JsonForms.log` |
| Platform services | `ConfigurationManager.log`, `Search.log`, `Tags.log`, `Jst.log`, `MOP.log` |

The orchestration and gateway connectivity logs are the highest value for operational
monitoring. `WorkFlowEngine.log` and `OperationsManager.log` contain job and task
execution events. `GatewayManager.log` records IAP-side communication with IAG5 instances
and is the key correlation point for end-to-end automation tracing.

### IAG5 (Automation Gateway 5)

IAG5 emits structured JSON logs to a single file:

| File | Description |
|---|---|
| `/var/log/gateway/gateway.log` | All gateway activity — device connections, automation execution, southbound protocol events |

`gateway.log` on the IAG5 side is the counterpart to `GatewayManager.log` on the IAP
side. Correlating events across these two files — using a shared job or correlation ID —
is the primary use case for end-to-end automation troubleshooting in Kibana.

### Supporting Infrastructure

| Component | Log Location | Notes |
|---|---|---|
| MongoDB | `/var/log/mongodb/mongod.log` | Enable slow query log for performance monitoring |
| Redis | `redis-cli SLOWLOG GET` | Slowlog entries; Sentinel state changes for HA deployments |
| Redis | `/var/log/redis/redis.log` | General Redis server events — startup, shutdown, configuration changes, replication events |
| Redis Sentinel | `/var/log/redis/sentinel.log` | Sentinel state changes, failover events, leader election — critical for HA deployments |
| BullMQ | Redis (via `redis_exporter`) | Queue depth and job state (waiting, active, completed, failed, delayed) monitored as Prometheus metrics via `itential.monitoring`; no separate log file |
| HAProxy | `/var/log/haproxy/` | If used in front of IAG5 or Vault |

---

## Section 2 — Architecture Patterns

Three integration patterns are supported, depending on environment maturity and
existing tooling.

### Pattern 1 — Direct (Filebeat → Elasticsearch)

Best for customers new to the Elastic Stack who want quick time-to-value. Filebeat
agents run alongside Itential components and ship logs directly to Elasticsearch.
No intermediate processing layer is required.

**When to use:** Greenfield ELK deployments, lower log volumes, no enrichment requirements.

### Pattern 2 — Enriched Pipeline (Filebeat → Logstash → Elasticsearch)

A Logstash instance sits between Filebeat and Elasticsearch, providing field
normalization, enrichment, tagging, and routing to multiple indices. This is the
recommended pattern for production deployments.

**When to use:** Production environments, multi-index routing, field enrichment,
log transformation requirements.

### Pattern 3 — Metrics (Prometheus → Elasticsearch)

For customers already running the `prometheus.prometheus` Ansible collection alongside
Itential's `itential.monitoring` collection. Structured metrics are shipped into
Elasticsearch alongside logs, enabling unified dashboards.

**When to use:** Customers with existing Prometheus infrastructure who want metrics
and logs in a single query plane.

---

## Section 3 — Filebeat Configuration

Filebeat is the recommended log shipper for all Itential deployments. The following
reference configuration should be adapted per customer environment.

Three discrete input stanzas are recommended for IAP rather than a single glob, to
allow clean per-category tagging and independent configuration:

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:

  # --- IAP main platform process ---
  - type: log
    enabled: true
    paths:
      - /var/log/itential/platform/platform.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: itential-platform-main
      environment: production
    fields_under_root: true

  # --- IAP webserver (HTTP API layer) ---
  - type: log
    enabled: true
    paths:
      - /var/log/itential/platform/webserver.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: itential-platform-web
      environment: production
    fields_under_root: true

  # --- IAP sub-process logs (all capital-letter named files) ---
  - type: log
    enabled: true
    paths:
      - /var/log/itential/platform/[A-Z]*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: itential-platform-service
      environment: production
    fields_under_root: true
    # sub-process name is extracted from log.file.path in Logstash

  # --- IAG5 (Automation Gateway 5) ---
  - type: log
    enabled: true
    paths:
      - /var/log/gateway/gateway.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: itential-gateway-5
      environment: production
    fields_under_root: true
    multiline.type: pattern
    multiline.pattern: '^\{'
    multiline.negate: true
    multiline.match: after

  # --- MongoDB ---
  - type: log
    enabled: true
    paths:
      - /var/log/mongodb/mongod.log
    fields:
      app: mongodb
      environment: production
    fields_under_root: true

  # --- Redis server ---
  - type: log
    enabled: true
    paths:
      - /var/log/redis/redis.log
    fields:
      app: redis
      environment: production
    fields_under_root: true

  # --- Redis Sentinel (HA deployments) ---
  - type: log
    enabled: true
    paths:
      - /var/log/redis/sentinel.log
    fields:
      app: redis-sentinel
      environment: production
    fields_under_root: true

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_fields:
      target: ''
      fields:
        vendor: itential

# Pattern 2: via Logstash (recommended)
output.logstash:
  hosts: ["logstash-host:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

# Pattern 1: direct to Elasticsearch (alternative)
# output.elasticsearch:
#   hosts: ["https://elasticsearch-host:9200"]
#   username: "filebeat_writer"
#   password: "${ELASTIC_PASSWORD}"
#   index: "itential-logs-%{+yyyy.MM.dd}"
#   ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

setup.kibana:
  host: "https://kibana-host:5601"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
```

### Key Configuration Notes

`json.keys_under_root: true` is critical for both IAP and IAG5. Both components emit
structured JSON logs, and this setting ensures fields are indexed at the top level in
Elasticsearch rather than nested under a `json` key.

The `[A-Z]*.log` glob for IAP sub-process logs captures all capital-letter named files
in a single input stanza. The sub-process name is derived from `log.file.path` in the
Logstash pipeline (see Section 4), making per-process filtering straightforward in Kibana.

TLS must be enforced for all shipper-to-Logstash and shipper-to-Elasticsearch
communication. This is particularly important in air-gapped enterprise environments
where certificate management is centralized.

---

## Section 4 — Logstash Pipeline (Pattern 2)

For production deployments, a Logstash pipeline provides field normalization,
sub-process name extraction, failure tagging, and index routing.

```ruby
# /etc/logstash/conf.d/itential.conf

input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key         => "/etc/logstash/certs/logstash.key"
  }
}

filter {

  # --- Extract IAP sub-process name from log file path ---
  if [app] == "itential-platform-service" {
    grok {
      match => {
        "[log][file][path]" => "/var/log/itential/platform/%{GREEDYDATA:itential.process}"
      }
    }
    mutate {
      gsub => ["itential.process", "\.log$", ""]
    }
  }

  # --- IAP main platform process enrichment ---
  if [app] == "itential-platform-main" {
    mutate { add_field => { "itential.process" => "platform" } }
  }

  # --- IAP webserver enrichment ---
  if [app] == "itential-platform-web" {
    mutate { add_field => { "itential.process" => "webserver" } }
  }

  # --- WorkFlowEngine: job/task execution events ---
  if [itential.process] == "WorkFlowEngine" {
    mutate {
      rename => { "job_id"        => "itential.job.id"          }
      rename => { "job_name"      => "itential.job.name"        }
      rename => { "task_id"       => "itential.task.id"         }
      rename => { "task_name"     => "itential.task.name"       }
      rename => { "workflow"      => "itential.workflow"         }
      rename => { "status"        => "itential.job.status"      }
      rename => { "duration"      => "itential.job.duration_ms" }
    }
    if [itential.job.status] == "error" or [itential.job.status] == "failed" {
      mutate { add_tag => ["automation_failure"] }
    }
  }

  # --- GatewayManager: IAP<->IAG5 communication events ---
  if [itential.process] == "GatewayManager" {
    mutate {
      rename => { "gateway_id"   => "itential.gateway.id"     }
      rename => { "gateway_name" => "itential.gateway.name"   }
      rename => { "operation"    => "itential.gateway.operation" }
      rename => { "status"       => "itential.gateway.status" }
    }
    if [itential.gateway.status] == "error" or [itential.gateway.status] == "unreachable" {
      mutate { add_tag => ["gateway_connectivity_failure"] }
    }
  }

  # --- IAG5: southbound device automation events ---
  if [app] == "itential-gateway-5" {
    mutate {
      rename => { "device"     => "itential.device.name"     }
      rename => { "protocol"   => "itential.device.protocol" }
      rename => { "operation"  => "itential.operation"       }
      rename => { "status"     => "itential.operation.status" }
      rename => { "job_id"     => "itential.job.id"          }
    }
    if [itential.operation.status] == "error" or [itential.operation.status] == "failed" {
      mutate { add_tag => ["automation_failure", "gateway_failure"] }
    }
  }

  # --- MongoDB slow query detection ---
  if [app] == "mongodb" {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp}\s+%{WORD:log_level}\s+%{GREEDYDATA:mongo_message}"
      }
    }
    if [mongo_message] =~ /ms$/ {
      mutate { add_tag => ["slow_query"] }
    }
  }

  # --- Normalize timestamp ---
  date {
    match        => ["timestamp", "ISO8601"]
    target       => "@timestamp"
    remove_field => ["timestamp"]
  }
}

output {
  # Route automation failures to dedicated index for alerting
  if "automation_failure" in [tags] {
    elasticsearch {
      hosts    => ["https://elasticsearch:9200"]
      index    => "itential-failures-%{+yyyy.MM.dd}"
      user     => "logstash_writer"
      password => "${ELASTIC_PASSWORD}"
      ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    }
  }

  # All logs routed by app name
  elasticsearch {
    hosts    => ["https://elasticsearch:9200"]
    index    => "itential-logs-%{app}-%{+yyyy.MM.dd}"
    user     => "logstash_writer"
    password => "${ELASTIC_PASSWORD}"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
  }
}
```

### Cross-Component Correlation

End-to-end tracing of an automation execution across IAP and IAG5 relies on two
distinct correlation identifiers that together cover the full execution trail:

**IAP Job ID** — a MongoDB ObjectID (e.g. `2365b6e1e9c4409ca356441f`) assigned when
a workflow job is created via `POST /jobs/start`. This ID appears throughout
`webserver.log` in subsequent polling requests (`GET /jobs/{id}`) and can be used
to reconstruct the full HTTP-level timeline of a job from submission to completion.

**Gateway Dispatch UUID** — a UUID (e.g. `36de0223-da67-4965-8d4f-4c3905d074bb`)
assigned by GatewayManager when a job is dispatched to an IAG5 gateway via BullMQ.
This UUID is passed to IAG5 as the JSON-RPC call ID and appears in both
`GatewayManager.log` and IAG5's `gateway.log`, making it the true cross-boundary
correlation key between IAP and IAG5.

The two IDs are temporally linked — the `POST /jobs/start` in `webserver.log`
triggers the GatewayManager UUID dispatch within milliseconds — but they are not
the same value. The relationship between them is timing-based at the log level,
though they may be explicitly linked in the IAP database.

The complete observable execution trail is:

| Step | Log File | Key Field |
|---|---|---|
| Job submitted | `webserver.log` | `POST /jobs/start` → IAP job ID in response URL |
| Job status polling | `webserver.log` | `GET /jobs/{iap_job_id}` |
| Gateway dispatch | `GatewayManager.log` | `itential.job.id` (UUID) + `itential.gateway.name` |
| RPC execution | `gateway.log` (IAG5) | `itential.rpc.id` (same UUID) |
| RPC response | `gateway.log` (IAG5) | `itential.rpc.id` (same UUID) |
| Job completion | `webserver.log` | `GET /jobs/{iap_job_id}` returns 200 |

In Kibana, a full end-to-end trace can be reconstructed by:
1. Filtering `webserver.log` on the IAP job ID to establish the job timeline
2. Using the dispatch timestamp to locate the corresponding GatewayManager UUID
3. Filtering `gateway.log` on that UUID to surface the IAG5 execution detail

---

## Section 5 — Index Strategy & ILM

Customers should adopt an Index Lifecycle Management (ILM) policy from day one
to prevent unbounded index growth.

**Index naming pattern:** `itential-logs-{app}-{date}`

Examples:
- `itential-logs-itential-platform-service-2025.03.24`
- `itential-logs-itential-gateway-5-2025.03.24`
- `itential-failures-2025.03.24`

**Index Lifecycle Policy** (apply via Kibana Dev Tools or Elasticsearch API):

```json
PUT _ilm/policy/itential-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "10gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink":     { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

The 90-day retention window is a reasonable starting point for most compliance
requirements. Customers should adjust based on internal data retention policies
and any applicable regulatory requirements.

---

## Section 6 — Kibana Dashboards & Saved Searches

Itential provides a starter set of exportable Kibana dashboard objects (`.ndjson`)
that customers can import via `Stack Management → Saved Objects → Import`.

### Automation Operations Dashboard

- Job execution rate over time — line chart bucketed by `itential.job.name`
- Job success/failure ratio — donut chart on `itential.job.status`
- Average job duration trend — line chart on `itential.job.duration_ms`
- Top 10 failing workflows — data table
- Live job execution stream — Discover-style log table, last 500 events

### Gateway Operations Dashboard

- IAG5 connectivity status over time
- Automation operations by target device (`itential.device.name`)
- Operations by protocol (`itential.device.protocol`)
- `GatewayManager.log` errors correlated with IAG5 `gateway.log` errors side-by-side
- End-to-end job trace by `itential.job.id` — spans IAP and IAG5 events

### Infrastructure Health Dashboard

- MongoDB slow queries over time
- Redis Sentinel state change events
- HAProxy upstream health (if applicable)
- BullMQ queue depth and job state over time (waiting, active, completed, failed, delayed) — sourced from `redis_exporter` metrics via Pattern 3

### Security & Audit Dashboard

- Authentication events by user (`webserver.log`)
- Failed login attempts over time
- RBAC permission denials
- API calls by endpoint — useful for anomaly detection

---

## Section 7 — Alerting Configuration

Kibana alerting rules should be configured under **Observability → Alerts** to
surface critical Itential failure conditions proactively.

| Rule Name | Condition | Severity | Suggested Action |
|---|---|---|---|
| Automation job failure spike | `automation_failure` tag count > 5 in 5 min | Critical | PagerDuty / Slack |
| Job queue depth rising | `itential.job.status: queued` count > 50 | Warning | Slack |
| IAG5 gateway connectivity failure | `gateway_connectivity_failure` tag count > 3 in 5 min | Critical | PagerDuty |
| MongoDB slow queries | `slow_query` tag count > 20 in 10 min | Warning | Email |
| Platform auth failures | Failed auth events > 10 in 5 min | High | PagerDuty |
| Filebeat shipping lag | No new docs in `itential-logs-*` for 5 min | High | PagerDuty |

---

## Section 8 — Air-Gapped / Offline Deployment

Many Itential customers operate in air-gapped environments. This section covers
the additional steps required to deploy the ELK integration without internet access.

### Package Distribution via Nexus

All Elastic packages must be mirrored into the customer's Nexus instance as RPM
repositories. The recommended Nexus repository structure:

```
nexus/
  repository/
    elastic-8.x/
      elasticsearch-8.x-x86_64.rpm
      kibana-8.x-x86_64.rpm
      logstash-8.x-x86_64.rpm
      filebeat-8.x-x86_64.rpm
```

Configure `/etc/yum.repos.d/elastic.repo` on all target hosts to point at the
Nexus group repository rather than `artifacts.elastic.co`.

### Certificate Management

All ELK TLS certificates should be issued from the customer's internal CA — the
same CA managing IAP and IAG5 gateway certificates. The same `ca.crt` used for
Itential TLS can serve as the trust anchor for Elasticsearch cluster communication,
simplifying certificate distribution.

Ansible roles in the `itential.monitoring` collection should be used to distribute
certificates to all ELK nodes as part of the standard deployment playbook.

### Disabling External Calls in Kibana

Several Kibana features attempt outbound connections that will fail silently or
generate noise in air-gapped environments. Disable them in `kibana.yml`:

```yaml
map.includeElasticMapsService: false
telemetry.enabled: false
newsfeed.enabled: false
xpack.fleet.registryUrl: ""
```

### SELinux Considerations

Customers running RHEL 8/9, Rocky Linux, or Oracle Linux with SELinux in enforcing
mode will need custom policy modules for Elasticsearch, Logstash, and Filebeat —
consistent with the approach used for IAP and IAG5. Key areas to address:

- Elasticsearch data and log directory contexts (`/var/lib/elasticsearch`, `/var/log/elasticsearch`)
- Logstash pipeline directory contexts
- Filebeat access to `/var/log/itential/` and `/var/log/gateway/`
- Network port labeling for 9200, 9300, 5044, and 5601

---

## Section 9 — Ansible Automation for Deployment

The ELK integration should be delivered as Ansible roles within the
`itential.monitoring` collection, enabling customers to deploy the full stack
as part of their existing Itential infrastructure automation playbooks.

### Proposed Role Structure

```
itential.monitoring/
  roles/
    elasticsearch/       # Install and configure ES cluster
    kibana/              # Install and configure Kibana
    logstash/            # Deploy pipeline configuration
    filebeat/            # Deploy filebeat.yml per host group
    elk_dashboards/      # Import Kibana saved objects via API
    elk_ilm/             # Apply ILM policies via ES API
    redis_exporter/      # Redis + BullMQ metrics (wraps prometheus.prometheus.redis_exporter)
```

### Example Playbook

```yaml
---
- name: Deploy ELK Stack for Itential observability
  hosts: elk_nodes
  become: true
  roles:
    - role: itential.monitoring.elasticsearch
    - role: itential.monitoring.kibana
    - role: itential.monitoring.logstash

- name: Deploy Filebeat to Itential app nodes
  hosts: itential_nodes
  become: true
  roles:
    - role: itential.monitoring.filebeat

- name: Configure Kibana dashboards and ILM
  hosts: elk_nodes[0]
  become: true
  roles:
    - role: itential.monitoring.elk_ilm
    - role: itential.monitoring.elk_dashboards
```

---

## Section 10 — Deployment Topology Reference

### Production Topology

The recommended production topology for a self-managed ELK deployment alongside
Itential includes:

- Two or more IAP nodes, each running a Filebeat agent
- One or more IAG5 nodes, each running a Filebeat agent
- Two Logstash nodes for pipeline processing redundancy (Filebeat load-balances across them)
- A three-node Elasticsearch cluster (two master-eligible data nodes, one data-only node)
- One Kibana node
- One Fleet Server node for centralized Elastic Agent management (optional)

All Elastic components and Filebeat packages are distributed via Nexus in
air-gapped environments. All inter-component communication is TLS-encrypted
using certificates issued by the customer's internal CA.

### Port Reference

| Source | Destination | Port | Protocol | Purpose |
|---|---|---|---|---|
| Filebeat | Logstash | 5044 | TCP/TLS | Log shipping (Beats protocol) |
| Filebeat | Elasticsearch | 9200 | HTTPS | Log shipping (Pattern 1 direct) |
| Logstash | Elasticsearch | 9200 | HTTPS | Processed log output |
| Kibana | Elasticsearch | 9200 | HTTPS | Query and management |
| Grafana | Elasticsearch | 9200 | HTTPS | Datasource queries |
| Elasticsearch | Elasticsearch | 9300 | TCP/TLS | Cluster transport |
| Users / browsers | Kibana | 5601 | HTTPS | Dashboard access |
| Ansible | Elasticsearch | 9200 | HTTPS | ILM and index template management |

---

## Appendix A — Grafana as an Alternative Visualization Layer

Customers with an existing Grafana deployment can add Elasticsearch as a native
datasource, enabling Itential operational data to appear alongside other monitoring
sources such as Prometheus device metrics or InfluxDB time-series data. This is
particularly useful for network operations teams who already manage interface
utilization, device health, and SNMP data in Grafana — IAP workflow execution and
IAG5 device automation events can be correlated in the same dashboard without
requiring adoption of Kibana.

Kibana remains the primary and fully-supported visualization path for this
integration. Grafana support is provided here as a starting point for customers
who prefer it. A full Grafana dashboard set and `itential.monitoring` Ansible role
for Grafana deployment are candidates for a future revision of this guide.

### Datasource Configuration

```yaml
# /etc/grafana/provisioning/datasources/elasticsearch.yaml

apiVersion: 1

datasources:
  - name: Itential-Elasticsearch
    type: elasticsearch
    access: proxy
    url: https://elasticsearch-host:9200
    basicAuth: true
    basicAuthUser: grafana_reader
    secureJsonData:
      basicAuthPassword: "${GRAFANA_ES_PASSWORD}"
    jsonData:
      index: "itential-logs-*"
      timeField: "@timestamp"
      esVersion: "8.0.0"
      logMessageField: message
      logLevelField: log.level
      tlsSkipVerify: false
      tlsCACert: "${ES_CA_CERT}"
    version: 1
    editable: false
```

### Elasticsearch Read-Only Role for Grafana

A dedicated `grafana_reader` role should be created in Elasticsearch with
read-only access to Itential indices. Do not reuse Kibana or Logstash credentials.

```json
POST _security/role/grafana_reader
{
  "indices": [
    {
      "names": ["itential-logs-*", "itential-failures-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}
```

### Air-Gapped Considerations

Add Grafana RPM packages to the Nexus mirror alongside the Elastic packages.
Disable Grafana's outbound telemetry, update checks, and plugin marketplace
in `/etc/grafana/grafana.ini`:

```ini
[analytics]
reporting_enabled  = false
check_for_updates  = false

[plugins]
allow_loading_unsigned_plugins =
plugin_catalog_url =

[grafana_net]
url =
```

---

## Appendix B — Troubleshooting

### Filebeat not shipping logs

- Verify Filebeat can reach Logstash or Elasticsearch: `curl -v telnet://logstash-host:5044`
- Check Filebeat harvest position in the registry: `/var/lib/filebeat/registry`
- Confirm SELinux is not blocking Filebeat reads on `/var/log/itential/` or `/var/log/gateway/`
- Review Filebeat logs at `/var/log/filebeat/filebeat`

### BullMQ queue metrics not appearing in Kibana

- Confirm `redis_exporter` is running on the Redis node: `systemctl status redis_exporter`
- Verify the Prometheus scrape target is reachable: `curl http://redis-host:9121/metrics`
- Confirm the Prometheus → Elasticsearch remote write path is configured and healthy
- Check that the `itential.monitoring` `redis_exporter` role was included in the deployment playbook

### JSON fields not appearing at top level in Elasticsearch

- Confirm `json.keys_under_root: true` is set in the relevant Filebeat input stanza
- Verify the log file is actually emitting valid JSON (malformed JSON falls back to a raw `message` field)

### Index mapping conflicts

- A mapping conflict occurs when the same field name appears with different data types
  across log sources. Use the Logstash `mutate` filter to normalize field types before
  indexing, or use separate index templates per `app` value.

### TLS / certificate errors

- Confirm the CA certificate distributed to Filebeat, Logstash, and Elasticsearch
  nodes is the same internal CA that signed the server certificates
- Check for SAN mismatches: the Elasticsearch node hostname used in `hosts:` must
  match a SAN on the Elasticsearch TLS certificate
- On RHEL-family systems, also update the system trust store:
  `update-ca-trust extract`

### GatewayManager and IAG5 events not correlating

- Confirm both IAP and IAG5 are writing a shared `job_id` field to their respective logs
- Verify the Logstash pipeline is renaming `job_id` to `itential.job.id` in both the
  `GatewayManager` and `itential-gateway-5` filter blocks
- In Kibana Discover, filter on `itential.job.id: "<value>"` across all indices
  using the `itential-logs-*` index pattern

---

*This document is a living reference. Sections covering IAP sub-process field
mappings and Kibana dashboard exports will be updated once field-level log samples
are available from IAP sub-process logs.*
