# grafana

Installs and configures Grafana with a Prometheus datasource and Itential-specific dashboards.

> **Note:** This role currently supports **RHEL/CentOS only**. The installation tasks use `yum_repository` and `dnf` without OS family conditionals. Debian/Ubuntu support will be added in a future release.

## Requirements

- RHEL/CentOS 8+
- Systemd
- Internet access to the Grafana RPM repository (or a local mirror)
- A running Prometheus instance in the `prometheus` inventory group

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `grafana_user` | `grafana` | Linux user for the Grafana service |
| `grafana_group` | `grafana` | Linux group for the Grafana service |
| `grafana_port` | `3000` | Port Grafana listens on |
| `grafana_repo_url` | `https://rpm.grafana.com` | Grafana RPM repository URL |
| `grafana_gpg_key` | `https://rpm.grafana.com/gpg.key` | Grafana GPG key URL |
| `grafana_install_dir` | `/etc/grafana` | Root Grafana configuration directory |
| `grafana_dashboard_dir` | `/etc/grafana/provisioning/dashboards` | Directory where dashboard definitions are uploaded |
| `grafana_allow_ui_updates` | `false` | Allow saving dashboards from the Grafana UI |

## Dashboards

The role bundles the following dashboard JSON files and provisions them automatically:

| File | Description |
|---|---|
| `grafana_dashboard_definition_iap.json` | IAP platform metrics |
| `grafana_dashboard_definition_iap2.json` | Additional IAP metrics |
| `grafana_dashboard_definition_mongo.json` | MongoDB metrics |
| `grafana_dashboard_definition_node.json` | Node exporter (system) metrics |
| `grafana_dashboard_definition_redis.json` | Redis metrics |

## Datasource

A Prometheus datasource named `Prometheus IAP` is provisioned automatically. The Prometheus URL is derived from the first host in the `prometheus` inventory group. If `prometheus_web_listen_address` is defined on that host it is used directly; otherwise the URL is built from `inventory_hostname:9090`.

## Firewalld

If `firewalld` is active on the host, the role opens `grafana_port/tcp` in the `public` zone permanently. If firewalld is absent or inactive the task is silently skipped.

## Example Playbook

```yaml
- hosts: grafana
  roles:
    - role: itential.monitoring.grafana
```

## Tags

| Tag | Description |
|---|---|
| `always` | Assert that the Grafana service is running |
