# logstash

Installs and configures Logstash for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+ or Debian/Ubuntu
- Systemd
- Internet access to the Elastic package repository (or a local mirror)
- Java is bundled with the Logstash package — no separate JDK required

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `logstash_version` | `8.13.0` | Logstash version to install |
| `logstash_heap_size` | `1g` | JVM heap size (applied to both Xms and Xmx) |
| `logstash_data_path` | `/var/lib/logstash` | Data directory |
| `logstash_log_path` | `/var/log/logstash` | Log directory |
| `logstash_beats_port` | `5044` | Port for the Beats input |
| `logstash_tls_enabled` | `true` | Enable TLS for the Beats input and the Elasticsearch output |
| `logstash_tls_cert` | `/etc/logstash/certs/logstash.crt` | Path to the server certificate presented to Beats (PEM) |
| `logstash_tls_key` | `/etc/logstash/certs/logstash.key` | Path to the server private key (PEM) |
| `logstash_tls_ca_cert` | `/etc/logstash/certs/ca.crt` | Path to the CA certificate used to verify Elasticsearch |
| `logstash_elasticsearch_hosts` | `["https://localhost:9200"]` | Elasticsearch output hosts |
| `logstash_pipeline_workers` | `{{ ansible_processor_vcpus }}` | Number of pipeline worker threads |
| `logstash_pipeline_batch_size` | `125` | Events per worker per batch |
| `logstash_pipeline_batch_delay` | `50` | Batch delay in milliseconds |

## TLS

TLS is enabled by default (`logstash_tls_enabled: true`). When enabled:
- The Beats input requires TLS from all Filebeat clients
- The Elasticsearch output includes `ssl_certificate_authorities` for server verification

Certificates must be in place on the host before the play runs. Set `logstash_tls_enabled: false`
in the inventory to disable TLS and also update `logstash_elasticsearch_hosts` to use `http://`.

The Elasticsearch output authenticates with the `logstash_writer` user. The password is read
from the `ELASTIC_PASSWORD` environment variable — set this in the Logstash systemd
environment or via the Logstash keystore.

## Pipeline

The pipeline at `/etc/logstash/conf.d/itential.conf` routes and enriches events by the
`app` field set by Filebeat inputs:

| `app` value | Enrichment |
|---|---|
| `itential-platform-service` | Extracts sub-process name from the log file path into `itential.process` |
| `itential-platform-main` | Sets `itential.process = platform` |
| `itential-platform-web` | Sets `itential.process = webserver` |
| `itential-platform-service` / `WorkFlowEngine` process | Renames job/task fields; tags `automation_failure` on error/failed status |
| `itential-platform-service` / `GatewayManager` process | Renames gateway fields; tags `gateway_connectivity_failure` on error/unreachable |
| `itential-gateway-5` | Renames device/operation fields; tags `automation_failure` and `gateway_failure` on error |
| `mongodb` | Grok-parses log lines; tags `slow_query` when a duration suffix is detected |
| `syslog` | Grok-parses syslog format; normalizes `@timestamp` |

**Output routing:**

- Events tagged `automation_failure` are written to `itential-failures-<date>` in addition to the main index.
- All events are written to `itential-logs-<app>-<date>`.

## Example Playbook

```yaml
- hosts: logstash
  roles:
    - role: itential.monitoring.logstash
      vars:
        logstash_heap_size: "2g"
        logstash_elasticsearch_hosts:
          - "https://es-node-1:9200"
          - "https://es-node-2:9200"
```

### TLS disabled

```yaml
- hosts: logstash
  roles:
    - role: itential.monitoring.logstash
      vars:
        logstash_tls_enabled: false
        logstash_elasticsearch_hosts:
          - "http://localhost:9200"
```

## Tags

| Tag | Description |
|---|---|
| `logstash_install` | Install the Logstash package and configure repositories |
| `logstash_configure` | Deploy `logstash.yml`, JVM options, and pipeline config |
| `always` | Ensure the service is running |
