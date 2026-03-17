# Proxmox VE Local Testbed

Ansible playbooks and roles to stand up a local Proxmox VE instance as a nested KVM VM, then provision guest VMs inside it via the Proxmox API. Designed for testing VM provisioning workflows without dedicated hardware.

## Architecture

```
┌─────────────────────────────────────────────┐
│  Local Host (KVM/QEMU)                      │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │  Proxmox VE VM (10.100.0.2)           │ │
│  │  ┌──────────┐  ┌──────────┐           │ │
│  │  │ test-vm-1│  │ test-vm-2│  ...      │ │
│  │  │ .101     │  │ .102     │           │ │
│  │  └──────────┘  └──────────┘           │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  Network: proxmox-testbed (10.100.0.0/24)   │
└─────────────────────────────────────────────┘
```

## Prerequisites

- Linux host with KVM/QEMU and nested virtualization enabled
- libvirt, virt-install, xorriso, genisoimage
- Ansible 2.13+
- ~16GB free RAM, ~150GB free disk

## Quick Start

```bash
# Install Ansible collection dependencies
ansible-galaxy collection install -r requirements.yml

# Run the full testbed (takes ~30-45 min on first run)
ansible-playbook playbooks/site.yml

# Or run individual stages
ansible-playbook playbooks/00-prereqs.yml
ansible-playbook playbooks/01-network.yml
ansible-playbook playbooks/02-proxmox-vm.yml
ansible-playbook playbooks/03-proxmox-configure.yml
ansible-playbook playbooks/04-provision-vms.yml

# Tear everything down
ansible-playbook playbooks/teardown.yml
```

## Playbooks

| Playbook | Description |
|---|---|
| `00-prereqs.yml` | Validate host: nested virt, packages, Python deps |
| `01-network.yml` | Create libvirt NAT network `proxmox-testbed` |
| `02-proxmox-vm.yml` | Download PVE ISO, remaster with auto-installer, create VM |
| `03-proxmox-configure.yml` | Disable enterprise repo, create API token, build cloud-init template |
| `04-provision-vms.yml` | Clone and start guest VMs via Proxmox API |
| `site.yml` | Runs all of the above in order |
| `teardown.yml` | Destroys everything: guests, Proxmox VM, network |

## Roles

| Role | Purpose |
|---|---|
| `host_prepare` | Validate nested virt, install packages |
| `libvirt_network` | Manage the `proxmox-testbed` libvirt network |
| `proxmox_vm` | ISO download, auto-installer remaster, VM creation |
| `proxmox_configure` | Post-install PVE config, cloud-init template setup |
| `proxmox_guest` | Provision guest VMs via `community.proxmox` modules |

## Configuration

All variables live in `inventory/group_vars/all.yml`. Key settings:

| Variable | Default | Description |
|---|---|---|
| `proxmox_vm_ram_mb` | `16384` | RAM for the Proxmox VM |
| `proxmox_vm_vcpus` | `8` | vCPUs for the Proxmox VM |
| `proxmox_vm_disk_size` | `120G` | Disk size for the Proxmox VM |
| `proxmox_root_password` | `testbed123` | Root password (vault-encrypt for real use) |
| `guest_vms` | 2 VMs | List of guest VMs to provision |

## Debugging

If the Proxmox installer hangs, connect to the VNC console:

```bash
virt-viewer proxmox-ve
# or
vncviewer 127.0.0.1:5900
```

## Network

All hosts share `10.100.0.0/24` via a libvirt NAT network:

- `10.100.0.1` — host gateway
- `10.100.0.2` — Proxmox VE
- `10.100.0.101+` — guest VMs

Proxmox web UI: `https://10.100.0.2:8006`
