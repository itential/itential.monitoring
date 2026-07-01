# kibana

Installs and configures Kibana for the Itential monitoring stack.

## Requirements

- RHEL/CentOS 8+
- Systemd
- Internet access to the Elastic package repository (or a local mirror)
- A running Elasticsearch instance

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `kibana_version` | `8.13.0` | Kibana version to install |
| `kibana_server_port` | `5601` | Port Kibana listens on |
| `kibana_server_host` | `0.0.0.0` | Network bind address |
| `kibana_server_name` | `{{ inventory_hostname }}` | Display name for the Kibana instance |
| `kibana_elasticsearch_hosts` | `["https://localhost:9200"]` | Elasticsearch hosts to connect to |
| `kibana_data_path` | `/var/lib/kibana` | Data directory |
| `kibana_log_path` | `/var/log/kibana` | Log directory |
| `kibana_server_base_path` | `""` | Base path when behind a reverse proxy (e.g. `/kibana`) |
| `kibana_server_rewrite_base_path` | `false` | Whether Kibana should rewrite requests using the base path |
| `kibana_tls_enabled` | `true` | Enable TLS for the Kibana server and the Elasticsearch connection |
| `kibana_tls_copy_certs` | `true` | Copy certificates from the control node to the target host |
| `kibana_pki_src_dir` | `""` | Directory on the control node containing the certificate files |
| `kibana_pki_base_dir` | `/etc/pki/kibana` | Base PKI directory on the target host |
| `kibana_tls_cert_file` | `{{ inventory_hostname }}.crt` | Certificate filename |
| `kibana_tls_key_file` | `{{ inventory_hostname }}.key` | Private key filename |
| `kibana_tls_ca_file` | `ca.crt` | CA certificate filename |

## TLS

TLS is enabled by default (`kibana_tls_enabled: true`). When enabled:
- The Kibana server serves HTTPS (`server.ssl.enabled: true`)
- The connection to Elasticsearch uses the CA cert for verification

Certificates must be in place on the host before the play runs. Set `kibana_tls_enabled: false`
in the inventory to disable TLS and also update `kibana_elasticsearch_hosts` to use `http://`.

## Example Playbook

```yaml
- hosts: kibana
  roles:
    - role: itential.monitoring.kibana
      vars:
        kibana_elasticsearch_hosts:
          - "https://es-node-1:9200"
          - "https://es-node-2:9200"
```

### TLS disabled

```yaml
- hosts: kibana
  roles:
    - role: itential.monitoring.kibana
      vars:
        kibana_tls_enabled: false
        kibana_elasticsearch_hosts:
          - "http://localhost:9200"
```

### Behind a reverse proxy

```yaml
- hosts: kibana
  roles:
    - role: itential.monitoring.kibana
      vars:
        kibana_server_base_path: "/kibana"
        kibana_server_rewrite_base_path: true
```

## Tags

| Tag | Description |
|---|---|
| `kibana_install` | Install the Kibana package and configure repositories |
| `kibana_configure` | Deploy kibana.yml |
| `always` | Ensure the service is running |
