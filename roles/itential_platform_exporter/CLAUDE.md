# itential_platform_exporter role

## Purpose
Installs the [itential-job-metrics-exporter](https://github.com/itential/job-metrics-exporter) —
a Prometheus exporter that connects directly to the Itential Platform MongoDB replica set and
exposes job/task lifecycle metrics (start/complete/error/cancel counters and status gauges).
Downloaded as a prebuilt binary from GitHub releases (not built from source), deployed with a
systemd service. This is unrelated to Itential Platform's own built-in `/prometheus_metrics`
endpoint (the `iap_exporter` scrape job in the `prometheus` role) — it is a separate process
that reads directly from MongoDB.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three blocks: `itential_platform_exporter_install`, `itential_platform_exporter_configure`, `always` |
| `templates/config.yaml.j2` | Exporter config — MongoDB connection, TLS, logging, change stream, polling |
| `templates/itential-job-metrics-exporter.service.j2` | Systemd service unit with hardening options |
| `handlers/main.yml` | Single handler: `Restart Itential Platform Exporter` |

## Task block structure
`tasks/main.yml` is split into three tagged blocks:

- **`itential_platform_exporter_install`** — creates a dedicated system user/group, checks the
  installed binary version via `--version`, downloads the release binary only if the pinned
  version differs, opens the exporter port in firewalld if the service is running.
- **`itential_platform_exporter_configure`** — templates `config.yaml` and the systemd unit to
  disk; both notify the restart handler.
- **`always`** — flushes handlers, starts/enables the service, asserts `ActiveState == active`,
  then polls `{{ itential_platform_exporter_metrics_path }}` (HTTPS when
  `itential_platform_exporter_tls_enabled`) with 12 retries (5s delay).

## Binary install
Unlike `loki` (zip archive), the upstream release assets are raw binaries — one file per arch,
no archive. The role maps `ansible_architecture` (`aarch64` → `arm64`, everything else → `amd64`)
and downloads directly to `{{ itential_platform_exporter_install_dir }}`. The version check
compares `itential_platform_exporter_version` against the installed `--version` output string;
the download/install task is skipped if it already matches.

## MongoDB prerequisites (not automated by this role)
The exporter requires a MongoDB **replica set** and a dedicated read-only user, plus two indexes
on the `itential` database. These must be created manually — this role only installs and
configures the exporter process itself:

```javascript
use admin
db.createUser({
  user: "prometheus",
  pwd: "<password>",
  roles: [{ role: "read", db: "itential" }]
})
```

```javascript
db.jobs.createIndex({ status: 1, _id: 1 }, { name: "itential_status", background: true })
db.tasks.createIndex(
  { status: 1, "metrics.server_id": 1 },
  { name: "itential_job_metrics_exporter_task_status_server", background: true }
)
```

The `jobs` index may already exist — it is typically created by the Itential Platform
application itself.

## Variables that should be overridden
- `itential_platform_exporter_version` — pin to a specific release; defaults to `1.0.1`
- `itential_platform_exporter_mongo_uri` — full connection string (recommended for replica
  sets); when unset, the individual `itential_platform_exporter_mongo_host`/`_port`/`_username`/
  `_password`/`_database`/`_auth_source` fields are used instead
- `itential_platform_exporter_mongo_password` — **must be set via Ansible Vault**; empty by default
- `itential_platform_exporter_change_stream_enabled` — requires MongoDB running as a replica
  set; set `itential_platform_exporter_polling_enabled: true` instead (and disable change
  stream) if it is not

## Not wired into the Prometheus scrape config
This role does not add a scrape job to the `prometheus` role's `scrape_configs.j2`. If the
Itential Platform Exporter should be scraped, that template needs a new target list (e.g. hosts
in a dedicated group, or the `platform` group) — this is a separate change to the `prometheus`
role and was intentionally left out of scope here.
