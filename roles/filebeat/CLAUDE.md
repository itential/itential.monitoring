# filebeat role

## Purpose
Installs Filebeat from the official Elastic package repository and manages drop-in input configurations for each Itential host type. Outputs to Logstash by default; can be switched to ship directly to Elasticsearch.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Install, configure (deploys `filebeat.yml` + `inputs.d/default.yml`), ensure running |
| `tasks/filebeat_*.yml` | App-specific input deployment tasks — called via `tasks_from` from playbooks |
| `templates/filebeat.yml.j2` | Main Filebeat config — points to `inputs.d/`, configures output and TLS |
| `templates/input_*.yml.j2` | Per-app input templates deployed to `/etc/filebeat/inputs.d/` |
| `handlers/main.yml` | Single handler: Restart Filebeat |

## Input architecture
`filebeat.yml` sets `filebeat.config.inputs.path: /etc/filebeat/inputs.d/*.yml` with `reload.enabled: true`. New input files are picked up without a service restart. `main.yml` always deploys `input_default.yml.j2` → `inputs.d/default.yml` (syslog/messages). All other app inputs are deployed by their `tasks/filebeat_*.yml` task file, called via `tasks_from` from the `filebeat.yml` playbook.

## App inputs and their task files
| Task file | Template deployed | Destination |
|---|---|---|
| `filebeat_itential_platform.yml` | `input_itential_platform.yml.j2` | `inputs.d/itential-platform.yml` |
| `filebeat_itential_gateway_client.yml` | `input_itential_gateway_client.yml.j2` | `inputs.d/itential-gateway-client.yml` |
| `filebeat_itential_gateway_server.yml` | `input_itential_gateway_server.yml.j2` | `inputs.d/itential-gateway-server.yml` |
| `filebeat_itential_gateway_runner.yml` | `input_itential_gateway_runner.yml.j2` | `inputs.d/itential-gateway-runner.yml` |
| `filebeat_mongodb.yml` | `input_mongodb.yml.j2` | `inputs.d/mongodb.yml` |
| `filebeat_redis.yml` | `input_redis.yml.j2` | `inputs.d/redis.yml` |
| `filebeat_redis_sentinel.yml` | `input_redis_sentinel.yml.j2` | `inputs.d/redis-sentinel.yml` |

## File ownership
`filebeat_user` and `filebeat_group` both default to `root`. Input files are deployed with `owner: root`, `group: root`, `mode: 0600`. The `filebeat.yml` is also `0600`.

## Output
Only one output should be enabled at a time. `filebeat_output_logstash_enabled: true` is the default. Setting `filebeat_output_elasticsearch_enabled: true` requires also setting `filebeat_output_logstash_enabled: false`.

## TLS
`filebeat_tls_enabled` adds an `ssl` block to whichever output is active, providing the CA cert, client cert, and client key for mutual TLS. Certs must be pre-placed before the play runs.

## OS support
RHEL/CentOS only. Package install uses `rpm_key` + `yum_repository`.

## Variables that should be overridden
- `filebeat_environment` — defaults to `"unset"`; should be set to `production`, `staging`, etc.
- `filebeat_output_logstash_hosts` — should point to the actual Logstash host, not localhost
