# Kolla-ansible Infrastructure Automation

Ansible roles and playbooks that extend Kolla-ansible's native infrastructure, sharing the same inventory (`/etc/kolla/multinode`).

## Directory Structure

```
kolla-ansible/
├── playbooks/
│   └── infra.yml              # Infrastructure playbook (pre-Kolla)
├── roles/
│   └── underlay-frr/          # FRR BGP EVPN VXLAN management
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/
│           ├── daemons.j2
│           └── frr.conf.j2
└── README.md
```

## Deployment Location

On kolla-deploy VM (192.168.99.40):
```
/etc/kolla/roles/underlay-frr/    # Role files
/etc/kolla/playbooks/infra.yml    # Playbook
/etc/kolla/multinode              # Shared inventory (add [vxlan] group)
```

## Inventory

Append to existing `/etc/kolla/multinode`:

```ini
[vxlan]
rack01 ansible_host=192.168.99.31 ansible_user=root
rack02 ansible_host=192.168.99.32 ansible_user=root
rack03 ansible_host=192.168.99.33 ansible_user=root
rack10 ansible_host=192.168.99.10 ansible_user=root
rack05 ansible_host=192.168.99.35 ansible_user=root
rack04 ansible_host=192.168.99.34 ansible_user=root

[vxlan:vars]
frr_asn=65000
ansible_ssh_pass=m4d3bu9.com
ansible_ssh_common_args=-o StrictHostKeyChecking=no -o PubkeyAuthentication=no
```

## Usage

```bash
# Activate Kolla virtualenv
source /opt/kolla-venv/bin/activate

# Deploy/update FRR on all VXLAN nodes
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml

# Remove legacy static fdb entries (after verifying BGP sessions are up)
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --tags remove_static_fdb

# Target a single node
ANSIBLE_ROLES_PATH=/etc/kolla/roles ansible-playbook -i /etc/kolla/multinode /etc/kolla/playbooks/infra.yml --limit rack05
```

## Adding a New Node

1. Add the node to `[vxlan]` group in `/etc/kolla/multinode`
2. Run the full playbook (all nodes get updated config with new peer)
3. Verify BGP sessions: `vtysh -c 'show bgp summary'`
4. Optionally clean static fdb: `--tags remove_static_fdb --limit <new_node>`

## Tested Environment (2026-05-18)

| Node | OS | FRR Version | BGP Peers | Status |
|------|-----|-------------|-----------|--------|
| rack01 | Debian 13 | 10.3 | 5/5 Established | ✅ |
| rack02 | Debian 13 | 10.3 | 5/5 Established | ✅ |
| rack03 | Debian 13 | 10.3 | 5/5 Established | ✅ |
| rack04 | ZOS (Ubuntu 22.04 based) | 8.1 | 5/5 Established | ✅ |
| rack05 | Ubuntu 24.04 | 10.1 | 5/5 Established | ✅ |
| rack10 | Fedora 43 | 10.4.1 | 5/5 Established | ✅ |

## Special Handling

- **rack04 (ZOS)**: Has immutable system files (`/etc/passwd`, `/etc/shadow`). Role automatically runs `chattr -i` before package installation when `ansible_distribution_version == 'zos'`.
- **nolearning**: Set at runtime via `ip link set`. Not persisted across reboots yet — needs NetworkManager/netplan integration (TODO).
- **Static fdb removal**: Uses `--tags never,remove_static_fdb` pattern. Only runs when explicitly requested.

## Verification

```bash
# Check BGP sessions
vtysh -c 'show bgp summary'

# Check EVPN VNI and remote VTEPs
vtysh -c 'show evpn vni'

# Check fdb entries (should be FRR-managed, not static)
bridge fdb show dev <vxlan_dev> | grep '00:00:00:00:00:00'
```
