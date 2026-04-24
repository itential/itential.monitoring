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
| `loki_tls_enabled` | Boolean | Enable TLS on the HTTP listener | `false` |
| `loki_tls_cert_file` | String | Path to the server certificate file | `/etc/loki/certs/loki.crt` |
| `loki_tls_key_file` | String | Path to the server private key file | `/etc/loki/certs/loki.key` |

## Tags

| Tag | Description |
|-----|-------------|
| `loki_install` | Install binary, create user/group/dirs, open firewall port |
| `loki_configure` | Deploy `loki-config.yml` and systemd service file |

## TLS

TLS is disabled by default. To enable server-side TLS on the Loki HTTP listener, set the
following in your inventory for the `loki` group. Certificates must be pre-placed on the
host before running the playbook — this role does not deploy them.

```yaml
loki:
  vars:
    loki_tls_enabled: true
    loki_tls_cert_file: /etc/loki/certs/loki.crt
    loki_tls_key_file: /etc/loki/certs/loki.key
```

When TLS is enabled, all clients connecting to port `{{ loki_http_listen_port }}` must use
HTTPS. Update `alloy_loki_url` to `https://<LOKI-HOST-IP>:3100` and `grafana_loki_datasource_url`
to `https://<LOKI-HOST-IP>:3100` in your inventory accordingly.

## Grafana Integration

After deploying Loki, enable the Loki datasource in Grafana by setting the following in your
inventory (on the `grafana` host group vars):

```yaml
grafana_loki_datasource_enabled: true
grafana_loki_datasource_url: "http://<LOKI-HOST-IP>:3100"
```

Then re-run the Loki playbook — the second play provisions the datasource in Grafana
automatically. Use `https://` if TLS is enabled on Loki.

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
