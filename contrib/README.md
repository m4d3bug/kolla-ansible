# Homelab Infrastructure Extensions for Kolla-ansible

Custom Ansible roles that extend Kolla-ansible, sharing the same inventory (`/etc/kolla/multinode`).

## Components

| Role | Purpose | Hosts |
|------|---------|-------|
| [underlay-frr](underlay-frr/) | FRR BGP EVPN replaces static VXLAN fdb | `[vxlan]` (all 6 nodes) |
| [thanos](thanos/) | Thanos Sidecar/Query + Grafana datasource/dashboard | `[monitoring]` (rack01/02/03) |

## Quick Start

```bash
# On kolla-deploy VM (192.168.99.40)
source /opt/kolla-venv/bin/activate

# Deploy everything (FRR + Thanos + Grafana)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml

# After first Thanos deploy, load Grafana provisioning via Kolla
kolla-ansible reconfigure --tags grafana -i /etc/kolla/multinode
```

## Inventory Requirements

Append these groups to `/etc/kolla/multinode`:

```ini
[vxlan]
rack01 ansible_host=192.168.99.31 ansible_user=root vxlan_ip=192.168.24.31
rack02 ansible_host=192.168.99.32 ansible_user=root vxlan_ip=192.168.24.32
rack03 ansible_host=192.168.99.33 ansible_user=root vxlan_ip=192.168.24.33
rack10 ansible_host=192.168.99.10 ansible_user=root
rack05 ansible_host=192.168.99.35 ansible_user=root
rack04 ansible_host=192.168.99.34 ansible_user=root

[vxlan:vars]
frr_asn=65000
ansible_ssh_pass=m4d3bu9.com
ansible_ssh_common_args=-o StrictHostKeyChecking=no -o PubkeyAuthentication=no
```

`[monitoring]` group already exists in Kolla's inventory — just add `vxlan_ip` to those hosts.

## Deployment Order

```
1. infra.yml (FRR + Thanos)     ← this repo
2. kolla-ansible deploy          ← upstream kolla-ansible
3. kolla-ansible reconfigure     ← load Grafana/HAProxy custom configs
```

## Adding a New Node

1. Add to `[vxlan]` in inventory
2. Run: `ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml`
3. All existing nodes auto-learn the new VTEP via BGP

## Important Notes

- VNI 1000 is excluded from `nolearning` — it's OVN's provider bridge (Neutron external network)
- rack04 (ZOS) has immutable system files; role handles this automatically
- Thanos Query binds `0.0.0.0:10903` directly; no HAProxy proxy needed (VIP floats via keepalived)
- Dashboard JSONs live in `/etc/kolla/grafana/dashboards/` on monitoring nodes; source repo: `m4d3bug/homelab-grafana-dashboards`
