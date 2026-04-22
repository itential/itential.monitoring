# itential.monitoring.alloy

Installs and configures [Grafana Alloy](https://grafana.com/docs/alloy/latest/) on Itential
application nodes. Alloy collects the systemd journal and application log files from each host
and ships them to a Loki endpoint.

Alloy replaces Promtail — the role stops and disables the Promtail service if it is running.

## Requirements

- RHEL/Rocky Linux 9 (uses `dnf`)
- Grafana RPM repository reachable (added automatically by the role)
- A running Loki instance — set `alloy_loki_url` to its push endpoint

## Role Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `alloy_loki_url` | String | Loki push endpoint URL — **must be set** (e.g. `http://10.0.0.50:3100`) | `""` |
| `alloy_http_listen_port` | Integer | Alloy HTTP port (metrics, health, UI) | `12345` |
| `alloy_service_name` | String | Systemd service name | `alloy` |
| `alloy_log_paths` | List | File-based log paths to tail (set via group_vars) | `[]` |
| `alloy_extra_groups` | List | Extra OS groups to add the alloy user to for log access | `[]` |

## Tags

| Tag | Description |
|-----|-------------|
| `alloy_install` | Install package, configure repo, set up user groups, open firewall port |
| `alloy_configure` | Deploy `config.alloy` |

## Inventory Group Vars

Set `alloy_log_paths` and `alloy_extra_groups` per host group in your inventory. The role
deploys a single config per host driven entirely by these variables — no per-component task
files are needed.

### Platform (IAP)

```yaml
# group_vars/platform.yml
alloy_log_paths:
  - path: /var/log/itential/webserver.log
    job: iap-http
  - path: /var/log/itential/platform/*.log
    job: iap
  - path: /var/log/itential/platform*.log
    job: iap
```

### IAG5 Servers / Runners / Clients

```yaml
# group_vars/iag5_servers.yml (repeat for iag5_runners, iag5_clients)
alloy_extra_groups:
  - itential   # gateway.log is 0660 itential:itential
alloy_log_paths:
  - path: /var/log/gateway/gateway.log
    job: iag
```

### MongoDB

```yaml
# group_vars/mongodb.yml
alloy_extra_groups:
  - mongod     # mongod.log is 0640 mongod:mongod
alloy_log_paths:
  - path: /var/log/mongodb/mongod.log
    job: mongodb
```

### Redis

```yaml
# group_vars/redis_master.yml (repeat for redis_replica, redis_sentinel)
alloy_log_paths:
  - path: /var/log/redis/redis.log
    job: redis
  - path: /var/log/redis/sentinel.log
    job: redis-sentinel
```

## Playbook

```bash
ansible-playbook itential.monitoring.alloy -i <inventory>
```
