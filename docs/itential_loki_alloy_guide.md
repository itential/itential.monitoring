# Itential Log Observability with Loki and Alloy

## Table of Contents

- [Overview](#overview)
- [Why Loki and Alloy?](#why-loki-and-alloy)
  - [The Problem: Logs Scattered Across Many Servers](#the-problem-logs-scattered-across-many-servers)
  - [Operational Benefits](#operational-benefits)
  - [How It Relates to Prometheus and Grafana](#how-it-relates-to-prometheus-and-grafana)
  - [How It Relates to the ELK Stack](#how-it-relates-to-the-elk-stack)
  - [Licensing](#licensing)
- [Architecture](#architecture)
  - [Log Streams and Labels](#log-streams-and-labels)
- [Server Requirements](#server-requirements)
  - [Loki](#loki)
  - [Alloy](#alloy)
  - [Topology Summary](#topology-summary)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Step 1 — Add Loki Host to Inventory](#step-1--add-loki-host-to-inventory)
  - [Step 2 — Set the Loki Push URL](#step-2--set-the-loki-push-url)
  - [Step 3 — Deploy Loki](#step-3--deploy-loki)
  - [Step 4 — Deploy Alloy](#step-4--deploy-alloy)
  - [Deploying Individual Components](#deploying-individual-components)
  - [Rerunning Specific Configuration Steps](#rerunning-specific-configuration-steps)
- [Retention](#retention)
- [Air-Gapped Deployments](#air-gapped-deployments)
- [Role Reference](#role-reference)
- [Related Documents](#related-documents)

---

## Overview

The `itential.monitoring` Ansible collection includes roles to deploy and configure Grafana
Loki and Grafana Alloy as a lightweight log observability stack for Itential deployments.
Loki stores and indexes log streams from all Itential components. Alloy runs on every
application node, collecting both the systemd journal and application log files, and
shipping them to Loki. Grafana queries Loki directly using LogQL, making log data available
in the same interface used for Prometheus metrics dashboards.

---

## Why Loki and Alloy?

### The Problem: Logs Scattered Across Many Servers

A single automation workflow execution in Itential touches multiple components — IAP
orchestrates the job, its GatewayManager sub-process dispatches it to IAG5, and IAG5
executes the automation against one or more network devices. Each step generates log data
on a different server, in a different log file. When something goes wrong, diagnosing the
failure requires logging into multiple servers, correlating timestamps by hand, and
assembling a picture from fragments spread across different files. At scale, this is slow,
error-prone, and unworkable.

Loki solves this by aggregating log streams from all Itential servers into a single,
queryable system. A failure that used to take an hour to diagnose can be investigated in
Grafana in minutes by querying all related streams at once.

### Operational Benefits

**Centralized log access** — Query logs from all IAP and IAG5 nodes from a single Grafana
panel. Filter by host, job, unit, or any label without logging into individual servers.

**Unified observability in Grafana** — Loki integrates directly into Grafana alongside
Prometheus metrics. You can correlate a metric anomaly (for example, a CPU spike on a
platform node) with log events from the same host and time window without switching tools.

**Lightweight footprint** — Loki is a single binary with no JVM, no cluster bootstrap,
and filesystem storage. It can run on the same VM as Grafana in smaller environments. Alloy
agents add negligible overhead to the application nodes they run on.

**Systemd journal collection** — Alloy captures all systemd unit logs automatically,
including service start/stop events, OOM kills, and authentication failures, without
requiring per-service log file configuration.

**Promtail replacement** — Alloy is the current Grafana agent, superseding Promtail. The
Alloy role stops and disables Promtail on any host where it is running.

### How It Relates to Prometheus and Grafana

Prometheus and Grafana are purpose-built for **metrics** — numeric, time-series data
sampled at regular intervals. They answer questions like "is Redis memory trending toward
capacity?" or "how many BullMQ jobs are queued right now?".

Loki is purpose-built for **logs** — structured or unstructured text events generated when
something happens. It answers questions like "what did the GatewayManager log when this
job failed?" or "which IAG5 host reported a command exit status of 1?".

Loki is designed as a companion to Prometheus and Grafana — it uses the same label-based
data model as Prometheus, making it familiar to query, and it integrates natively into
Grafana's explore and dashboard interfaces. If you are already running Prometheus and
Grafana, adding Loki requires no additional visualization tooling.

| Observability Need | Best Tool |
|---|---|
| IAP / IAG5 workflow execution logs | Loki |
| Systemd service events (start, stop, crash) | Loki |
| Cross-host log correlation by job or host | Loki |
| Redis / BullMQ queue depth over time | Prometheus + Grafana |
| IAP node CPU, memory, disk trends | Prometheus + Grafana |
| MongoDB performance metrics | Prometheus + Grafana |
| Infrastructure health dashboards | Prometheus + Grafana |

### How It Relates to the ELK Stack

The ELK Stack (Elasticsearch, Logstash, Kibana, Filebeat) and the Loki/Alloy stack both
provide centralized log aggregation for Itential, but they serve different operational
contexts.

| Capability | Loki / Alloy | ELK Stack |
|---|---|---|
| Grafana-native — same UI as metrics | Yes | No (Kibana separate) |
| Full-text search across log content | Via LogQL | Yes (Kibana KQL) |
| Log enrichment pipeline (field extraction, tagging) | Basic (labels only) | Rich (Logstash filters) |
| Pre-built Itential pipeline (GatewayManager parsing, job field extraction) | No | Yes |
| Infrastructure footprint | Minimal (1 binary + agent) | Significant (4 components, JVM) |
| Storage efficiency | High (gzip compression, index-free) | Lower (Elasticsearch indices) |
| Suitable for smaller / resource-constrained environments | Yes | With care |
| Suitable for large-scale, multi-team deployments | With LogQL expertise | Yes |

Use Loki/Alloy when you are already running Prometheus and Grafana and want log visibility
without deploying a second UI. Use the ELK Stack when you need the richer Logstash pipeline
for structured field extraction and tagging, or when your team prefers Kibana for log
analysis.

The two stacks are not mutually exclusive. Both can be deployed simultaneously using the
`itential.monitoring` collection.

### Licensing

Loki and Alloy are open-source projects licensed under the AGPLv3. All features described
in this guide are available at no cost. Grafana itself is available under the Apache 2.0
license.

---

## Architecture

The integration follows a simple agent-to-server log pipeline:

```
┌──────────────────────────────────────────┐     ┌───────────────────────────────┐
│           Itential Platform              │     │     Monitoring Infrastructure │
│                                          │     │                               │
│  IAP ──► Alloy (journal + app logs) ─────┼────►│                               │
│  IAG5 ──► Alloy (journal + app logs) ────┼────►│  Loki                         │
│  MongoDB ──► Alloy (journal + app logs) ─┼────►│    │                          │
│  Redis ──► Alloy (journal + app logs) ───┼────►│    │                          │
│                                          │     │  Grafana ◄────────────────────┤
└──────────────────────────────────────────┘     │  (Explore, Dashboards)        │
                                                 └───────────────────────────────┘
```

**Alloy** runs on every Itential application node as a systemd service. It collects two
sources of log data from each host:

- **Systemd journal** — all service events on the host, with the `unit` label populated
  from the systemd unit name. This captures service lifecycle events, OOM kills, and
  any service that writes to the journal.
- **Application log files** — specific log file paths configured per host group via the
  `alloy_log_paths` inventory variable. Each path is assigned a `job` label that
  identifies the application.

Alloy pushes log streams to Loki over HTTP using the Loki push API. It retains a small
local buffer to handle brief Loki unavailability without dropping events.

**Loki** receives log streams from all Alloy agents and stores them on the local
filesystem using gzip compression and a TSDB index. It exposes a query API on port 3100
that Grafana uses to execute LogQL queries.

**Grafana** is configured with a Loki datasource pointing at the Loki host. Log data is
available in the Explore view and can be included in any Grafana dashboard panel alongside
Prometheus metrics panels.

### Log Streams and Labels

Every log stream in Loki is identified by a set of labels. The Alloy configuration
attaches the following labels to each stream:

| Label | Value | Source |
|---|---|---|
| `job` | `iap-http`, `iap`, `iag5-server`, `iag5-runner`, `iag5-client`, `mongodb`, `redis`, `redis-sentinel`, `journal` | collection `playbooks/group_vars/` |
| `host` | Inventory hostname of the Alloy agent | `inventory_hostname` |
| `unit` | Systemd unit name (journal streams only) | Journal field `__journal__systemd_unit` |

These labels are the primary axes for filtering in Grafana's Explore view and in LogQL
queries. Keep label cardinality low — do not use high-cardinality values (like job IDs or
UUIDs) as labels.

---

## Server Requirements

### Loki

Loki is significantly lighter than Elasticsearch. It stores log data as gzip-compressed
chunks on the filesystem and maintains a small TSDB index. Memory and CPU requirements
are modest for most Itential deployments.

| Tier | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Development / lab | 2 | 4 GB | 100 GB | Can be co-located with Grafana |
| Small production | 4 | 8 GB | 500 GB SSD | Dedicated VM recommended |
| Production | 8 | 16 GB | 1 TB SSD | Scale disk based on retention and log volume |

**Disk sizing guidance:** Loki stores data at roughly 10–20% of raw log volume after
compression (gzip). A typical Itential deployment with 5–10 IAP and IAG5 nodes generates
approximately 2–10 GB of raw log data per day. With 30-day retention, size disk as:
`(peak daily GB × 0.15 × 30) × 1.5` for safety margin.

**SSD is recommended** for the Loki data directory but not strictly required. Unlike
Elasticsearch, Loki's write path is append-only and less sensitive to disk latency.

### Alloy

Alloy runs on every Itential application node alongside the existing processes. It is
designed to be lightweight.

| Component | Additional vCPU | Additional RAM | Notes |
|---|---|---|---|
| Alloy agent (per host) | 0.1–0.3 | 64–128 MB | Negligible impact on Itential processes |

No additional servers are required for Alloy. It is deployed to the existing IAP, IAG5,
MongoDB, and Redis hosts.

### Topology Summary

The minimal production topology requires one dedicated server for Loki (or co-location
with Grafana), with Alloy deployed to all existing Itential nodes:

| Server | Count | Min vCPU | Min RAM | Min Disk |
|---|---|---|---|---|
| Loki (dedicated) | 1 | 4 | 8 GB | 500 GB SSD |
| Alloy agent | One per Itential node | — | +128 MB per host | — |

For development and lab setups, Loki and Grafana can be co-located on a single VM
(4 vCPU, 8 GB RAM, 200 GB disk).

---

## Prerequisites

Before running the Loki and Alloy playbooks:

- Ansible 2.9.10 or later installed on the control node
- The `itential.monitoring` collection installed
- Target hosts running RHEL/Rocky Linux 9 (the roles use `dnf`)
- Internet access to `github.com` on the Loki host for binary download, or the binary
  pre-staged at `{{ loki_install_dir }}/loki` with the correct version
- Internet access to `rpm.grafana.com` on all Alloy hosts, or Alloy RPM mirrored
  in a local Nexus repository
- Prometheus and Grafana already deployed via `itential.monitoring.prometheus` and
  `itential.monitoring.grafana` if integrating the Loki datasource into an existing
  Grafana instance
- `alloy_loki_url` set in `all.vars` in inventory (private/VPC IP of the Loki host)

---

## Installation

### Step 1 — Add Loki Host to Inventory

Add a `loki` group to your inventory. For a dev or lab setup, the Loki host can be
the same VM as Grafana. For production, use a dedicated server.

```yaml
all:
  children:
    loki:
      hosts:
        <LOKI-HOST>:

    # For dev/lab — same VM as Grafana:
    grafana:
      hosts:
        <LOKI-HOST>:
      vars:
        grafana_loki_datasource_url: "http://<LOKI-PRIVATE-IP>:3100"

    # Existing Itential host groups — Alloy will be deployed to these
    platform:
      hosts:
        iap-1.example.com:
        iap-2.example.com:

    iag5_servers:
      hosts:
        iag-1.example.com:

    mongodb:
      hosts:
        mongo-1.example.com:
        mongo-2.example.com:
        mongo-3.example.com:

    redis_master:
      hosts:
        redis-1.example.com:

    redis_replica:
      hosts:
        redis-2.example.com:
        redis-3.example.com:
```

### Step 2 — Set the Loki Push URL

Set `alloy_loki_url` under `all.vars` in your inventory. This is the only Alloy variable
required for standard Itential deployments — log file paths and OS group memberships for
each host group are pre-configured in the collection's `playbooks/group_vars/`.

```yaml
all:
  vars:
    alloy_loki_url: "http://<LOKI-PRIVATE-IP>:3100"
```

> **Note:** Use the private/VPC-internal IP of the Loki host. Cloud instances cannot
> route to their own public IP, so Alloy agents will fail to push logs if a public IP
> is used.

### Step 3 — Deploy Loki

Run the Loki playbook to install and start the Loki server:

```bash
ansible-playbook itential.monitoring.loki -i <inventory>
```

This playbook runs two passes:

1. Creates the `loki` system user and group
2. Downloads and installs the Loki binary from GitHub releases
3. Creates the data and config directories
4. Deploys `loki-config.yml` and the systemd service file
5. Opens port 3100 in firewalld if the service is running
6. Starts Loki and waits for the `/ready` endpoint to respond
7. Provisions the Loki datasource in Grafana — writes the datasource file and restarts Grafana

Grafana must already be installed. `grafana_loki_datasource_url` must be set in inventory for
the `grafana` group (see Step 1).

### Step 4 — Deploy Alloy

Run the Alloy playbook to install and configure Alloy on all Itential application nodes:

```bash
ansible-playbook itential.monitoring.alloy -i <inventory>
```

This playbook:

1. Adds the Grafana RPM repository
2. Installs the Alloy package
3. Stops and disables Promtail if it is running
4. Adds the alloy user to `systemd-journal` and any configured `alloy_extra_groups`
5. Deploys `/etc/alloy/config.alloy` with the journal and file log source blocks
6. Opens port 12345 in firewalld if the service is running
7. Starts Alloy and asserts the service is active

### Deploying Individual Components

```bash
# Loki server only
ansible-playbook itential.monitoring.loki -i <inventory>

# Alloy agents only
ansible-playbook itential.monitoring.alloy -i <inventory>
```

### Rerunning Specific Configuration Steps

Each role supports tags to re-run specific phases without a full reinstall:

```bash
# Redeploy Loki config only (e.g. after changing retention or query limits)
ansible-playbook itential.monitoring.loki -i <inventory> --tags loki_configure

# Redeploy Alloy config only (e.g. after updating alloy_log_paths in group_vars)
ansible-playbook itential.monitoring.alloy -i <inventory> --tags alloy_configure
```

---

## Retention

Loki's default retention window is **7 days** (`loki_reject_old_samples_max_age: 168h`).
Adjust this to match your internal log retention policy. Common values:

| Retention | `loki_reject_old_samples_max_age` |
|---|---|
| 7 days | `168h` |
| 14 days | `336h` |
| 30 days | `720h` |
| 90 days | `2160h` |

Set the variable in your inventory on the `loki` host group vars:

```yaml
loki:
  vars:
    loki_reject_old_samples_max_age: "720h"   # 30 days
```

Then re-run the Loki playbook with the `loki_configure` tag:

```bash
ansible-playbook itential.monitoring.loki -i <inventory> --tags loki_configure
```

> **Note:** Loki does not enforce deletion of on-disk data when the retention window is
> changed. The `reject_old_samples_max_age` setting controls the ingest boundary — events
> older than this value are rejected at write time. For active purging of stored data,
> configure the `compactor` retention settings in `loki-config.yml`. Refer to the
> [Loki retention documentation](https://grafana.com/docs/loki/latest/operations/storage/retention/)
> for details.

---

## Air-Gapped Deployments

### Loki Binary

For environments without internet access, pre-stage the Loki binary on the target host
before running the playbook. Download the correct version from the
[Loki releases page](https://github.com/grafana/loki/releases) and place it at
`{{ loki_install_dir }}/loki` (default: `/usr/local/bin/loki`) with permissions `0755`.

The install task checks the installed version before attempting a download. If the binary
is already present at the correct version, the download is skipped automatically.

### Alloy RPM

For environments where `rpm.grafana.com` is not reachable, mirror the Alloy RPM in your
internal Nexus repository and override the repository baseurl in your inventory:

```yaml
# Override applied to all Alloy hosts
alloy_repo_baseurl: "https://nexus.example.com/repository/grafana-rpm"
```

> **Note:** The Alloy role currently adds the Grafana RPM repo using the public URL.
> If using a Nexus mirror, update the `yum_repository` task in
> `roles/alloy/tasks/main.yml` to use `alloy_repo_baseurl`.

---

## Role Reference

| Role | Playbook | Description |
|---|---|---|
| `itential.monitoring.loki` | `loki` | Installs Loki binary, deploys config and systemd service |
| `itential.monitoring.alloy` | `alloy` | Installs Alloy, deploys log collection config to all Itential nodes |

For full variable references, see each role's `README.md` in the `roles/` directory.

---

## Related Documents

- [ELK Installation Guide](itential_elk_guide.md)
- [Kibana Users Guide](itential_kibana_users_guide.md)
- `itential.monitoring` collection README
