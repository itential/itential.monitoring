# Ansible Collection - itential.monitoring

The playbooks and roles discussed in this guide install and configure three independent monitoring
stacks for the Itential platform:

- **Prometheus / Grafana** — installs [Prometheus](https://prometheus.io/),
  [Grafana](https://grafana.com/), and a set of metrics exporters. Prometheus and Grafana can be
  installed on separate hosts or co-located together, but should not be co-located with
  Itential-related hosts. Exporters are installed on the Itential hosts where they expose metrics
  (e.g. the MongoDB exporter runs on `mongodb` hosts).

- **ELK Stack** — installs [Elasticsearch](https://www.elastic.co/elasticsearch/),
  [Logstash](https://www.elastic.co/logstash/), [Kibana](https://www.elastic.co/kibana/), and
  [Filebeat](https://www.elastic.co/beats/filebeat/) to provide centralized log aggregation and
  search across all Itential hosts. Elasticsearch, Logstash, and Kibana each run on dedicated
  hosts. Filebeat is deployed to all Itential application nodes to ship logs.

- **Loki / Alloy** — installs [Grafana Loki](https://grafana.com/docs/loki/latest/) and
  [Grafana Alloy](https://grafana.com/docs/alloy/latest/) to provide lightweight log aggregation
  tightly integrated with the Grafana dashboards. Loki runs on a dedicated host (or co-located
  with Grafana). Alloy is deployed to all Itential application nodes to collect systemd journal
  events and application log files and ship them to Loki.

**&#9432; Note:**
These are optional playbooks and roles and are not required for operation of the Itential platform.

## Overview

This collection provides three independent monitoring stacks:

- **Prometheus / Grafana** — metrics collection and dashboards
- **ELK Stack** — log aggregation and search (Elasticsearch, Logstash, Kibana, Filebeat)
- **Loki / Alloy** — lightweight log aggregation integrated with Grafana (Loki, Alloy)

All stacks are optional and can be deployed independently or together.

## Setup

### Install Prometheus Collection

In order to run the Prometheus playbooks and roles, the `prometheus.prometheus` collection must be
installed manually. It does not get installed automatically when the `itential.monitoring` collection
is installed. Also, this playbook requires prometheus version >= 0.22.0.

```bash
ansible-galaxy collection install prometheus.prometheus
```

### MacOS Requirements

GNU tar must be installed locally.

```bash
brew install gnu-tar
```

Then add GNU tar to your PATH by adding this to your `~/.zshrc` or `~/.bash_profile`:

```bash
export PATH="/opt/homebrew/opt/gnu-tar/libexec/gnubin:$PATH"
```

If you see the following error when running the Prometheus-related playbooks:

```bash
objc[58735]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called.
objc[58735]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
ERROR! A worker was found in a dead state
```

Add this environment variable to your `~/.zshrc` or `~/.bash_profile`:

```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

## Roles

### ELK Stack Roles

#### Elasticsearch Role

The `itential.monitoring.elasticsearch` role installs and configures
[Elasticsearch](https://www.elastic.co/elasticsearch/) as the log storage backend. It supports
single-node and multi-node cluster deployments. xpack security and TLS are enabled by default.

#### Logstash Role

The `itential.monitoring.logstash` role installs and configures
[Logstash](https://www.elastic.co/logstash/) as the log processing pipeline. It receives events
from Filebeat over the Beats protocol, enriches and routes them by the `app` field, and writes
them to Elasticsearch. The role manages the Logstash keystore, storing the Elasticsearch
`logstash_writer` password as `ELASTIC_PASSWORD` and configuring the service environment so
Logstash can decrypt the keystore at startup.

#### Kibana Role

The `itential.monitoring.kibana` role installs and configures
[Kibana](https://www.elastic.co/kibana/) as the log visualization UI. It connects to
Elasticsearch and optionally serves HTTPS when TLS is enabled.

#### Filebeat Role

The `itential.monitoring.filebeat` role installs and configures
[Filebeat](https://www.elastic.co/beats/filebeat/) on Itential application nodes. Inputs are
managed as drop-in files under `/etc/filebeat/inputs.d/` and ship logs to Logstash (default)
or directly to Elasticsearch. App-specific input configurations are available for:

| Host Group | Input |
|---|---|
| `platform*` | IAP main, web, and sub-process logs |
| `iag5_clients` | IAG5 gateway client logs |
| `iag5_servers` | IAG5 gateway server logs |
| `iag5_runners` | IAG5 gateway runner logs |
| `mongodb*` | MongoDB logs |
| `redis_master`, `redis_replica` | Redis logs |
| `redis_sentinel` | Redis Sentinel logs |

---

### Loki / Alloy Roles

#### Loki Role

The `itential.monitoring.loki` role installs and configures
[Grafana Loki](https://grafana.com/docs/loki/latest/) as the log storage backend. Loki is
installed as a binary downloaded from GitHub releases and runs under a dedicated systemd
service. It uses filesystem storage and is suitable for single-node deployments. The role
opens port 3100 in firewalld when the service is active.

#### Alloy Role

The `itential.monitoring.alloy` role installs and configures
[Grafana Alloy](https://grafana.com/docs/alloy/latest/) on Itential application nodes.
Alloy collects the systemd journal and application log files from each host and ships them
to Loki. It is installed via the Grafana RPM repository. The role stops and disables
Promtail if it is running. Log file paths and OS group memberships for each host group are
pre-configured in the collection's `playbooks/group_vars/`:

| Host Group | Log paths (`job` label) | `alloy_extra_groups` |
|---|---|---|
| `platform*` | webserver.log (`iap-http`), platform/*.log (`iap`) | — |
| `iag5_servers` | gateway.log (`iag5-server`) | `[itential]` |
| `iag5_runners` | gateway.log (`iag5-runner`) | `[itential]` |
| `iag5_clients` | gateway.log (`iag5-client`) | `[itential]` |
| `mongodb*` | mongod.log (`mongodb`) | `[mongod]` |
| `redis_master`, `redis_replica` | redis.log (`redis`) | — |
| `redis_sentinel` | sentinel.log (`redis-sentinel`) | — |

The only variable required in inventory is `alloy_loki_url` set under `all.vars`.

---

### Prometheus / Grafana Roles

There are currently two `itential.monitoring` and several `prometheus.prometheus` roles responsible
for installing all of the necessary components.

## Monitoring Roles

### Prometheus Install Role

The `itential.monitoring.prometheus` role performs a base installation of
[Prometheus](https://prometheus.io/) by calling the `prometheus.prometheus.prometheus` role. It
then configures the Prometheus scrape targets dynamically based on the Itential-related hosts in
the inventory. By default, the scrape targets will use the `inventory_hostname` of the host and the
exporter default web listen port.  Optionally, the scrape targets can be specified using the
`<exporter>_web_listen_address` in the inventory.

### Grafana Install Role

The `itential.monitoring.grafana` role performs a base installation of
[Grafana](https://grafana.com/). The Grafana installation will include some basic dashboards that
can immediately provide monitoring value of your Itential installation.

## Prometheus Roles

The Prometheus and Exporter roles in this section are part of the
[Ansible Prometheus Collection](https://galaxy.ansible.com/ui/repo/published/prometheus/prometheus/).

### Prometheus Role

The `itential.monitoring` uses the community Prometheus role. Refer to the [`prometheus.prometheus.prometheus`](https://prometheus-community.github.io/ansible/branch/main/prometheus_role.html#ansible-collections-prometheus-prometheus-prometheus-role) documentation for all available configuration parameters.

### Exporter Roles

The `itential.monitoring` uses the following community Prometheus exporter roles (refer to the linked documentation for all available configuration parameters):

- [`prometheus.prometheus.redis_exporter`](https://prometheus-community.github.io/ansible/branch/main/redis_exporter_role.html#ansible-collections-prometheus-prometheus-redis-exporter-role)
- [`prometheus.prometheus.mongodb_exporter`](https://prometheus-community.github.io/ansible/branch/main/mongodb_exporter_role.html#ansible-collections-prometheus-prometheus-mongodb-exporter-role)
- [`prometheus.prometheus.node_exporter`](https://prometheus-community.github.io/ansible/branch/main/node_exporter_role.html#ansible-collections-prometheus-prometheus-node-exporter-role)
- [`prometheus.prometheus.process_exporter`](https://prometheus-community.github.io/ansible/branch/main/process_exporter_role.html#ansible-collections-prometheus-prometheus-process-exporter-role)

In addition, `itential.monitoring` ships its own exporter role — the `itential.monitoring.itential_platform_exporter` role installs the
[itential-job-metrics-exporter](https://github.com/itential/job-metrics-exporter), which connects
directly to the Itential Platform MongoDB replica set and exposes job/task lifecycle metrics.
Unlike the exporters above, it is downloaded from GitHub releases rather than installed via the
`prometheus.prometheus` collection. Refer to the [role README](roles/itential_platform_exporter/README.md)
for full configuration details and MongoDB prerequisites.

Each exporter is a lightweight Go application that exposes the metrics on a standard HTTP endpoint. Each exporter requires a port to be opened so that Prometheus can access the metrics that are being exposed by the exporter. The following ports will be utilized by these roles:

| Exporter | Default Port | Description |
| -------- | ------------ | ----------- |
| [node exporter](https://github.com/prometheus/node_exporter) | 9100 | The node exporter is installed on all Itential-related hosts and will expose system and sysadmin type metrics. |
| [process exporter](https://github.com/ncabatoff/process-exporter) | 9256 | The process exporter is installed on `platform` and `gateway` hosts and will expose individual processes from Itential Platform and IAG. |
| [mongodb exporter](https://github.com/percona/mongodb_exporter) | 9216 | The mongo exporter is installed on `mongodb` hosts and will expose information about the MongoDB installation and any replica sets. |
| [redis exporter](https://github.com/oliver006/redis_exporter) | 9121 | The redis exporter is installed on `redis` hosts and will expose information about the Redis installation and any replica sets. |
| [itential platform exporter](https://github.com/itential/job-metrics-exporter) | 9477 | The Itential Platform exporter is installed on `itential_platform_exporter` hosts and will expose job/task lifecycle metrics read from the Itential Platform MongoDB replica set. |

#### Process Exporter Notes

The process exporter role by default will monitor all processes on the host. To monitor only
Itential-related processes, add the `process_exporter_names` to the groups vars in the inventory.
Refer to the [Example Inventory](#example-inventory) section.

## Variables

### Global Variables

There are no global variables.

### ELK Stack Role Variables

#### Elasticsearch Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `elasticsearch_version` | String | Elasticsearch version to install | `8.13.0` |
| `elasticsearch_http_port` | Integer | HTTP API port | `9200` |
| `elasticsearch_transport_port` | Integer | Inter-node transport port | `9300` |
| `elasticsearch_cluster_name` | String | Cluster name | `itential-monitoring` |
| `elasticsearch_heap_size` | String | JVM heap size | `1g` |
| `elasticsearch_tls_enabled` | Boolean | Enable xpack security and TLS | `true` |
| `elasticsearch_discovery_seed_hosts` | List | Seed hosts for cluster discovery | `[]` |
| `elasticsearch_cluster_initial_master_nodes` | List | Initial master nodes for bootstrap | `[]` |

#### Logstash Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `logstash_version` | String | Logstash version to install | `8.13.0` |
| `logstash_heap_size` | String | JVM heap size | `1g` |
| `logstash_beats_port` | Integer | Port for the Beats input | `5044` |
| `logstash_tls_enabled` | Boolean | Enable TLS for Beats input and Elasticsearch output | `true` |
| `logstash_elasticsearch_hosts` | List | Elasticsearch output hosts | `["https://localhost:9200"]` |
| `logstash_keystore_password` | String | Logstash keystore password — **must be set via Ansible Vault** | `""` |
| `logstash_elastic_password` | String | Password for the `logstash_writer` Elasticsearch user — **must be set via Ansible Vault** | `""` |
| `logstash_pipeline_workers` | Integer | Pipeline worker threads | `ansible_processor_vcpus` |

#### Kibana Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `kibana_version` | String | Kibana version to install | `8.13.0` |
| `kibana_server_port` | Integer | Port Kibana listens on | `5601` |
| `kibana_elasticsearch_hosts` | List | Elasticsearch hosts to connect to | `["https://localhost:9200"]` |
| `kibana_tls_enabled` | Boolean | Enable TLS for Kibana server and Elasticsearch connection | `true` |

#### Filebeat Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `filebeat_version` | String | Filebeat version to install | `8.13.0` |
| `filebeat_environment` | String | Environment label attached to every event (e.g. `production`) | unset |
| `filebeat_tls_enabled` | Boolean | Enable TLS for the output | `true` |
| `filebeat_output_logstash_enabled` | Boolean | Send output to Logstash | `true` |
| `filebeat_output_logstash_hosts` | List | Logstash hosts | `["localhost:5044"]` |
| `filebeat_output_elasticsearch_enabled` | Boolean | Send output directly to Elasticsearch | `false` |

### Prometheus Role Variables

All Prometheus variables are handled by the `prometheus.prometheus.prometheus` role.  Refer to the
[Prometheus Role](#prometheus-role) section.

### Exporters Role Variables

All community exporter variables are handled by the exporter roles. Refer to the documentation
links in the [Exporter Roles](#exporter-roles) section. The `itential_platform_exporter` role's
variables are its own — see [Itential Platform Exporter Role Variables](#itential-platform-exporter-role-variables) below.

### Redis Exporter Requirements

In order for the Redis exporter to function, the `redis_prometheus_user_enabled` variable on the
`redis` hosts must be set to `true`. By default, this variable is set to `false` and the
`prometheus` user is not create.

### MongoDB Exporter Recommendations

We recommend setting the `mongodb_exporter_global_conn_pool` variable to `true` in the `mongodb`
group variables section when MongoDB replication is enabled.  Otherwise the exporter may consume
all available file descriptors and cause the mongod process to crash.

```yaml
all:
  children:
    mongodb:
      hosts:
        <MONGODB-HOST-1>:
        <MONGODB-HOST-N>:
    vars:
      mongodb_exporter_global_conn_pool: true
```

### Grafana Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `grafana_user` | String | The Grafana linux user. | `grafana` |
| `grafana_group` | String | The Grafana linux group. | `grafana` |
| `grafana_port` | Integer | The Grafana port that is used by the application. | `3000` |
| `grafana_repo_url` | String | The public URL where the grafana application can be downloaded from. | `https://rpm.grafana.com` |
| `grafana_gpg_key` | String | The public URL where the grafana gpg key can be downloaded from. | `https://rpm.grafana.com/gpg.key` |
| `grafana_install_dir` | String | The root installation directory where grafana will be installed. | `/etc/grafana` |
| `grafana_dashboard_dir` | String | The directory path where the dashboards are uploaded to. | `/etc/grafana/provisioning/dashboards` |
| `grafana_allow_ui_updates` | Boolean | A flag to enable/disable saving dashboards in the grafana UI. | `false` |
| `grafana_loki_datasource_enabled` | Boolean | Provision a Loki datasource in Grafana. Set to `true` after Loki is deployed. | `false` |
| `grafana_loki_datasource_url` | String | Loki URL for the Grafana datasource (e.g. `http://<LOKI-HOST>:3100`). Required when `grafana_loki_datasource_enabled` is `true`. | `""` |

### Loki Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `loki_version` | String | Loki version to install | `3.7.1` |
| `loki_http_listen_port` | Integer | HTTP API and push endpoint port | `3100` |
| `loki_grpc_listen_port` | Integer | gRPC port | `9096` |
| `loki_data_dir` | String | Loki data directory | `/var/lib/loki` |
| `loki_config_dir` | String | Loki config directory | `/etc/loki` |
| `loki_install_dir` | String | Directory for the Loki binary | `/usr/local/bin` |
| `loki_reject_old_samples_max_age` | String | Reject logs older than this age (e.g. `168h` = 7 days, `720h` = 30 days) | `168h` |
| `loki_max_entries_limit` | Integer | Max log entries returned per query | `5000` |
| `loki_ingestion_rate_mb` | Integer | Ingestion rate limit in MB/s | `16` |
| `loki_ingestion_burst_size_mb` | Integer | Ingestion burst size in MB | `32` |
| `loki_tls_enabled` | Boolean | Enable TLS on the Loki HTTP listener | `false` |
| `loki_tls_cert_file` | String | Path to the Loki server certificate file (required when `loki_tls_enabled: true`) | `/etc/loki/certs/loki.crt` |
| `loki_tls_key_file` | String | Path to the Loki server private key file (required when `loki_tls_enabled: true`) | `/etc/loki/certs/loki.key` |

### Alloy Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `alloy_loki_url` | String | Loki push endpoint URL. Set once under `all.vars` in inventory (use private/VPC IP). Use `https://` when `alloy_tls_enabled: true`. | `""` |
| `alloy_http_listen_port` | Integer | Alloy HTTP port for metrics, health, and UI | `12345` |
| `alloy_log_paths` | List | File-based log paths to tail. Pre-configured per host group in `playbooks/group_vars/`. Override in inventory to customise. | `[]` |
| `alloy_extra_groups` | List | Extra OS groups to add the alloy user to for log file read access. Pre-configured for `mongodb*` and `iag5*` groups. | `[]` |
| `alloy_tls_enabled` | Boolean | Enable TLS for the Alloy → Loki push connection | `false` |
| `alloy_tls_ca_file` | String | Path to the CA certificate used to verify the Loki server cert (required when `alloy_tls_enabled: true`) | `/etc/alloy/certs/ca.crt` |

### Itential Platform Exporter Role Variables

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `itential_platform_exporter_version` | String | Exporter version to install | `1.0.1` |
| `itential_platform_exporter_listen_address` | String | Address/port the exporter listens on | `:9477` |
| `itential_platform_exporter_mongo_uri` | String | Full MongoDB connection string (takes precedence over the individual `_mongo_host`/`_mongo_port`/... fields) | `""` |
| `itential_platform_exporter_mongo_password` | String | MongoDB password — **must be set via Ansible Vault** | `""` |
| `itential_platform_exporter_tls_enabled` | Boolean | Enable TLS on the exporter's own HTTP listener | `false` |
| `itential_platform_exporter_change_stream_enabled` | Boolean | Enable real-time MongoDB change stream counters (requires a replica set) | `true` |
| `itential_platform_exporter_polling_enabled` | Boolean | Enable background aggregation polling (use when change streams are unavailable) | `false` |

See the [role README](roles/itential_platform_exporter/README.md) for the full variable list and
required MongoDB setup steps (dedicated read-only user and indexes).

## Building Your Inventory

### ELK Stack Inventory

Add `elasticsearch`, `logstash`, and `kibana` groups for the ELK server components. Filebeat is
deployed to the existing Itential host groups — no additional groups are needed for it.

TLS certificates must be distributed to each host before the play runs. Each role expects
its certs under `/etc/<role>/certs/` by default (configurable via the `*_tls_cert`,
`*_tls_key`, and `*_tls_ca_cert` variables).

#### ELK Example Inventory

```yaml
all:
  vars:
    filebeat_environment: production
    filebeat_output_logstash_hosts:
      - "logstash-host:5044"

  children:
    elasticsearch:
      hosts:
        <ELASTICSEARCH-HOST>:
      vars:
        elasticsearch_heap_size: "4g"

    logstash:
      hosts:
        <LOGSTASH-HOST>:
      vars:
        logstash_heap_size: "2g"
        logstash_elasticsearch_hosts:
          - "https://<ELASTICSEARCH-HOST>:9200"
        logstash_keystore_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        logstash_elastic_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...

    kibana:
      hosts:
        <KIBANA-HOST>:
      vars:
        kibana_elasticsearch_hosts:
          - "https://<ELASTICSEARCH-HOST>:9200"

    platform:
      hosts:
        <PLATFORM-HOST-1>:
        <PLATFORM-HOST-N>:

    mongodb:
      hosts:
        <MONGODB-HOST-1>:
        <MONGODB-HOST-N>:

    redis:
      hosts:
        <REDIS-HOST-1>:
        <REDIS-HOST-N>:
```

### Loki / Alloy Inventory

Add a `loki` group for the Loki server. For a dev or lab setup, this can be the same VM
as `grafana`. Set `alloy_loki_url` once under `all.vars` — log paths and OS group
memberships for each host group are pre-configured in the collection and require no
additional inventory config.

#### Loki / Alloy Example Inventory

```yaml
all:
  vars:
    alloy_loki_url: "http://<LOKI-PRIVATE-IP>:3100"   # private/VPC IP — set once here

  children:
    loki:
      hosts:
        <LOKI-HOST>:

    grafana:
      hosts:
        <GRAFANA-HOST>:   # can be the same VM as loki
      vars:
        grafana_loki_datasource_url: "http://<LOKI-PRIVATE-IP>:3100"

    platform:
      hosts:
        <PLATFORM-HOST-1>:
        <PLATFORM-HOST-N>:

    mongodb:
      hosts:
        <MONGODB-HOST-1>:
        <MONGODB-HOST-N>:

    redis_master:
      hosts:
        <REDIS-MASTER>:

    redis_replica:
      hosts:
        <REDIS-REPLICA-1>:
        <REDIS-REPLICA-N>:

    redis_sentinel:
      hosts:
        <REDIS-SENTINEL-1>:
        <REDIS-SENTINEL-N>:

    iag5_servers:
      hosts:
        <IAG5-SERVER>:
```

> **&#9432; Note:** Use the private/VPC-internal IP of the Loki host for `alloy_loki_url`
> and `grafana_loki_datasource_url`. Cloud instances cannot route to their own public IP.

### Prometheus / Grafana Inventory

To install and configure Prometheus and Grafana, add `prometheus` and `grafana` groups and hosts to
your inventory (in addition to the other Itential-related groups and hosts). To also install the
Itential Platform Exporter, add an `itential_platform_exporter` group and host.

### Example Inventory

```yaml
all:
  vars:
    platform_release: 6

  children:
    redis:
      hosts:
        <REDIS-HOST-1>:
        <REDIS_HOST-N>:
      vars:
        redis_prometheus_user_enabled: true
        redis_exporter_password: <REDIS-PROMETHEUS-PASSWORD>

    mongodb:
      hosts:
        <MONGODB-HOST-1>:
        <MONGODB-HOST-N>:
      vars:
        mongodb_exporter_admin_password: <MONGODB-ADMIN-PASSWORD>

    platform:
      hosts:
        <PLATFORM-HOST-1>:
        <PLATFORM-HOST-N>:
      vars:
        platform_encryption_key: <openssl rand -hex 32> # 64-length hex string, representing a 256-bit AES  encryption key.
        process_exporter_names: |
          {% raw %}
            - cmdline:
                - Pronghorn
            - cmdline:
                - python3
          {% endraw %}

    gateway:
      hosts:
        <GATEWAY-HOST-1>:
        <GATEWAY-HOST-N>:
      vars:
        process_exporter_names: |
          {% raw %}
            - cmdline:
                - python3.9
          {% endraw %}

    itential_platform_exporter:
      hosts:
        <EXPORTER-HOST>:
      vars:
        itential_platform_exporter_mongo_uri: "mongodb://prometheus:<MONGODB-PROMETHEUS-PASSWORD>@<MONGODB-HOST-1>:27017,<MONGODB-HOST-N>:27017/itential?replicaSet=rs0&authSource=admin"

    prometheus:
      hosts:
        <PROMETHEUS-HOST>:

    grafana:
      hosts:
        <GRAFANA-HOST>:
```

## Running the Playbooks

### Loki / Alloy Playbooks

#### Quick Start

Assumes Grafana and Prometheus are already deployed. The Loki VM can be the same host as
Grafana or a standalone server.

**Step 1 — Add `loki` and `grafana` groups to your inventory**

```yaml
all:
  vars:
    alloy_loki_url: "http://<LOKI-PRIVATE-IP>:3100"

  children:
    loki:
      hosts:
        <LOKI-HOST>:
          ansible_host: <LOKI-IP>

    grafana:
      hosts:
        <GRAFANA-HOST>:
          ansible_host: <GRAFANA-IP>
      vars:
        grafana_loki_datasource_url: "http://<LOKI-PRIVATE-IP>:3100"
```

For a single-VM dev setup, both `loki` and `grafana` can point to the same host and
`grafana_loki_datasource_url` can be `http://localhost:3100`.

**Step 2 — Deploy Loki and wire it into Grafana**

```bash
ansible-playbook itential.monitoring.loki -i <inventory>
```

This installs Loki, starts the service, and provisions the Loki datasource in Grafana in
one run. No separate Grafana playbook step is needed.

**Step 3 — Deploy Alloy to all Itential application nodes**

```bash
ansible-playbook itential.monitoring.alloy -i <inventory>
```

Log paths and OS group memberships for each host group (`platform*`, `mongodb*`,
`redis_master`, `redis_replica`, `redis_sentinel`, `iag5_servers`, `iag5_runners`,
`iag5_clients`) are pre-configured in the collection. No additional inventory variables
are needed beyond `alloy_loki_url`.

After both playbooks complete, open **Grafana → Explore**, select the **Loki** datasource,
and run a query such as `{host="<hostname>"}` to verify logs are flowing.

---

#### Selective Execution

Re-run specific phases without a full reinstall:

```bash
# Redeploy Loki config only (e.g. after changing retention)
ansible-playbook itential.monitoring.loki -i <inventory> --tags loki_configure

# Redeploy Alloy config only (e.g. after adding a new log path)
ansible-playbook itential.monitoring.alloy -i <inventory> --tags alloy_configure
```

| Tag | Description |
| ---- | ----------- |
| `loki_install` | Install the Loki binary, create user/group/directories, open firewall port |
| `loki_configure` | Deploy `loki-config.yml` and the systemd service file |
| `alloy_install` | Install the Alloy package, configure repository, set up user groups, open firewall port |
| `alloy_configure` | Deploy `/etc/alloy/config.alloy` |

### ELK Stack Playbooks

To deploy the full ELK stack (Elasticsearch, Logstash, Kibana, and Filebeat), run the `elk` playbook:

```bash
ansible-playbook itential.monitoring.elk -i <inventory>
```

To deploy individual components:

```bash
ansible-playbook itential.monitoring.elasticsearch -i <inventory>
ansible-playbook itential.monitoring.logstash -i <inventory>
ansible-playbook itential.monitoring.kibana -i <inventory>
ansible-playbook itential.monitoring.filebeat -i <inventory>
```

You can selectively execute portions of each role using the following tags:

| Tag | Description |
| ---- | ----------- |
| `elasticsearch_install` | Install the Elasticsearch package and configure repositories |
| `elasticsearch_configure` | Deploy Elasticsearch configuration files |
| `logstash_install` | Install the Logstash package and configure repositories |
| `logstash_configure` | Deploy Logstash config files and manage keystore secrets |
| `kibana_install` | Install the Kibana package and configure repositories |
| `kibana_configure` | Deploy `kibana.yml` |
| `filebeat_install` | Install the Filebeat package and configure repositories |
| `filebeat_configure` | Deploy `filebeat.yml` and input configurations |

### Prometheus / Grafana Playbooks

To execute the installation of Prometheus, Grafana and all the exporters, run the `prometheus_site` playbook:

```bash
ansible-playbook itential.monitoring.prometheus_site -i <inventory>
```

To install Prometheus only, run the `prometheus` playbook:

```bash
ansible-playbook itential.monitoring.prometheus -i <inventory>
```

To install Grafana only, run the `grafana` playbook:

```bash
ansible-playbook itential.monitoring.grafana -i <inventory>
```

To install the exports, run the `prometheus_exporters` playbook:

```bash
ansible-playbook itential.monitoring.prometheus_exports -i <inventory>
```

You can also selectively execute portions of the role by using the following tags:

| Tag  | Description |
| ---- | ----------- |
| `prometheus_install` | This will execute the tasks to install Prometheus. |
| `itential_scrape_config_install` | This will execute the task to create the Itential scrape config file. |
| `node_exporter_install`  | This will execute the tasks to install the node exporter. The node exporter is installed on all Itential-related hosts and will expose system and sysadmin type metrics. |
| `process_exporter_install` | This will execute the tasks to install the process exporter. The process exporter is installed on `platform` and `gateway` hosts. |
| `mongodb_exporter_install` | This will execute the tasks to install the mongo exporter. The mongo exporter is installed on `mongodb` hosts. |
| `redis_exporter_install` | This will execute the tasks to install the redis exporter. The redis exporter is installed on `redis` hosts. |
| `itential_platform_exporter_install` | This will execute the tasks to install the Itential Platform exporter. It is installed on `itential_platform_exporter` hosts. |
| `grafana_install` | This will execute the tasks to install Grafana. |
