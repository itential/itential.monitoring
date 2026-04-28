# elasticsearch role

## Purpose
Installs Elasticsearch from the official Elastic package repository and deploys configuration. Supports both single-node and multi-node cluster topologies.

## Key files
| File | Purpose |
|---|---|
| `defaults/main.yml` | All role variables with defaults |
| `tasks/main.yml` | Three tagged blocks: install, configure, ensure running |
| `templates/elasticsearch.yml.j2` | Main Elasticsearch config — controls cluster, network, discovery, and TLS |
| `templates/jvm.options.j2` | JVM heap — deployed as a drop-in to `/etc/elasticsearch/jvm.options.d/itential-heap.options`, not the main `jvm.options` |
| `handlers/main.yml` | Single handler: Restart Elasticsearch |

## Topology
- `elasticsearch_discovery_seed_hosts: []` (default empty) → sets `discovery.type: single-node`
- Populating `elasticsearch_discovery_seed_hosts` and `elasticsearch_cluster_initial_master_nodes` switches to multi-node cluster mode

## TLS / Security
`elasticsearch_tls_enabled` controls three xpack flags together: `xpack.security.enabled`, `xpack.security.transport.ssl.enabled`, and `xpack.security.http.ssl.enabled`. Elasticsearch 8.x requires all three to be consistent — do not split them. TLS defaults to `true`.

Certificates must be pre-placed on the host before the play runs. The role does not distribute certs.

## OS support
Supports both RHEL/CentOS (`rpm_key` + `yum_repository`) and Debian/Ubuntu (`apt_key` + `apt_repository`). All package tasks are guarded with `when: ansible_os_family == "RedHat/Debian"`.

## User and role provisioning
`tasks/create-users.yml` creates the `logstash_writer` role and user, the `grafana_reader` role, and applies the `itential-logs-policy` ILM policy via the Elasticsearch REST API. It runs after the service is confirmed active and is tagged `elasticsearch_users`. Both `elasticsearch_elastic_password` and `elasticsearch_logstash_writer_password` must be set (use Ansible Vault).

## Variables that must be overridden
None are strictly required — the defaults produce a working single-node instance with TLS enabled. However:
- `elasticsearch_heap_size` should be tuned (no more than 50% of host RAM)
- `elasticsearch_tls_enabled: false` requires certs to be absent or the template will still reference the cert paths
