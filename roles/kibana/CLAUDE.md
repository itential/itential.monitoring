# kibana role

## Purpose
Installs Kibana from the official Elastic package repository and deploys configuration to connect it to Elasticsearch.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three tagged blocks: install, configure, ensure running |
| `templates/kibana.yml.j2` | Main Kibana config — server, TLS, Elasticsearch connection, logging |
| `handlers/main.yml` | Single handler: Restart Kibana |

## TLS
`kibana_tls_enabled` controls two things together:
- Kibana server HTTPS (`server.ssl.enabled`)
- Elasticsearch connection CA verification (`elasticsearch.ssl.certificateAuthorities`)

Disabling TLS requires updating `kibana_elasticsearch_hosts` to use `http://` — the role does not do this automatically.

Certificates must be pre-placed on the host before the play runs. The role does not distribute certs.

## Logging
`kibana.yml.j2` configures structured JSON logging to `{{ kibana_log_path }}/kibana.log` in addition to the default console appender. This is hardcoded in the template and not variable-controlled.

## Reverse proxy
Set `kibana_server_base_path` to a non-empty string to enable the `server.basePath` / `server.rewriteBasePath` block. When empty (default), neither setting is written to `kibana.yml`.

## OS support
Supports both RHEL/CentOS (`rpm_key` + `yum_repository`) and Debian/Ubuntu (`apt_key` + `apt_repository`). All package tasks are guarded with `when: ansible_os_family == "RedHat/Debian"`.

## Variables that must be overridden
- `kibana_elasticsearch_hosts` — should point to the actual Elasticsearch host(s), not localhost
