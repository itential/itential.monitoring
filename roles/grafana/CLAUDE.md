# grafana role

## Purpose
Installs Grafana via the official RPM repository and provisions a Prometheus datasource and a set of Itential-specific dashboards.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | User-facing variables (port, paths, repo URLs) |
| `vars/main.yml` | Internal variable: `grafana_prometheus_web_listen_port: 9090` |
| `tasks/main.yml` | Creates user/group, installs package, deploys datasource + dashboards, opens firewalld port, starts service |
| `templates/grafana_datasources.yml.j2` | Provisioned Prometheus datasource — reads the first host in the `prometheus` inventory group |
| `templates/grafana_dashboard_config_iap.yml.j2` | Grafana dashboard provider config pointing to the bundled JSON files |
| `files/definitions/default/` | Bundled dashboard JSON files (IAP, MongoDB, Node, Redis) |

## OS support
**RHEL/CentOS only.** The tasks use `yum_repository` and `dnf` without `when: ansible_os_family` guards. Debian/Ubuntu support is not implemented. See the role README for details.

## Datasource provisioning
The datasource template reads `groups.prometheus | first` to determine the Prometheus URL. If `prometheus_web_listen_address` is defined on that host it is used directly; otherwise `inventory_hostname:grafana_prometheus_web_listen_port` is used. The `prometheus` inventory group must be present.

## Firewalld
The role opens `grafana_port/tcp` in the `public` firewalld zone only when the `firewalld.service` is defined, running, and enabled. If firewalld is absent or inactive the task is silently skipped.

## Dashboard JSON files
Dashboard definitions are static JSON files under `files/definitions/default/`. They are copied to `{{ grafana_install_dir }}/provisioning/dashboards/definitions/`. To add or update a dashboard, edit or add a JSON file in that directory — no task changes are needed.

## Variables that must be overridden
None are strictly required. Notable defaults:
- `grafana_port: 3000`
- `grafana_allow_ui_updates: false` — prevents dashboard saves from the UI; set to `true` in dev environments if needed
