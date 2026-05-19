# Underlay VXLAN Declarative Management

Idempotent, declarative management of VXLAN overlay networks. Handles the full lifecycle: bridge creation, VXLAN device creation, IP assignment, and static fdb entries.

## What It Manages

```
Phase 1: Bridges       — create if missing, ensure UP
Phase 2: VXLAN devices — create if missing, bind to bridge
Phase 3: Binding       — verify VXLAN is enslaved to correct bridge
Phase 4: MTU           — set bridge MTU to declared value
Phase 5: IP addresses  — add declared IPs if missing (from host_vars)
Phase 6: Static fdb    — ensure all peers have flood entries
```

All phases are idempotent — running the playbook multiple times produces no changes if state is already correct.

## Configuration

### defaults/main.yml (network topology)

```yaml
vxlan_networks:
  - bridge: vxbr0
    vni: 999
    dstport: 4789
    mtu: 1450
  - bridge: vxbr1
    vni: 1000
    dstport: 4789
    mtu: 1450
  - bridge: vxbr2
    vni: 1001
    dstport: 4789
    mtu: 1450

vtep_local_ip: "{{ ansible_host }}"
```

### host_vars (per-node IP addresses)

Define `vxlan_host_bridges` in host_vars or inventory to assign IPs:

```yaml
# host_vars/rack01.yml
vxlan_host_bridges:
  - bridge: vxbr0
    addresses:
      - "192.168.24.31/24"
  - bridge: vxbr1
    addresses:
      - "10.0.0.31/16"
  - bridge: vxbr2
    addresses:
      - "10.9.6.31/24"
```

If `vxlan_host_bridges` is not defined for a host, Phase 5 (IP assignment) is skipped — useful for nodes where IPs are managed by NetworkManager or another tool.

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

# Full declarative apply (creates missing infra + ensures fdb)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml

# Initialize a brand new node
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit new-rack

# Check mode (dry run)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --check
```

## Adding a New Node

1. Add to `[vxlan]` group in inventory
2. Optionally define `vxlan_host_bridges` in host_vars for IP assignment
3. Run playbook — all nodes get the new peer's fdb entry, new node gets full initialization

## Design Decisions

- **No nolearning**: All VXLAN interfaces keep `learning` mode. OVN provider bridges (VNI 1000) require data-plane MAC learning.
- **Handles naming differences**: Discovers existing VXLAN devices by VNI, not by name (supports both `vxlan999` and `vx999br0` naming).
- **IP assignment is optional**: Only runs if `vxlan_host_bridges` is defined for the host. Existing nodes managed by NetworkManager are unaffected.
- **fdb append, not replace**: Uses `append` with existence check to avoid duplicates while being safe on older iproute2 versions.
