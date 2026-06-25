# elasticsearch

Installs and configures Elasticsearch for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+ or Debian/Ubuntu
- Systemd
- Internet access to the Elastic package repository (or a local mirror)

## Role Variables

### General

| Variable | Default | Description |
|---|---|---|
| `elasticsearch_version` | `8.13.0` | Elasticsearch version to install |
| `elasticsearch_http_port` | `9200` | HTTP API port |
| `elasticsearch_transport_port` | `9300` | Inter-node transport port |
| `elasticsearch_network_host` | `0.0.0.0` | Network bind address |
| `elasticsearch_cluster_name` | `itential-monitoring` | Cluster name |
| `elasticsearch_node_name` | `{{ inventory_hostname }}` | Node name |
| `elasticsearch_discovery_seed_hosts` | `[]` | Seed hosts for cluster discovery (single-node mode when empty) |
| `elasticsearch_cluster_initial_master_nodes` | `[]` | Initial master nodes for cluster bootstrap |
| `elasticsearch_data_path` | `/var/lib/elasticsearch` | Data directory |
| `elasticsearch_log_path` | `/var/log/elasticsearch` | Log directory |
| `elasticsearch_heap_size` | `1g` | JVM heap size (applied to both Xms and Xmx) |

### Passwords

These have empty string defaults and must be set in inventory using Ansible Vault.

| Variable | Description |
|---|---|
| `elasticsearch_elastic_password` | Password for the built-in `elastic` superuser |
| `elasticsearch_logstash_writer_password` | Password for the `logstash_writer` user created by this role |
| `elasticsearch_grafana_reader_password` | Password for the `grafana_reader` user created by this role |

### TLS

| Variable | Default | Description |
|---|---|---|
| `elasticsearch_tls_enabled` | `true` | Enable xpack security and TLS for HTTP and transport |
| `elasticsearch_tls_copy_certs` | `true` | Copy certificates from the control node to the target host |
| `elasticsearch_pki_src_dir` | `""` | Directory on the control node containing the certificate files |
| `elasticsearch_pki_base_dir` | `/etc/pki/elasticsearch` | Base PKI directory on the target host |
| `elasticsearch_tls_cert_file` | `{{ inventory_hostname }}.crt` | Certificate filename |
| `elasticsearch_tls_key_file` | `{{ inventory_hostname }}.key` | Private key filename |
| `elasticsearch_tls_ca_file` | `ca.crt` | CA certificate filename |

### ILM

ILM variables have no defaults and must be defined in inventory. When set, the
`elasticsearch_users` task applies the `itential-logs-policy` ILM policy to
Elasticsearch via the REST API.

| Variable | Example | Description |
|---|---|---|
| `elasticsearch_ilm_rollover_size` | `10gb` | Roll over the hot index when it reaches this size |
| `elasticsearch_ilm_rollover_age` | `1d` | Roll over the hot index after this age |
| `elasticsearch_ilm_warm_age` | `7d` | Move to warm phase after this age |
| `elasticsearch_ilm_cold_age` | `30d` | Move to cold phase (frozen) after this age |
| `elasticsearch_ilm_delete_age` | `90d` | Delete the index after this age |

## TLS

TLS is enabled by default (`elasticsearch_tls_enabled: true`). When enabled, xpack security,
transport SSL, and HTTP SSL are all turned on together — Elasticsearch 8.x requires all three
to be consistent.

Certificates must be in place on the host before the play runs. Set `elasticsearch_tls_enabled: false`
in the inventory to disable TLS (e.g. for a local development environment).

## Users and Roles

The `elasticsearch_users` task creates the following after the service is confirmed active:

| Name | Type | Description |
|---|---|---|
| `logstash_writer` | Role | Write access to `itential-logs-*` and `itential-failures-*` indices |
| `logstash_writer` | User | Service account used by Logstash; password set from `elasticsearch_logstash_writer_password` |
| `grafana_reader` | Role | Read-only access to `itential-logs-*` and `itential-failures-*` indices |
| `grafana_reader` | User | Service account used by Grafana; password set from `elasticsearch_grafana_reader_password` |

The `itential-logs-policy` ILM policy is applied separately via the `elasticsearch_apply_ilm_policy` tag.

## Example Playbook

### Single-node

```yaml
- hosts: elasticsearch
  roles:
    - role: itential.monitoring.elasticsearch
      vars:
        elasticsearch_heap_size: "2g"
        elasticsearch_elastic_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        elasticsearch_logstash_writer_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        elasticsearch_ilm_rollover_size: "10gb"
        elasticsearch_ilm_rollover_age: "1d"
        elasticsearch_ilm_warm_age: "7d"
        elasticsearch_ilm_cold_age: "30d"
        elasticsearch_ilm_delete_age: "90d"
```

### Single-node, TLS disabled

```yaml
- hosts: elasticsearch
  roles:
    - role: itential.monitoring.elasticsearch
      vars:
        elasticsearch_tls_enabled: false
        elasticsearch_heap_size: "2g"
```

### Multi-node cluster

```yaml
- hosts: elasticsearch
  roles:
    - role: itential.monitoring.elasticsearch
      vars:
        elasticsearch_cluster_name: itential-monitoring
        elasticsearch_discovery_seed_hosts:
          - es-node-1
          - es-node-2
          - es-node-3
        elasticsearch_cluster_initial_master_nodes:
          - es-node-1
          - es-node-2
          - es-node-3
        elasticsearch_heap_size: "16g"
        elasticsearch_elastic_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        elasticsearch_logstash_writer_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        elasticsearch_ilm_rollover_size: "10gb"
        elasticsearch_ilm_rollover_age: "1d"
        elasticsearch_ilm_warm_age: "7d"
        elasticsearch_ilm_cold_age: "30d"
        elasticsearch_ilm_delete_age: "90d"
```

## Tags

| Tag | Description |
|---|---|
| `elasticsearch_install` | Install the Elasticsearch package and configure repositories |
| `elasticsearch_configure` | Deploy configuration files and manage the keystore |
| `elasticsearch_certificates` | Copy TLS certificates to the target host |
| `elasticsearch_users` | Create roles and users via the Elasticsearch API |
| `elasticsearch_apply_ilm_policy` | Apply the `itential-logs-policy` ILM policy via the Elasticsearch API |
| `always` | Ensure the service is enabled and running |
