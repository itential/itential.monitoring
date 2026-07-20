# prometheus role

## Purpose
Thin wrapper around the community `prometheus.prometheus.prometheus` role that adds an Itential-specific scrape config file dynamically built from the inventory.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | Default exporter listen ports used when per-host overrides are absent |
| `tasks/main.yml` | Imports `prometheus.prometheus.prometheus`, deploys scrape config, ensures service running |
| `templates/scrape_configs.j2` | Builds `scrape_configs` YAML from inventory groups |
| `handlers/main.yml` | Single handler: Restart Prometheus |

## Scrape config generation
`scrape_configs.j2` reads these inventory groups to build static targets:
| Group | Jobs generated |
|---|---|
| `platform` | `node_exporter`, `process_exporter`, `iap_exporter` |
| `gateway` | `node_exporter`, `process_exporter` |
| `mongodb` | `node_exporter`, `mongo_exporter` |
| `redis_*` (all groups matching `^redis_.*`) | `node_exporter`, `redis_exporter` |
| `itential_platform_exporter` | `itential_platform_exporter` |
| `vault` | `node_exporter` |

The scrape config file is written to `{{ prometheus_config_dir }}/scrape_configs/itential.yml`. `prometheus_config_dir` is provided by the upstream `prometheus.prometheus.prometheus` role.

## Per-host port overrides
Each exporter target uses `inventory_hostname:default_port` unless the host defines a matching hostvars override:
- `node_exporter_web_listen_address`
- `process_exporter_web_listen_address`
- `redis_exporter_web_listen_address`
- `mongodb_exporter_web_listen_address`
- `itential_platform_exporter_web_listen_address` — note this is a scrape-target-only override,
  distinct from the `itential_platform_exporter` role's own `itential_platform_exporter_listen_address`
  variable (which configures the exporter's actual bind address, e.g. `:9477`)

When the override is set, its full value (host:port) is used as-is.

## Dependency
Requires `prometheus.prometheus` collection >= 0.22.0. Must be installed manually:
```bash
ansible-galaxy collection install prometheus.prometheus
```
This collection is not installed automatically as a dependency of `itential.monitoring`.

## Variables
All Prometheus server variables (`prometheus_config_dir`, `prometheus_system_user`, etc.) are owned by the upstream `prometheus.prometheus.prometheus` role. The only variables defined in this role are the default exporter ports in `defaults/main.yml`.
