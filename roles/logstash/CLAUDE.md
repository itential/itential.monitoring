# logstash role

## Purpose
Installs Logstash from the official Elastic package repository, deploys an Itential-specific pipeline, and manages the Logstash keystore for secrets at rest.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three tagged blocks: install, configure, ensure running |
| `templates/logstash.yml.j2` | Main Logstash settings (paths, pipeline workers) |
| `templates/jvm.options.j2` | JVM heap — deployed to `/etc/logstash/jvm.options` |
| `templates/pipeline.conf.j2` | Itential pipeline — input (Beats), filter (per-app enrichment), output (Elasticsearch) |
| `handlers/main.yml` | Single handler: Restart Logstash |

## Keystore management
The configure block manages the keystore in this order:
1. `stat` check on `/etc/logstash/logstash.keystore`
2. Create keystore (skipped if file exists) — protected by `logstash_keystore_password`
3. Remove existing `ELASTIC_PASSWORD` key (always runs, `failed_when: false`)
4. Add `ELASTIC_PASSWORD` with the value of `logstash_elastic_password`
5. Write `LOGSTASH_KEYSTORE_PASS` to `/etc/sysconfig/logstash` so the service can decrypt the keystore at startup

The pipeline template references `${ELASTIC_PASSWORD}`, which Logstash resolves from the keystore at runtime.

## Variables that must be set
Both of these have empty string defaults and will cause the keystore create to fail if not overridden. Use Ansible Vault:
- `logstash_keystore_password` — Logstash 8+ does not allow empty keystore passwords
- `logstash_elastic_password` — password for the `logstash_writer` Elasticsearch user

## changed_when behavior
- "Create Logstash keystore" detects success via `'Created Logstash keystore'` in stdout
- "Set ELASTIC_PASSWORD" detects success via `'Added 'elastic_password' to the Logstash keystore'` in stdout — Logstash outputs the key name in lowercase even when the key was added as uppercase `ELASTIC_PASSWORD`

## Pipeline routing
The pipeline at `/etc/logstash/conf.d/itential.conf` routes by the `app` field set by Filebeat inputs. Automation failures are dual-written to `itential-failures-<date>`. All events go to `itential-logs-<app>-<date>`.

## OS support
RHEL/CentOS only. Package install uses `rpm_key` + `yum_repository`. Keystore password is written to `/etc/sysconfig/logstash`.

## Known gaps
- The `logstash_writer` Elasticsearch user is not created by any role or playbook. Authentication worked in testing without it, which suggests the Elasticsearch security configuration may allow it — this should be investigated.
