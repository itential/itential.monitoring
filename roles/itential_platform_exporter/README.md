# itential.monitoring.itential_platform_exporter

Installs and configures the [itential-job-metrics-exporter](https://github.com/itential/job-metrics-exporter),
a Prometheus exporter that connects directly to the Itential Platform MongoDB replica set and
exposes job/task lifecycle metrics. Deployed as a binary (downloaded from GitHub releases)
running under a dedicated systemd service.

## Requirements

- RHEL/Rocky Linux 8/9, Amazon Linux 2023, or Oracle Linux 8/9 (amd64 or arm64)
- MongoDB 4.2+ running as a **replica set** (required for change streams)
- A dedicated MongoDB read-only user on the `itential` database, plus two indexes — see
  [MongoDB Setup](#mongodb-setup) below (not automated by this role)
- Outbound internet access to `github.com` for binary download (or pre-stage the binary)
- Firewalld managed automatically if the service is running

## MongoDB Setup

Create a read-only user:

```javascript
use admin
db.createUser({
  user: "prometheus",
  pwd: "<password>",
  roles: [{ role: "read", db: "itential" }]
})
```

Create the required indexes on the `itential` database:

```javascript
db.jobs.createIndex({ status: 1, _id: 1 }, { name: "itential_status", background: true })
db.tasks.createIndex(
  { status: 1, "metrics.server_id": 1 },
  { name: "itential_job_metrics_exporter_task_status_server", background: true }
)
```

The `jobs` index (`itential_status`) may already exist — it is typically created by the
Itential Platform application itself.

## Role Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `itential_platform_exporter_version` | String | Exporter version to install | `1.0.1` |
| `itential_platform_exporter_listen_address` | String | Address/port the exporter listens on | `:9477` |
| `itential_platform_exporter_metrics_path` | String | Metrics endpoint path | `/metrics` |
| `itential_platform_exporter_config_dir` | String | Config directory | `/etc/itential-job-metrics-exporter` |
| `itential_platform_exporter_install_dir` | String | Directory for the binary | `/usr/local/bin` |
| `itential_platform_exporter_user` | String | System user the exporter runs as | `itential_platform_exporter` |
| `itential_platform_exporter_group` | String | System group | `itential_platform_exporter` |
| `itential_platform_exporter_service_name` | String | Systemd service name | `itential-job-metrics-exporter` |
| `itential_platform_exporter_mongo_uri` | String | Full MongoDB connection string (takes precedence over the individual fields below) | `""` |
| `itential_platform_exporter_mongo_host` | String | MongoDB host (used when `_mongo_uri` is unset) | `localhost` |
| `itential_platform_exporter_mongo_port` | Integer | MongoDB port | `27017` |
| `itential_platform_exporter_mongo_username` | String | MongoDB username | `prometheus` |
| `itential_platform_exporter_mongo_password` | String | MongoDB password — **must be set via Ansible Vault** | `""` |
| `itential_platform_exporter_mongo_database` | String | MongoDB database name | `itential` |
| `itential_platform_exporter_mongo_auth_source` | String | MongoDB auth source database | `admin` |
| `itential_platform_exporter_mongo_tls_enabled` | Boolean | Enable TLS for the MongoDB connection | `false` |
| `itential_platform_exporter_mongo_tls_ca_file` | String | Path to the CA file used to verify MongoDB's cert | `/etc/itential-job-metrics-exporter/ca.pem` |
| `itential_platform_exporter_tls_enabled` | Boolean | Enable TLS on the exporter's HTTP listener | `false` |
| `itential_platform_exporter_tls_cert_file` | String | Path to the exporter server certificate | `/etc/itential-job-metrics-exporter/server.crt` |
| `itential_platform_exporter_tls_key_file` | String | Path to the exporter server private key | `/etc/itential-job-metrics-exporter/server.key` |
| `itential_platform_exporter_log_level` | String | Log level (`debug`\|`info`\|`warn`\|`error`) | `info` |
| `itential_platform_exporter_log_format` | String | Log format (`json`\|`text`) | `json` |
| `itential_platform_exporter_change_stream_enabled` | Boolean | Enable real-time MongoDB change stream counters (requires a replica set) | `true` |
| `itential_platform_exporter_polling_enabled` | Boolean | Enable background aggregation polling (use when change streams are unavailable) | `false` |
| `itential_platform_exporter_polling_interval` | String | Polling interval | `60s` |

## Tags

| Tag | Description |
|-----|-------------|
| `itential_platform_exporter_install` | Install binary, create user/group/config dir, open firewall port |
| `itential_platform_exporter_configure` | Deploy `config.yaml` and systemd service file |

## TLS

Both the MongoDB connection and the exporter's own HTTP listener support TLS independently.
Certificates must be pre-placed on the host before running the playbook — this role does not
deploy them.

```yaml
itential_platform_exporter:
  vars:
    itential_platform_exporter_tls_enabled: true
    itential_platform_exporter_tls_cert_file: /etc/itential-job-metrics-exporter/server.crt
    itential_platform_exporter_tls_key_file: /etc/itential-job-metrics-exporter/server.key
```

## Inventory

Add an `itential_platform_exporter` group to your inventory, and set the MongoDB password via
Ansible Vault:

```yaml
itential_platform_exporter:
  hosts:
    <EXPORTER-HOST>:
  vars:
    itential_platform_exporter_mongo_uri: "mongodb://prometheus:{{ vault_itential_platform_exporter_mongo_password }}@<mongo-host-1>:27017,<mongo-host-2>:27017,<mongo-host-3>:27017/itential?replicaSet=rs0&authSource=admin"
```

## Playbook

```bash
ansible-playbook itential.monitoring.itential_platform_exporter -i <inventory>
```

## Not wired into the Prometheus scrape config

This role does not add a scrape job for the exporter to the `prometheus` role's
`scrape_configs.j2`. If you want Prometheus to scrape it automatically, that template needs a
new target list — this is a separate change outside the scope of this role.
