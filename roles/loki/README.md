# itential.monitoring.loki

Installs and configures [Grafana Loki](https://grafana.com/docs/loki/latest/) as the log storage
backend for the Itential monitoring stack. Loki is deployed as a binary (downloaded from GitHub
releases) running under a dedicated systemd service.

## Requirements

- RHEL/Rocky Linux 9 (uses `dnf`)
- `unzip` installable from configured repos
- Outbound internet access to `github.com` for binary download (or pre-stage the binary)
- Firewalld managed automatically if the service is running

## Role Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `loki_version` | String | Loki version to install | `3.7.1` |
| `loki_http_listen_port` | Integer | HTTP API and push endpoint port | `3100` |
| `loki_grpc_listen_port` | Integer | gRPC port | `9096` |
| `loki_data_dir` | String | Loki data directory | `/var/lib/loki` |
| `loki_config_dir` | String | Loki config directory | `/etc/loki` |
| `loki_install_dir` | String | Directory for the Loki binary | `/usr/local/bin` |
| `loki_user` | String | System user Loki runs as | `loki` |
| `loki_group` | String | System group for Loki | `loki` |
| `loki_service_name` | String | Systemd service name | `loki` |
| `loki_reject_old_samples_max_age` | String | Reject logs older than this age | `168h` |
| `loki_max_entries_limit` | Integer | Max entries returned per query | `5000` |
| `loki_ingestion_rate_mb` | Integer | Ingestion rate limit in MB/s | `16` |
| `loki_ingestion_burst_size_mb` | Integer | Ingestion burst size in MB | `32` |

## Tags

| Tag | Description |
|-----|-------------|
| `loki_install` | Install binary, create user/group/dirs, open firewall port |
| `loki_configure` | Deploy `loki-config.yml` and systemd service file |

## Grafana Integration

After deploying Loki, enable the Loki datasource in Grafana by setting the following in your
inventory (on the `grafana` host group vars):

```yaml
loki_datasource_enabled: true
loki_datasource_url: "http://<LOKI-HOST-IP>:3100"
```

Then re-run the Grafana playbook to provision the datasource.

## Inventory

Add a `loki` group to your inventory. For a dev setup, you can point it at the same host as
Grafana:

```yaml
loki:
  hosts:
    <LOKI-HOST>:

grafana:
  hosts:
    <LOKI-HOST>:  # same VM — both roles run against it
```

## Playbook

```bash
ansible-playbook itential.monitoring.loki -i <inventory>
```
