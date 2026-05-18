# Infrastructure Playbooks

Playbooks that orchestrate custom roles before/after Kolla-ansible deployment.

## infra.yml

Unified entry point for all infrastructure automation.

```yaml
# Play 1: VXLAN underlay (all nodes in [vxlan] group)
- FRR BGP EVPN deployment
- VXLAN nolearning (excludes VNI 1000 / OVN provider bridge)
- Optional static fdb cleanup (--tags remove_static_fdb)

# Play 2: Monitoring stack (nodes in [monitoring] group)
- Thanos Sidecar + Query (systemd-managed)
- Grafana datasource provisioning
- Dashboard sync
```

## Usage

```bash
source /opt/kolla-venv/bin/activate

# Full run
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml

# Limit to specific hosts
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit rack01

# Only monitoring (Thanos + Grafana)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit monitoring

# Remove static fdb (after verifying BGP is up)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --tags remove_static_fdb
```
