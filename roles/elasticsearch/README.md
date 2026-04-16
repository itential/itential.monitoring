# elasticsearch

Installs and configures Elasticsearch for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+ or Debian/Ubuntu
- Systemd
- Internet access to the Elastic package repository (or a local mirror)

## Role Variables

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
| `elasticsearch_tls_enabled` | `true` | Enable xpack security and TLS for HTTP and transport |
| `elasticsearch_tls_cert` | `/etc/elasticsearch/certs/elasticsearch.crt` | Path to the node certificate (PEM) |
| `elasticsearch_tls_key` | `/etc/elasticsearch/certs/elasticsearch.key` | Path to the node private key (PEM) |
| `elasticsearch_tls_ca_cert` | `/etc/elasticsearch/certs/ca.crt` | Path to the CA certificate used to verify peer nodes |

## TLS

TLS is enabled by default (`elasticsearch_tls_enabled: true`). When enabled, xpack security,
transport SSL, and HTTP SSL are all turned on together — Elasticsearch 8.x requires all three
to be consistent.

Certificates must be in place on the host before the play runs. Set `elasticsearch_tls_enabled: false`
in the inventory to disable TLS (e.g. for a local development environment).

## Example Playbook

### Single-node

```yaml
- hosts: elasticsearch
  roles:
    - role: itential.monitoring.elasticsearch
      vars:
        elasticsearch_heap_size: "2g"
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
        elasticsearch_heap_size: "4g"
```

## Tags

| Tag | Description |
|---|---|
| `elasticsearch_install` | Install the Elasticsearch package and configure repositories |
| `elasticsearch_configure` | Deploy configuration files |
| `always` | Ensure the service is running |
