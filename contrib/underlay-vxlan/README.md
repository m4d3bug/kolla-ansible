# Underlay VXLAN Static FDB Management

Manages static bridge fdb entries for VXLAN overlay networks using Ansible. Adding a new node requires only updating the inventory and running the playbook — all nodes get updated automatically.

## How It Works

```
inventory [vxlan] group defines all VTEP peers
  → Ansible generates fdb entries for each node (excluding self)
  → bridge fdb append 00:00:00:00:00:00 dev <vxlan_dev> dst <peer_ip> self permanent
```

No additional daemons, no runtime dependencies. Pure static configuration managed by Ansible.

## Inventory

```ini
[vxlan]
rack01 ansible_host=192.168.99.31 ansible_user=root
rack02 ansible_host=192.168.99.32 ansible_user=root
rack03 ansible_host=192.168.99.33 ansible_user=root
rack10 ansible_host=192.168.99.10 ansible_user=root
rack05 ansible_host=192.168.99.35 ansible_user=root
rack04 ansible_host=192.168.99.34 ansible_user=root

[vxlan:vars]
ansible_ssh_pass=m4d3bu9.com
ansible_ssh_common_args=-o StrictHostKeyChecking=no -o PubkeyAuthentication=no
```

## Usage

```bash
source /opt/kolla-venv/bin/activate

# Apply static fdb to all VXLAN nodes
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml

# Target a single node
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit rack05
```

## Adding a New Node

1. Add the node to `[vxlan]` group in `/etc/kolla/multinode`
2. Run the playbook (all nodes get the new peer's fdb entry)

## Design Decisions

- **No nolearning**: VXLAN interfaces keep default `learning` mode. This is safe for OVN provider bridges (VNI 1000) which rely on data-plane MAC learning.
- **No FRR/BGP**: Static fdb is sufficient for a fixed-topology homelab (6 nodes). BGP EVPN adds complexity and risk (broke OVN metadata when nolearning was set on VNI 1000) without meaningful benefit at this scale.
- **Idempotent**: Only adds missing entries, does not touch existing ones.
- **fdb append vs replace**: Uses `append` so multiple peers can coexist on the same device. Checks for existing entries before adding to avoid duplicates.
