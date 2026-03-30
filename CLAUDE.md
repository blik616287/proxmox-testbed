# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbooks and roles that stand up a local Proxmox VE instance as a nested KVM VM on a Linux host, then provision guest VMs inside it via the Proxmox API. Designed for testing VM provisioning workflows without dedicated hardware.

## Commands

```bash
# Install Ansible collection dependencies (required first time)
ansible-galaxy collection install -r requirements.yml

# Run the full testbed (~30-45 min first run)
ansible-playbook playbooks/site.yml

# Run individual stages in order
ansible-playbook playbooks/00-prereqs.yml    # Validate host
ansible-playbook playbooks/01-network.yml    # Create libvirt network
ansible-playbook playbooks/02-proxmox-vm.yml # Download ISO, remaster, create VM
ansible-playbook playbooks/03-proxmox-configure.yml  # Configure PVE + cloud-init template
ansible-playbook playbooks/04-provision-vms.yml      # Clone and start guest VMs

# Tear everything down
ansible-playbook playbooks/teardown.yml

# Syntax check a playbook
ansible-playbook --syntax-check playbooks/<playbook>.yml

# Dry run (check mode)
ansible-playbook --check playbooks/<playbook>.yml

# Run with verbose output for debugging
ansible-playbook -vvv playbooks/<playbook>.yml

# Debug Proxmox VM install via VNC
virt-viewer proxmox-ve
```

## Architecture

The testbed runs entirely on one Linux host using nested virtualization:

1. **Host** runs KVM/QEMU + libvirt
2. **libvirt NAT network** (`proxmox-testbed`, `10.100.0.0/24`) connects everything
3. **Proxmox VE VM** (`10.100.0.2`) is installed from a remastered ISO with auto-installer
4. **Guest VMs** (`10.100.0.101+`) are cloned from a cloud-init Ubuntu template inside Proxmox

Playbooks run sequentially: host prep -> network -> Proxmox VM install -> PVE configuration -> guest provisioning.

## Key Implementation Details

**ISO Remastering** (`roles/proxmox_vm`): The Proxmox ISO is remastered with an embedded `answer.toml` (from `templates/proxmox-answer.toml.j2`) and patched `grub.cfg` for unattended serial-console install. Uses `xorriso` to rebuild a hybrid MBR+UEFI bootable ISO.

**Proxmox VM Installation**: Uses `virt-install` with `--cpu host-passthrough` for nested virt. Serial console output logs to `/var/log/proxmox-testbed/console.log`. After install, the VM is re-defined to boot from disk and the playbook polls the Proxmox API until it responds.

**Cloud-Init Template** (`roles/proxmox_configure`): Downloads Ubuntu cloud image, creates a Proxmox VM template (vmid 9000) with `qm` commands — imports disk to `local-lvm`, attaches cloud-init drive on `ide2`, serial console on `vga serial0`.

**Guest Provisioning** (`roles/proxmox_guest`): Full-clones from the template via `proxmox_kvm` module, configures cloud-init networking (static IP), resizes disks, starts VMs, and waits for SSH.

## Variables

All configuration lives in `inventory/group_vars/all.yml`. Key variables: `proxmox_vm_ram_mb`, `proxmox_vm_vcpus`, `proxmox_vm_disk_size`, `proxmox_root_password`, `guest_vms` list, cloud image URL, and network subnet settings.

## Ansible Dependencies

Required collections (defined in `requirements.yml`): `community.proxmox`, `community.libvirt`, `community.general`, `ansible.posix`. Python packages: `proxmoxer`, `requests`.

## Inventory Groups

- `hypervisor` — localhost (runs libvirt/KVM tasks)
- `proxmox` — the Proxmox VE VM (runs PVE configuration and guest provisioning)

## Templates

- `templates/proxmox-answer.toml.j2` — Proxmox auto-installer answer file (TOML format)
- `roles/libvirt_network/templates/network.xml.j2` — libvirt NAT network definition XML
