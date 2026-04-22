# alloy role

## Purpose
Installs Grafana Alloy on Itential application nodes and configures it to collect the
systemd journal and application log files, shipping all streams to a central Loki instance.
Alloy replaces Promtail — the role stops and disables Promtail if it is found running.
Installed via the Grafana RPM repository (same repo used by the Grafana role).

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three blocks: `alloy_install`, `alloy_configure`, `always` |
| `templates/config.alloy.j2` | Alloy config — Loki push endpoint, journal source, optional file log source |
| `handlers/main.yml` | Single handler: `Restart Alloy` |

## Task block structure
`tasks/main.yml` is split into three tagged blocks:

- **`alloy_install`** — adds the Grafana RPM repo, installs the `alloy` package, stops
  Promtail (`failed_when: false` so it does not fail if Promtail is absent), adds the
  `alloy` user to `systemd-journal` and any `alloy_extra_groups`, opens the Alloy HTTP
  port in firewalld if the service is running.
- **`alloy_configure`** — templates `config.alloy` to `/etc/alloy/config.alloy` with
  `validate: alloy fmt %s`. The template will fail to deploy if the rendered HCL is
  syntactically invalid. Notifies `Restart Alloy`.
- **`always`** — flushes handlers, starts and enables the service, asserts
  `ActiveState == active`.

## Config architecture
`config.alloy.j2` renders three sections:

1. **`loki.write "default"`** — push endpoint at `{{ alloy_loki_url }}/loki/api/v1/push`.
   This is always rendered; `alloy_loki_url` must be set.
2. **`loki.source.journal "journal"`** — collects the systemd journal with `max_age = 12h`.
   Labels: `job = "journal"`, `host = inventory_hostname`. A `loki.relabel` block extracts
   `__journal__systemd_unit` → `unit` label. Always rendered.
3. **`local.file_match` + `loki.source.file "app_logs"`** — rendered only when
   `alloy_log_paths | length > 0`. Each entry in `alloy_log_paths` becomes a path target
   with its `job` and `host` labels.

## group_vars defaults
Log paths and OS group memberships for all standard Itential host groups are pre-configured
in `playbooks/group_vars/`. The only variable that must be set in inventory is
`alloy_loki_url` under `all.vars`. Individual group defaults can be overridden in the
user's own inventory group_vars — inventory group_vars take precedence over playbook
group_vars.

| group_vars file | `alloy_log_paths` job labels | `alloy_extra_groups` |
|---|---|---|
| `platform.yml` | `iap-http`, `iap` | — |
| `iag5_servers.yml` | `iag5-server` | `[itential]` |
| `iag5_runners.yml` | `iag5-runner` | `[itential]` |
| `iag5_clients.yml` | `iag5-client` | `[itential]` |
| `mongodb.yml` | `mongodb` | `[mongod]` |
| `redis_master.yml`, `redis_replica.yml` | `redis` | — |
| `redis_sentinel.yml` | `redis-sentinel` | — |

## OS group membership
`alloy_extra_groups` adds the `alloy` user to additional OS groups so it can read
log files that are not world-readable (e.g. `mongod.log` is `0640 mongod:mongod`).
If omitted for a group that requires it, Alloy will silently fail to tail those files
without producing a service error.

## Variables that should be overridden
- `alloy_loki_url` — **required**; must be the private/VPC-internal IP of the Loki host.
  EC2 instances cannot route to their own public IP. Defaults to `""` — the role will
  deploy a broken config if this is not set.
