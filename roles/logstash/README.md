# logstash

Installs and configures Logstash for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+ or Debian/Ubuntu
- Systemd
- Internet access to the Elastic package repository (or a local mirror)
- Java is bundled with the Logstash package — no separate JDK required
- A running Elasticsearch instance with a `logstash_writer` user configured

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
| `logstash_keystore_password` | `""` | Password for the Logstash keystore — **must be overridden** (use Ansible Vault) |
| `logstash_elastic_password` | `""` | Password for the `logstash_writer` Elasticsearch user — **must be overridden** (use Ansible Vault) |
| `logstash_pipeline_workers` | `{{ ansible_processor_vcpus }}` | Number of pipeline worker threads |
| `logstash_pipeline_batch_size` | `125` | Events per worker per batch |
| `logstash_pipeline_batch_delay` | `50` | Batch delay in milliseconds |

## TLS

TLS is enabled by default (`logstash_tls_enabled: true`). When enabled:
- The Beats input requires TLS from all Filebeat clients
- The Elasticsearch output includes `ssl_certificate_authorities` for server verification

Certificates must be in place on the host before the play runs. Set `logstash_tls_enabled: false`
in the inventory to disable TLS and also update `logstash_elasticsearch_hosts` to use `http://`.

## Secrets / Keystore

The role manages the Logstash keystore automatically:

1. Creates `/etc/logstash/logstash.keystore` if it does not exist, protected by `logstash_keystore_password`.
2. Stores `logstash_elastic_password` in the keystore under the key `ELASTIC_PASSWORD`.
3. Writes `LOGSTASH_KEYSTORE_PASS` to the service environment file (`/etc/sysconfig/logstash` on RHEL, `/etc/default/logstash` on Debian) so Logstash can decrypt the keystore at startup.

The pipeline template references `${ELASTIC_PASSWORD}` which Logstash resolves from the keystore at runtime.

Both `logstash_keystore_password` and `logstash_elastic_password` must be set. Use Ansible Vault to encrypt them:

```yaml
logstash_keystore_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...

logstash_elastic_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

## Pipeline

The pipeline at `/etc/logstash/conf.d/itential.conf` routes and enriches events by the
`app` field set by Filebeat inputs:

| `app` value | Enrichment |
|---|---|
| `itential-platform-service` | Extracts sub-process name from the log file path into `itential.process` |
| `itential-platform-main` | Sets `itential.process = platform` |
| `itential-platform-web` | Sets `itential.process = webserver`; parses HTTP access log fields |
| `itential-platform-service` / `WorkFlowEngine` process | Renames job/task fields; tags `automation_failure` on error/failed status |
| `itential-platform-service` / `GatewayManager` process | Extracts gateway name and BullMQ queue; tags `gateway_connectivity_failure` on error |
| `itential-gateway-server` | Parses gRPC method, job IDs, command exit status; tags `gateway_connectivity_failure` on failure |
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
        logstash_keystore_password: "{{ vault_logstash_keystore_password }}"
        logstash_elastic_password: "{{ vault_logstash_elastic_password }}"
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
        logstash_keystore_password: "{{ vault_logstash_keystore_password }}"
        logstash_elastic_password: "{{ vault_logstash_elastic_password }}"
```

## Tags

| Tag | Description |
|---|---|
| `logstash_install` | Install the Logstash package and configure repositories |
| `logstash_configure` | Deploy `logstash.yml`, JVM options, pipeline config, and keystore secrets |
| `always` | Ensure the service is running |
