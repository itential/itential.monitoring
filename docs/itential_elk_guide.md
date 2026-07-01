# Itential Log Observability with the Elastic Stack

## Table of Contents

- [Overview](#overview)
- [Why the Elastic Stack?](#why-the-elastic-stack)
  - [The Problem: Logs Scattered Across Many Servers](#the-problem-logs-scattered-across-many-servers)
  - [Operational Benefits](#operational-benefits)
  - [How It Relates to Prometheus and Grafana](#how-it-relates-to-prometheus-and-grafana)
  - [Licensing](#licensing)
- [Architecture](#architecture)
  - [Index Naming](#index-naming)
- [Server Requirements](#server-requirements)
  - [Elasticsearch](#elasticsearch)
  - [Logstash](#logstash)
  - [Kibana](#kibana)
  - [Filebeat](#filebeat)
  - [Production Topology Summary](#production-topology-summary)
  - [AWS EC2 Instance Mapping](#aws-ec2-instance-mapping)
- [Prerequisites](#prerequisites)
  - [TLS Certificates Required](#tls-certificates-required)
- [Installation](#installation)
  - [Step 1 — Add ELK Hosts to Inventory](#step-1--add-elk-hosts-to-inventory)
  - [Step 2 — Distribute TLS Certificates](#step-2--distribute-tls-certificates)
  - [Step 3 — Deploy the Full ELK Stack](#step-3--deploy-the-full-elk-stack)
  - [Step 4 — Deploy App-Specific Filebeat Inputs](#step-4--deploy-app-specific-filebeat-inputs)
  - [Step 5 — Configure Index Lifecycle Management (ILM)](#step-5--configure-index-lifecycle-management-ilm)
  - [Deploying Individual Components](#deploying-individual-components)
  - [Rerunning Specific Configuration Steps](#rerunning-specific-configuration-steps)
- [Air-Gapped Deployments](#air-gapped-deployments)
- [Role Reference](#role-reference)
- [Related Documents](#related-documents)

---

## Overview

The `itential.monitoring` Ansible collection includes roles to deploy and configure the
Elastic Stack — Elasticsearch, Logstash, Kibana, and Filebeat — as a centralized log
observability platform for Itential deployments. This guide covers why the Elastic Stack
is the right tool for Itential log observability, how to deploy it using the collection,
and how to use Kibana to analyze and troubleshoot automation operations.

---

## Why the Elastic Stack?

### The Problem: Logs Scattered Across Many Servers

A single automation workflow execution in Itential touches multiple components — IAP
orchestrates the job, its GatewayManager sub-process dispatches it to IAG5, and IAG5
executes the automation against one or more network devices. Each of these steps
generates log data on a different server, in a different log file. When something goes
wrong, diagnosing the failure requires logging into multiple servers, correlating
timestamps by hand, and assembling a picture from fragments spread across different
files. At scale, this is slow, error-prone, and unworkable.

The Elastic Stack solves this by continuously collecting log data from all Itential
servers into a single, searchable system. A failure that used to take an hour to
diagnose can be reconstructed in Kibana in minutes by querying all related events at
once.

### Operational Benefits

**Centralized log search** — Full-text search across all IAP and IAG5 log data from a
single Kibana interface. Filter by job ID, user, workflow name, device, or any field
in seconds, across any time range.

**End-to-end job tracing** — Automation workflows span IAP and IAG5. The Logstash
pipeline extracts and normalizes the correlation identifiers that link IAP's
GatewayManager logs to IAG5's gateway logs. A single query in Kibana can reconstruct
the complete execution trail — from job submission in IAP through device-level
execution in IAG5.

**Proactive alerting** — Kibana's alerting engine detects failure conditions and
notifies teams via email, Slack, or PagerDuty before they become outages. Alert on
automation failure spikes, gateway connectivity loss, rising job queue depth, or
platform authentication anomalies.

**Audit trail** — IAP's webserver access logs record every API call — who submitted
which automation, when, and what the outcome was. This data is indexed and searchable
in Kibana, supporting both internal audit requirements and regulatory compliance
obligations (SOX, PCI-DSS, HIPAA, and similar change management policies).

**Operational metrics** — Kibana dashboards can surface workflow success rates,
average execution times, top failing automations, and gateway utilization trends over
time — data that is invisible when logs remain siloed. This supports data-driven
prioritization of automation improvements and capacity planning.

### How It Relates to Prometheus and Grafana

Prometheus and Grafana are purpose-built for **metrics** — numeric, time-series data
sampled at regular intervals. They answer questions like "is Redis memory trending
toward capacity?" or "how many BullMQ jobs are queued right now?".

The Elastic Stack is purpose-built for **logs and events** — structured records
generated when something happens. It answers questions like "what exactly happened
during this failed workflow?" or "which user submitted the job that caused the error?".

The two stacks are complementary, not competing. For Itential deployments:

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

### Licensing

All features described in this guide — Elasticsearch storage and search, Kibana
dashboards, index lifecycle management, Logstash pipelines, and Filebeat agents —
are available under the **free Basic tier** of the Elastic Stack. No paid subscription
is required.

> **Note:** Third-party alerting connectors (PagerDuty, Slack) may require a paid
> Standard subscription for self-managed deployments. Email and webhook connectors are
> available in the free tier. Confirm connector availability against your intended
> subscription tier before relying on those integrations for production alerting.

---

## Architecture

The integration follows a standard log aggregation pattern with four components:

```
┌──────────────────────────────────┐       ┌──────────────────────────────────┐
│        Itential Platform         │       │          Elastic Stack           │
│                                  │       │                                  │
│  IAP ──► Filebeat ───────────────┼──────►│                                  │
│  IAG5 ──► Filebeat ──────────────┼──────►│  Logstash ──► Elasticsearch      │
│  MongoDB ──► Filebeat ───────────┼──────►│                    │             │
│  Redis ──► Filebeat ─────────────┼──────►│             Kibana ◄─────────────┤
│                                  │       │        (Dashboards, Alerting)    │
└──────────────────────────────────┘       └──────────────────────────────────┘
```

**Filebeat** runs alongside each Itential component on every server. It watches log
files and forwards new entries to Logstash as they are written. Filebeat is designed
to be lightweight — it adds no measurable overhead to the Itential processes it
monitors. It tracks its position in each log file so no events are lost if the network
is interrupted.

**Logstash** receives log events from Filebeat, enriches them with structured fields
(component identity, sub-process name, correlation IDs), tags failure conditions, and
routes events to the appropriate Elasticsearch index. Events tagged `automation_failure`
are written to a dedicated index for alerting, in addition to the main per-application
index.

**Elasticsearch** stores all log events in time-based indices, providing fast full-text
search and aggregation. Events are indexed as they arrive, making them immediately
searchable in Kibana.

**Kibana** provides the operational interface: log search and exploration, dashboards,
and alerting.

### Index Naming

Logstash routes events to indices named by application and date:

| Index Pattern | Contents |
|---|---|
| `itential-logs-itential-platform-service-<date>` | IAP sub-process logs |
| `itential-logs-itential-platform-main-<date>` | IAP main platform process |
| `itential-logs-itential-platform-web-<date>` | IAP webserver / HTTP API logs |
| `itential-logs-itential-gateway-server-<date>` | IAG5 gateway logs |
| `itential-logs-mongodb-<date>` | MongoDB logs |
| `itential-logs-redis-<date>` | Redis logs |
| `itential-failures-<date>` | All events tagged `automation_failure` (duplicated from above) |

The `itential-logs-*` wildcard covers all application indices and is the default
index pattern to use for broad searches in Kibana.

---

## Server Requirements

The specifications below are based on standard Elastic Stack sizing guidance for a
typical Itential deployment. Actual requirements will vary with log volume, the number
of Itential nodes shipping logs, and data retention period. Use these as a starting
point and monitor resource utilization after the initial deployment.

### Elasticsearch

Elasticsearch is the most resource-intensive component. Memory is the primary
constraint — the JVM heap must not exceed 50% of total host RAM, and must never
exceed 31 GB (the compressed-object-pointer boundary beyond which JVM performance
degrades). Disk sizing depends directly on log volume and the ILM retention period.

| Tier | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Development / lab (single node) | 4 | 16 GB | 200 GB SSD | Set `elasticsearch_heap_size: "8g"` |
| Small production (single node) | 8 | 32 GB | 1 TB SSD | Set `elasticsearch_heap_size: "16g"` |
| Production cluster (per node, 3 nodes) | 8 | 32 GB | 1 TB SSD per node | Set `elasticsearch_heap_size: "16g"` |

**Disk sizing guidance:** Calculate approximate daily ingest volume by multiplying the
number of Itential nodes by the expected events per second for that node type. A typical
IAP node generates 1–5 GB of log data per day under normal load; IAG5 generates less.
With 90-day ILM retention and daily rollover, size total disk as: `(peak daily GB × 90) × 1.5`
(the 1.5x multiplier accounts for replica shards and indexing overhead).

**SSD is strongly recommended.** Elasticsearch write performance on spinning disk is
significantly lower than on SSD and may not keep pace with log ingest at production
volumes.

### Logstash

Logstash's resource requirements scale with events-per-second throughput and the
number of pipeline workers (`logstash_pipeline_workers`, which defaults to the number
of vCPUs). For most Itential deployments, a single Logstash node is sufficient; two
nodes provide redundancy with Filebeat load-balancing across them.

| Tier | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Development / lab | 2 | 4 GB | 20 GB | Set `logstash_heap_size: "1g"` |
| Small production | 4 | 8 GB | 20 GB | Set `logstash_heap_size: "2g"` |
| Production | 8 | 16 GB | 20 GB | Set `logstash_heap_size: "4g"` |

Logstash is stateless — it does not store log data, so disk requirements are small
(pipeline configuration and logs only). The JVM heap should be set to no more than
50% of host RAM.

### Kibana

Kibana is a Node.js application. Its resource requirements are driven primarily by
the number of concurrent dashboard users and the complexity of queries executed. For
most Itential operations teams (5–20 users), a single node is sufficient.

| Tier | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Development / lab | 2 | 4 GB | 20 GB | |
| Small production | 4 | 8 GB | 20 GB | |
| Production | 4 | 8 GB | 20 GB | Scale horizontally behind a load balancer for more users |

### Filebeat

Filebeat runs on every Itential application server alongside the existing processes.
It is designed to be lightweight and does not require dedicated resources.

| Component | Additional vCPU | Additional RAM | Notes |
|---|---|---|---|
| Filebeat agent (per host) | 0.1–0.5 | 100–256 MB | Negligible impact on Itential processes |

No additional servers are required for Filebeat. It is deployed to the existing
IAP, IAG5, MongoDB, and Redis hosts.

### Production Topology Summary

The recommended production topology — two Logstash nodes, a three-node Elasticsearch
cluster, and one Kibana node — requires the following dedicated servers in addition to
the existing Itential infrastructure:

| Server | Count | Min vCPU | Min RAM | Min Disk |
|---|---|---|---|---|
| Elasticsearch node | 3 | 8 | 32 GB | 1 TB SSD |
| Logstash node | 2 | 4 | 8 GB | 20 GB |
| Kibana node | 1 | 4 | 8 GB | 20 GB |

For smaller or non-production deployments, Logstash and Kibana can be co-located on
a single server (combined 8 vCPU, 16 GB RAM, 40 GB disk), with a single-node
Elasticsearch instance on a separate server.

### AWS EC2 Instance Mapping

The tables below map the sizing tiers above to AWS EC2 instance types. All storage
recommendations use EBS gp3 volumes unless noted.

**Elasticsearch**

| Tier | EC2 Instance | vCPU | RAM | Storage | Notes |
|---|---|---|---|---|---|
| Development / lab | `m6g.xlarge` | 4 | 16 GB | 200 GB EBS gp3 | Set `elasticsearch_heap_size: "8g"` |
| Small production | `m6g.2xlarge` | 8 | 32 GB | 1 TB EBS gp3 | Set `elasticsearch_heap_size: "16g"` |
| Production (per node, ×3) | `m6g.2xlarge` | 8 | 32 GB | 1 TB EBS gp3 | Set `elasticsearch_heap_size: "16g"` |

> **Alternatives:** `r6g.2xlarge` (8 vCPU, 64 GB RAM) provides additional OS page cache
> headroom beyond the 16g heap, which benefits read-heavy query workloads. `i3.2xlarge`
> (8 vCPU, 61 GB RAM, 1.9 TB local NVMe) is a strong option when local NVMe throughput
> is preferred over EBS for high-volume log ingest.

**Logstash**

| Tier | EC2 Instance | vCPU | RAM | Storage | Notes |
|---|---|---|---|---|---|
| Development / lab | `t3.medium` | 2 | 4 GB | 20 GB EBS gp3 | Set `logstash_heap_size: "1g"` |
| Small production | `c6g.xlarge` | 4 | 8 GB | 20 GB EBS gp3 | Set `logstash_heap_size: "2g"` |
| Production | `c6g.2xlarge` | 8 | 16 GB | 20 GB EBS gp3 | Set `logstash_heap_size: "4g"` |

> Compute-optimized (`c6g`) instances are recommended over general-purpose (`m6g`) for
> Logstash because the grok and mutate filters in the Itential pipeline are CPU-bound.
> Logstash pipeline workers default to the number of vCPUs, so the `c6g.2xlarge` runs
> 8 workers in parallel. Disk is minimal — Logstash stores no log data.

**Kibana**

| Tier | EC2 Instance | vCPU | RAM | Storage | Notes |
|---|---|---|---|---|---|
| Development / lab | `t3.medium` | 2 | 4 GB | 20 GB EBS gp3 | |
| Small production | `c6g.xlarge` | 4 | 8 GB | 20 GB EBS gp3 | |
| Production | `c6g.xlarge` | 4 | 8 GB | 20 GB EBS gp3 | Add a second node behind an ALB to scale horizontally |

**Production Topology — EC2 Summary**

| Server | Count | EC2 Instance | EBS Volume |
|---|---|---|---|
| Elasticsearch node | 3 | `m6g.2xlarge` | 1 TB gp3 per node |
| Logstash node | 2 | `c6g.2xlarge` | 20 GB gp3 |
| Kibana node | 1 | `c6g.xlarge` | 20 GB gp3 |

---

## Prerequisites

Before running the ELK playbooks:

- Ansible 2.12 or later installed on the control node
- The `itential.monitoring` collection installed
- Target hosts running RHEL/CentOS 8+
- Internet access to `artifacts.elastic.co` on target hosts, or Elastic packages
  mirrored in a local Nexus repository for air-gapped environments
- TLS certificates for each component generated and distributed to the target hosts
  before the play runs (the roles do not generate or distribute certificates)
- For the Logstash keystore, the `logstash_keystore_password` and
  `logstash_elastic_password` variables must be set and encrypted with Ansible Vault

### TLS Certificates Required

| Host Group | Cert path variables |
|---|---|
| `elasticsearch` | `elasticsearch_pki_base_dir`, `elasticsearch_tls_cert_file`, `elasticsearch_tls_key_file`, `elasticsearch_tls_ca_file` |
| `logstash` | `logstash_pki_base_dir`, `logstash_tls_cert_file`, `logstash_tls_key_file`, `logstash_tls_ca_file` |
| `kibana` | `kibana_pki_base_dir`, `kibana_tls_cert_file`, `kibana_tls_key_file`, `kibana_tls_ca_file` |
| All Filebeat hosts | `filebeat_pki_base_dir`, `filebeat_tls_cert_file`, `filebeat_tls_key_file`, `filebeat_tls_ca_file` |

Certificate directories follow the `/etc/pki/<component>` standard. Private keys are placed
in a `private/` subdirectory under the base PKI directory. Certificates should be issued
from the same internal CA used for the rest of the Itential deployment.

---

## Installation

### Step 1 — Add ELK Hosts to Inventory

Add `elasticsearch`, `logstash`, and `kibana` groups to your inventory. Filebeat is
deployed to the existing Itential host groups — no additional groups are needed for it.

```yaml
all:
  vars:
    filebeat_environment: production
    filebeat_output_logstash_hosts:
      - "logstash-host.example.com:5044"

  children:
    elasticsearch:
      hosts:
        es-node-1.example.com:
      vars:
        elasticsearch_heap_size: "4g"

    logstash:
      hosts:
        logstash-1.example.com:
      vars:
        logstash_heap_size: "2g"
        logstash_elasticsearch_hosts:
          - "https://es-node-1.example.com:9200"
        logstash_keystore_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        logstash_elastic_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...

    kibana:
      hosts:
        kibana-1.example.com:
      vars:
        kibana_elasticsearch_hosts:
          - "https://es-node-1.example.com:9200"

    # Existing Itential host groups — Filebeat will be deployed to these
    platform:
      hosts:
        iap-1.example.com:
        iap-2.example.com:

    mongodb:
      hosts:
        mongo-1.example.com:
        mongo-2.example.com:
        mongo-3.example.com:

    redis:
      hosts:
        redis-1.example.com:
```

### Step 2 — Distribute TLS Certificates

Before running the playbooks, ensure TLS certificates are in place on each target host
at the paths configured in the role variables. All certificates should be signed by the
same internal CA.

For a minimal deployment, you need:

- **Elasticsearch host:** `ca.crt` and `<hostname>.crt` under `/etc/pki/elasticsearch/`; `<hostname>.key` under `/etc/pki/elasticsearch/private/`
- **Logstash host:** `ca.crt` and `<hostname>.crt` under `/etc/pki/logstash/`; `<hostname>.key` under `/etc/pki/logstash/private/`
- **Kibana host:** `ca.crt` and `<hostname>.crt` under `/etc/pki/kibana/`; `<hostname>.key` under `/etc/pki/kibana/private/`
- **All Filebeat hosts:** `ca.crt` and `<hostname>.crt` under `/etc/pki/filebeat/`; `<hostname>.key` under `/etc/pki/filebeat/private/`

### Step 3 — Deploy the Full ELK Stack

Run the `elk` playbook to deploy Elasticsearch, Logstash, Kibana, and Filebeat in a
single operation:

```bash
ansible-playbook itential.monitoring.elk -i <inventory> --ask-vault-pass
```

This playbook executes the following in order:

1. `itential.monitoring.elasticsearch` — installs and configures Elasticsearch
2. `itential.monitoring.logstash` — installs Logstash, deploys the Itential pipeline,
   and configures the keystore with `ELASTIC_PASSWORD`
3. `itential.monitoring.kibana` — installs and configures Kibana
4. `itential.monitoring.filebeat` — installs Filebeat and deploys the default syslog
   input on all Itential hosts

### Step 4 — Deploy App-Specific Filebeat Inputs

The `filebeat` playbook deploys app-specific log inputs to each host group based on
the services running there. Run it after the ELK stack is up:

```bash
ansible-playbook itential.monitoring.filebeat -i <inventory> --ask-vault-pass
```

This deploys the following inputs to the appropriate host groups:

| Host Group | Input Deployed | Logs Collected |
|---|---|---|
| `platform*` | `inputs.d/itential-platform.yml` | IAP main, webserver, and sub-process logs |
| `iag5_clients` | `inputs.d/itential-gateway-client.yml` | IAG5 gateway client logs |
| `iag5_servers` | `inputs.d/itential-gateway-server.yml` | IAG5 gateway server logs |
| `iag5_runners` | `inputs.d/itential-gateway-runner.yml` | IAG5 gateway runner logs |
| `mongodb*` | `inputs.d/mongodb.yml` | MongoDB logs |
| `redis_master`, `redis_replica` | `inputs.d/redis.yml` | Redis server logs |
| `redis_sentinel` | `inputs.d/redis-sentinel.yml` | Redis Sentinel / failover logs |

### Step 5 — Configure Index Lifecycle Management (ILM)

Apply an ILM policy in Kibana to automatically manage index aging and prevent unbounded
storage growth. Navigate to **Stack Management → Index Lifecycle Policies → Create Policy**
and configure the following phases:

| Phase | Trigger | Actions |
|---|---|---|
| Hot | Immediately on write | Roll over at 10 GB or 1 day |
| Warm | After 7 days | Shrink to 1 shard, force-merge to 1 segment |
| Cold | After 30 days | Freeze |
| Delete | After 90 days | Delete |

Adjust the retention window to match your internal data retention policy and any
applicable regulatory requirements.

### Deploying Individual Components

To deploy only specific components rather than the full stack:

```bash
# Elasticsearch only
ansible-playbook itential.monitoring.elasticsearch -i <inventory>

# Logstash only
ansible-playbook itential.monitoring.logstash -i <inventory> --ask-vault-pass

# Kibana only
ansible-playbook itential.monitoring.kibana -i <inventory>

# Filebeat only
ansible-playbook itential.monitoring.filebeat -i <inventory>
```

### Rerunning Specific Configuration Steps

Each role supports tags to re-run specific phases without a full reinstall:

```bash
# Redeploy Logstash pipeline config and keystore secrets only
ansible-playbook itential.monitoring.logstash -i <inventory> --tags logstash_configure --ask-vault-pass

# Redeploy all Filebeat input configurations
ansible-playbook itential.monitoring.filebeat -i <inventory> --tags filebeat_configure
```

---

## Air-Gapped Deployments

For environments without internet access, mirror Elastic packages in your existing
Nexus repository and configure the Ansible roles to use the local repository URL.

Override the repository base URL in your inventory:

```yaml
# For RHEL-family hosts — override the YUM repository baseurl in the role
# by pointing it at your Nexus mirror
elasticsearch_repo_baseurl: "https://nexus.example.com/repository/elastic-8.x/yum"
logstash_repo_baseurl: "https://nexus.example.com/repository/elastic-8.x/yum"
kibana_repo_baseurl: "https://nexus.example.com/repository/elastic-8.x/yum"
filebeat_repo_baseurl: "https://nexus.example.com/repository/elastic-8.x/yum"
```

Additionally, disable Kibana's outbound connections to prevent errors and noise in
air-gapped environments by adding the following to the Kibana configuration:

```yaml
# /etc/kibana/kibana.yml additions for air-gapped environments
map.includeElasticMapsService: false
telemetry.enabled: false
newsfeed.enabled: false
xpack.fleet.registryUrl: ""
```

---

## Role Reference

| Role | Playbook | Description |
|---|---|---|
| `itential.monitoring.elasticsearch` | `elasticsearch` | Installs and configures Elasticsearch |
| `itential.monitoring.logstash` | `logstash` | Installs Logstash, deploys pipeline, manages keystore |
| `itential.monitoring.kibana` | `kibana` | Installs and configures Kibana |
| `itential.monitoring.filebeat` | `filebeat` | Installs Filebeat and deploys log inputs per host group |

For full variable references, see each role's `README.md` in the `roles/` directory.

---

## Related Documents

- [Kibana Users Guide](kibana_users_guide.md)
- [High Level Design — Itential + Elastic Stack](itential_elk_hld.md)
- [Low Level Design — Itential + Elastic Stack](itential_elk_lld.md)
- `itential.monitoring` collection README
