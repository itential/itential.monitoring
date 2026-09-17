# filebeat

Installs and configures Filebeat for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+
- Systemd
- Internet access to the Elastic package repository (or a local mirror)
- A running Logstash or Elasticsearch instance to receive events

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `filebeat_version` | `8.13.0` | Filebeat version to install |
| `filebeat_data_path` | `/var/lib/filebeat` | Data directory |
| `filebeat_inputs_path` | `/etc/filebeat/inputs.d` | Drop-in input configuration directory |
| `filebeat_log_path` | `/var/log/filebeat` | Log directory |
| `filebeat_environment` | `unset` | Environment label attached to every event (e.g. `production`, `staging`) |
| `filebeat_tls_enabled` | `true` | Enable TLS for the Logstash or Elasticsearch output |
| `filebeat_tls_copy_certs` | `true` | Copy certificates from the control node to the target host |
| `filebeat_pki_src_dir` | `""` | Directory on the control node containing the certificate files |
| `filebeat_pki_base_dir` | `/etc/pki/filebeat` | Base PKI directory on the target host |
| `filebeat_tls_cert_file` | `{{ inventory_hostname }}.crt` | Certificate filename |
| `filebeat_tls_key_file` | `{{ inventory_hostname }}.key` | Private key filename |
| `filebeat_tls_ca_file` | `ca.crt` | CA certificate filename |
| `filebeat_output_logstash_enabled` | `true` | Send output to Logstash |
| `filebeat_output_logstash_hosts` | `["localhost:5044"]` | Logstash hosts |
| `filebeat_output_elasticsearch_enabled` | `false` | Send output directly to Elasticsearch |
| `filebeat_output_elasticsearch_hosts` | `["https://localhost:9200"]` | Elasticsearch hosts |
| `filebeat_output_elasticsearch_index` | `filebeat-%{+YYYY.MM.dd}` | Elasticsearch index pattern |
| `filebeat_setup_kibana_host` | `https://localhost:5601` | Kibana host for dashboard setup |
| `filebeat_setup_dashboards_enabled` | `false` | Load Kibana dashboards on first run |

> Only one output (`logstash` or `elasticsearch`) should be enabled at a time.

## TLS

TLS is enabled by default (`filebeat_tls_enabled: true`). When enabled, the `ssl` block is
added to the active output with the CA cert for server verification and a client cert/key pair
for mutual TLS.

Certificates must be in place on the host before the play runs. Set `filebeat_tls_enabled: false`
in the inventory to disable TLS (e.g. for a local development environment).

## Input architecture

Inputs are managed as drop-in files under `filebeat_inputs_path` (`/etc/filebeat/inputs.d/`).
`filebeat.yml` points to that directory with `reload.enabled: true` so new input files are
picked up automatically without a service restart.

The role always deploys `input_default.yml.j2` → `inputs.d/default.yml` (syslog / messages).
Additional app-specific input templates are available and can be deployed separately:

| Template | `app` field | Log paths |
|---|---|---|
| `input_default.yml.j2` | `syslog` | `/var/log/syslog`, `/var/log/messages` |
| `input_itential_platform.yml.j2` | `itential-platform-main`, `itential-platform-web`, `itential-platform-service` | `/var/log/itential/platform/` |
| `input_itential_gateway_client.yml.j2` | `itential-gateway-client` | `/home/itential/.gateway.d/gateway.log` |
| `input_itential_gateway_server.yml.j2` | `itential-gateway-server` | (see template) |
| `input_itential_gateway_runner.yml.j2` | `itential-gateway-runner` | (see template) |
| `input_mongodb.yml.j2` | `mongodb` | `/var/log/mongodb/mongod.log` |
| `input_redis.yml.j2` | `redis` | `/var/log/redis/redis-server.log` |
| `input_redis_sentinel.yml.j2` | `redis-sentinel` | (see template) |

Every input attaches `app` and `environment` fields at the root level
(`fields_under_root: true`) so the Logstash pipeline can route events by `[app]`.

## Example Playbooks

### Ship to Logstash (default)

```yaml
- hosts: all
  roles:
    - role: itential.monitoring.filebeat
      vars:
        filebeat_environment: production
        filebeat_output_logstash_hosts:
          - "logstash-host:5044"
```

### TLS disabled

```yaml
- hosts: all
  roles:
    - role: itential.monitoring.filebeat
      vars:
        filebeat_tls_enabled: false
        filebeat_environment: development
        filebeat_output_logstash_hosts:
          - "logstash-host:5044"
```

### Ship directly to Elasticsearch

```yaml
- hosts: all
  roles:
    - role: itential.monitoring.filebeat
      vars:
        filebeat_environment: staging
        filebeat_output_logstash_enabled: false
        filebeat_output_elasticsearch_enabled: true
        filebeat_output_elasticsearch_hosts:
          - "https://es-node-1:9200"
```

## Tags

| Tag | Description |
|---|---|
| `filebeat_install` | Install the Filebeat package and configure repositories |
| `filebeat_configure` | Deploy `filebeat.yml`, `inputs.d/default.yml`, and optionally load Kibana dashboards |
| `always` | Ensure the service is running |
