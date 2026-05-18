# Thanos Monitoring Role

Deploys Thanos Sidecar + Query alongside Kolla-ansible's Prometheus, managed by systemd. Includes Grafana datasource provisioning and dashboard sync.

## Architecture

```
Thanos Query (0.0.0.0:10903, each monitoring node)
  ↕ connects to all sidecars
Thanos Sidecar (each monitoring node, reads local Prometheus TSDB)
  ↕
Prometheus (Kolla-managed, :9091)

Grafana → Thanos-Query (default datasource, via keepalived VIP)
```

No HAProxy proxy needed — Thanos Query binds `0.0.0.0:10903` on all monitoring nodes. Kolla's keepalived VIP floats between nodes, so VIP:10903 always reaches a live Query instance.

## Prerequisites

- Kolla-ansible deployed with `enable_prometheus: yes` and `enable_grafana: yes`
- Thanos container image: `quay.io/thanos/thanos:v0.37.0`
- Monitoring nodes in `[monitoring]` group with `vxlan_ip` variable set

## Inventory

Append `vxlan_ip` to monitoring hosts in `/etc/kolla/multinode`:

```ini
[monitoring]
rack01 ansible_host=192.168.99.31 ansible_user=root vxlan_ip=192.168.24.31
rack02 ansible_host=192.168.99.32 ansible_user=root vxlan_ip=192.168.24.32
rack03 ansible_host=192.168.99.33 ansible_user=root vxlan_ip=192.168.24.33
```

## Usage

```bash
source /opt/kolla-venv/bin/activate

# Deploy Thanos + Grafana datasource (included in infra.yml)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit monitoring

# After deploying, apply Grafana provisioning via Kolla
kolla-ansible reconfigure --tags grafana -i /etc/kolla/multinode
```

## Dashboard Sync

Dashboards are stored in `/etc/kolla/grafana/dashboards/` on each monitoring node. To update:

```bash
# Copy new/updated JSON files from git repo
scp ~/grafana-dashboards/*.json root@192.168.99.31:/etc/kolla/grafana/dashboards/
scp ~/grafana-dashboards/*.json root@192.168.99.32:/etc/kolla/grafana/dashboards/
scp ~/grafana-dashboards/*.json root@192.168.99.33:/etc/kolla/grafana/dashboards/

# Restart Grafana to load
podman restart grafana  # on each node, or via kolla-ansible reconfigure --tags grafana
```

## Systemd Units

| Unit | Description |
|------|-------------|
| `thanos-sidecar.service` | Reads local Prometheus TSDB, serves via gRPC :10902 |
| `thanos-query.service` | Aggregates all sidecars, serves Prometheus-compatible API :10903 |

```bash
systemctl status thanos-sidecar thanos-query
journalctl -u thanos-sidecar -f
```

## Verification

```bash
# Health check
curl http://192.168.24.31:10903/-/healthy

# Cross-rack data
curl -s 'http://192.168.24.31:10903/api/v1/query?query=up' | python3 -m json.tool | head

# Grafana datasources
curl -u admin:<password> http://192.168.99.90:3000/api/datasources
```

## Tested (2026-05-18)

| Node | Thanos Sidecar | Thanos Query | Grafana Datasource |
|------|---------------|-------------|-------------------|
| rack01 | active | active | Thanos-Query (default) |
| rack02 | active | active | Thanos-Query (default) |
| rack03 | active | active | Thanos-Query (default) |

18 dashboards loaded via Kolla provisioning path.

## Known Issues

- Thanos container runs as non-root; `http_client.yaml` must be mode 0644 (not 0600)
- Migration from podman `--restart unless-stopped` to systemd requires container recreation (handled automatically by role)
- HAProxy proxy for Thanos causes port conflict (Query already binds the port) — template retained for reference but task removed
