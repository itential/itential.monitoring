# loki role

## Purpose
Installs Grafana Loki as a single-node log storage backend. Loki is downloaded as a binary
from GitHub releases (not via package manager), deployed with a systemd service, and uses
filesystem storage with a TSDB index. Suitable for co-location with Grafana or as a
standalone server.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three blocks: `loki_install`, `loki_configure`, `always` |
| `templates/loki-config.yml.j2` | Loki server config — storage paths, listen ports, limits, retention |
| `templates/loki.service.j2` | Systemd service unit with hardening options |
| `handlers/main.yml` | Two handlers: `Reload systemd`, `Restart Loki` |

## Task block structure
`tasks/main.yml` is split into three tagged blocks:

- **`loki_install`** — creates system user/group, checks installed version, downloads and
  extracts the binary from GitHub only if the version has changed, opens the HTTP port in
  firewalld if the service is running.
- **`loki_configure`** — templates `loki-config.yml` and `loki.service` to disk; both notify
  handlers. The service file notifies both `Reload systemd` and `Restart Loki` so daemon-reload
  runs before the restart.
- **`always`** — flushes handlers, ensures the service is started and enabled, asserts
  `ActiveState == active`, then polls `localhost:{{ loki_http_listen_port }}/ready` (HTTPS when `loki_tls_enabled`)
  with 12 retries (5s delay) before declaring success.

## Binary install
The role checks `loki --version` output before downloading. If `loki_version` is already
in the version string, the download and extract tasks are skipped. The binary is downloaded
as a zip, extracted to `/tmp/`, installed to `{{ loki_install_dir }}/loki`, then the zip
is removed. The binary runs as root but the process drops to `loki_user` via the systemd
service `User=` directive.

## Storage
Single-node filesystem storage only. `auth_enabled: false`. Data directory layout:

| Path | Contents |
|---|---|
| `{{ loki_data_dir }}/chunks` | Compressed log chunks |
| `{{ loki_data_dir }}/rules` | Alert rules directory |

Schema is TSDB v13, 24h index period. Changing `loki_data_dir` after initial deployment
requires manually moving existing data.

## Variables that should be overridden
- `loki_version` — pin to a specific release; defaults to `3.7.1`
- `loki_reject_old_samples_max_age` — defaults to `168h` (7 days); set to match your
  retention policy (e.g. `720h` for 30 days)
- `loki_ingestion_rate_mb` / `loki_ingestion_burst_size_mb` — increase for high-volume
  deployments with many Alloy agents
