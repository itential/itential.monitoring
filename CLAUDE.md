# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Collection Identity

Ansible collection `itential.monitoring` (v1.1.0, namespace: `itential`). Requires Ansible >= 2.9.10.

## Commands

### Run playbooks

```bash
# Full stacks
ansible-playbook itential.monitoring.elk -i <inventory>
ansible-playbook itential.monitoring.site -i <inventory>        # prometheus + grafana + exporters

# Individual components
ansible-playbook itential.monitoring.loki -i <inventory>
ansible-playbook itential.monitoring.alloy -i <inventory>
ansible-playbook itential.monitoring.prometheus -i <inventory>
ansible-playbook itential.monitoring.grafana -i <inventory>
ansible-playbook itential.monitoring.prometheus_exporters -i <inventory>
ansible-playbook itential.monitoring.elasticsearch -i <inventory>
ansible-playbook itential.monitoring.logstash -i <inventory>
ansible-playbook itential.monitoring.kibana -i <inventory>
ansible-playbook itential.monitoring.filebeat -i <inventory>
```

### Selective execution (tags)

Each role tags its blocks as `<role>_install` and `<role>_configure`. The `always` block (service start + health assert) always runs.

```bash
ansible-playbook itential.monitoring.loki -i <inventory> --tags loki_configure
ansible-playbook itential.monitoring.alloy -i <inventory> --tags alloy_configure
ansible-playbook itential.monitoring.elasticsearch -i <inventory> --tags elasticsearch_configure
```

### Lint

```bash
ansible-lint
```

Config in `.ansible-lint`. The following are warnings (not errors): `yaml[line-length]`, `var-naming[no-role-prefix]`, `meta-runtime[unsupported-version]`, `run-once[task]`.

## Prerequisites

**Prometheus stack** — `prometheus.prometheus` is not installed as a collection dependency and must be added manually:

```bash
ansible-galaxy collection install prometheus.prometheus  # requires >= 0.22.0
```

**macOS** — GNU tar is required for Prometheus-related playbooks:

```bash
brew install gnu-tar
export PATH="/opt/homebrew/opt/gnu-tar/libexec/gnubin:$PATH"
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES   # prevents fork() crash
```

## Architecture

Three independent monitoring stacks — all optional, deployable together or separately.

### Role structure pattern

Every role uses the same three-block pattern in `tasks/main.yml`:

1. **`<role>_install`** — package/binary install, user/group creation, firewalld rules
2. **`<role>_configure`** — template deployment; notifies the restart handler
3. **`always`** — flush handlers → start/enable service → assert `ActiveState == active`

Each role has a single restart handler. Variable defaults live in `defaults/main.yml`; the only exception is `roles/grafana/vars/main.yml`, which holds the internal Prometheus port constant.

---

### Prometheus / Grafana (metrics)

`itential.monitoring.prometheus` is a thin wrapper around the community `prometheus.prometheus.prometheus` role. Its only additions:
- Generates `{{ prometheus_config_dir }}/scrape_configs/itential.yml` from inventory using `templates/scrape_configs.j2`
- The scrape template iterates `platform`, `gateway`, `mongodb`, `vault`, and all groups matching `^redis_.*`
- Per-host port overrides use `<exporter>_web_listen_address` hostvars; the default port from `defaults/main.yml` is used if the hostvar is absent

`itential.monitoring.grafana` is **RHEL/CentOS only** — tasks use `yum_repository`/`dnf` with no `ansible_os_family` guards. Bundled dashboard JSON files live in `roles/grafana/files/definitions/`; adding a file there is enough to include a new dashboard (no task change needed). The Prometheus datasource is provisioned from `groups.prometheus | first`.

`prometheus_exporters.yml` wraps four community exporter roles and adds firewalld rules and — for MongoDB — injects `MONGODB_USER`/`MONGODB_PASSWORD` env vars into the systemd service unit. Redis exporter requires `redis_prometheus_user_enabled: true` on redis hosts. MongoDB exporter requires `mongodb_exporter_global_conn_pool: true` when replication is enabled (prevents file descriptor exhaustion).

`site.yml` chains: exporters → prometheus → grafana.

---

### ELK Stack (centralized logs)

`elk.yml` chains: elasticsearch → logstash → kibana → filebeat.

**TLS is enabled by default** across all four ELK components (`*_tls_enabled: true`). Certificates must be pre-placed on hosts at `/etc/<role>/certs/` before any play runs — no role distributes certs.

**Elasticsearch** defaults to `discovery.type: single-node`. Setting both `elasticsearch_discovery_seed_hosts` and `elasticsearch_cluster_initial_master_nodes` switches to multi-node. JVM heap is deployed as a drop-in to `/etc/elasticsearch/jvm.options.d/itential-heap.options` (not overwriting the main `jvm.options`).

**Logstash** uses a keystore to store `ELASTIC_PASSWORD` at rest. Both `logstash_keystore_password` and `logstash_elastic_password` must be set via Ansible Vault — empty defaults will cause keystore creation to fail. The pipeline (`templates/pipeline.conf.j2`) routes events by the `app` field set in Filebeat input templates; automation failures are dual-written to `itential-failures-<date>`. The keystore task detects Logstash's lowercase key-name output in stdout for idempotency (`changed_when`).

**Filebeat** deploys a drop-in `inputs.d/` directory (`reload.enabled: true`). `main.yml` only deploys `input_default.yml` (syslog). All app-specific inputs are deployed via separate plays in `playbooks/filebeat.yml` using `import_role: tasks_from: filebeat_<app>` — this means app inputs are not deployed when running the role directly; use the `filebeat.yml` playbook.

---

### Loki / Alloy (lightweight log aggregation)

`loki.yml` installs Loki on the `loki` group and — in the same run — calls `tasks_from: loki_datasource` on the `grafana` group to wire the datasource. No separate Grafana playbook step is needed.

**Loki** is installed as a binary from GitHub releases (not via package manager). The binary is idempotent: if `loki --version` already matches `loki_version`, the download is skipped. Single-node filesystem storage only — `auth_enabled: false`. Changing `loki_data_dir` post-deployment requires manually migrating existing data.

**Alloy** replaces Promtail — the install block stops and disables `promtail` (`failed_when: false` so it does not fail if Promtail is absent). Installed via the Grafana RPM repository. The config template (`config.alloy.j2`) is validated with `alloy fmt %s` on every deploy; syntactically invalid HCL will fail the task before writing.

`playbooks/group_vars/` pre-configures `alloy_log_paths` and `alloy_extra_groups` for all standard Itential host groups. The only variable required in inventory is `alloy_loki_url` under `all.vars`. User-defined inventory group_vars take precedence over the collection's playbook group_vars.

`alloy_loki_url` must use the private/VPC-internal IP of the Loki host — EC2 instances cannot route to their own public IP. If this is left as the default empty string, the role deploys a broken config without failing.

`alloy_extra_groups` adds the `alloy` user to OS groups so it can read files not world-readable (e.g. `mongod.log` is `0640 mongod:mongod`). If omitted for a host group that requires it, Alloy silently fails to tail those files.

---

## Secrets (Ansible Vault required in production)

| Variable | Role |
|---|---|
| `logstash_keystore_password` | logstash |
| `logstash_elastic_password` | logstash |
| `mongodb_exporter_admin_password` | prometheus_exporters |
| `redis_exporter_password` | prometheus_exporters |
