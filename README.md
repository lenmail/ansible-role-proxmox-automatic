# Ansible Role: proxmox_automatic

[![CI](https://github.com/lenmail/ansible-role-proxmox-automatic/workflows/CI/badge.svg)](https://github.com/lenmail/ansible-role-proxmox-automatic/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-proxmox__automatic-blue.svg)](https://galaxy.ansible.com/lenmail/proxmox_automatic)
[![Ansible](https://img.shields.io/badge/ansible-2.14%2B-red.svg)](https://ansible.com)
[![GitHub release](https://img.shields.io/github/release/lenmail/ansible-role-proxmox-automatic.svg)](https://github.com/lenmail/ansible-role-proxmox-automatic/releases)

This role automatically creates virtual machines on a **Proxmox VE Cluster** with backend-specific unattended installer media. It currently supports **Rocky Linux via Kickstart** and **Debian 13 via Preseed**, generates the required ISO artifacts, uploads them to Proxmox, and starts VMs with complex storage and network configurations.

For `rocky_kickstart`, the role only generates the second config CD. The primary Rocky/RHEL installer ISO must already be prepared to load `ks.cfg` from that second CD by boot parameter.

## 🚀 Features

- **Automatic Dependency Installation**: Installs `xorriso` and `syslinux` on various distributions
- **Hierarchical Configuration**: Defaults → group_vars → host_vars → Playbook variables
- **Multi-OS Installer Backends**: `rocky_kickstart` and `debian13_preseed`
- **Dynamic Storage Configuration**: Multi-disk support with LVM and flexible sizes
- **Automatic Config Generation**: Individual Kickstart or Preseed files per VM
- **ISO Management**: Rocky uses original ISO + config CD, Debian remasters a bootable installer ISO
- **Install Source Modes**: Online mirrors or CD-only installation, controlled by variables
- **Flexible User Configuration**: Hierarchical user management with SSH keys
- **Multi-Network Support**: Multiple network interfaces with VLAN support
- **Rocky Linux 9 Base Provisioning**: Minimal first-boot basis without additional shell customizations
- **Debian 13 Ready**: Remastered ISO flow with embedded `preseed.cfg`

## 📋 Requirements

### Proxmox VE
- API user with VM management permissions
- Storage for VMs and ISOs

### Ansible Controller Dependencies

**Automatic installation (recommended):**
```yaml
proxmox_automatic_install_dependencies: true
```

**Supported systems:**
- Red Hat family (RHEL, Rocky, CentOS, Fedora)
- Debian family (Debian, Ubuntu)
- macOS (via Homebrew)
- SUSE family (openSUSE, SLES)
- Arch family (Arch, Manjaro)

### Collections

**Required Ansible collections:**
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

**Note:** This role uses the new `community.proxmox` collection.

## 📚 Quick Start

### Minimal Configuration

```yaml
# playbook.yml
- hosts: proxmox_vms
  roles:
    - lenmail.proxmox_automatic
  vars:
    proxmox_automatic_api_host: "pve.example.com"
    proxmox_automatic_api_password: "your-api-password"
    proxmox_automatic_hypervisor: "pve-node1"
    proxmox_automatic_installer_backend: "rocky_kickstart"
```

```ini
# inventory
[proxmox_vms]
web01 proxmox_automatic_vmid=101 ansible_host=web01.example.com
db01 proxmox_automatic_vmid=102 ansible_host=db01.example.com
```

`ansible_host` is currently also used as the Proxmox VM name and as the guest hostname in the generated installer configuration. From an operations perspective, treat `ansible_host` as the desired system FQDN for this role, not only as a transport address.

### Example Inventory Layout

This repository ships a complete example inventory under [examples/inventory](/Users/dominik.lenhardt/git/ansible_roles/ansible-role-proxmox-automatic/examples/inventory) plus an example playbook at [examples/playbook.yml](/Users/dominik.lenhardt/git/ansible_roles/ansible-role-proxmox-automatic/examples/playbook.yml).

Structure:

```text
examples/
├── playbook.yml
└── inventory/
    ├── hosts.yml
    ├── group_vars/
    │   ├── all.yml
    │   ├── rocky_vms.yml
    │   └── debian_vms.yml
    └── host_vars/
        ├── rocky01.example.com.yml
        ├── rocky02.example.com.yml
        ├── debian01.example.com.yml
        └── debian02.example.com.yml
```

Usage:

```bash
ansible-playbook -i examples/inventory/hosts.yml examples/playbook.yml
```

The example inventory demonstrates:
- shared API, storage and HA settings in `group_vars/all.yml`
- backend-specific defaults in `group_vars/rocky_vms.yml` and `group_vars/debian_vms.yml`
- per-host VM IDs, IPs and install source modes in `host_vars/`
- both online and CD-only installation flows
- Debian with multiple NICs

### Debian 13 Example

```yaml
# host_vars/swarm01.yml
proxmox_automatic_installer_backend: "debian13_preseed"
proxmox_automatic_source_iso_path: "/srv/iso/debian-13.0.0-amd64-netinst.iso"
proxmox_automatic_networks:
  - name: net0
    interface_name: ens18
    bridge: vmbr1
    vlanid: 30
    ip: "192.0.2.11"
    netmask: "255.255.255.0"
    gateway: "192.0.2.1"
proxmox_automatic_users:
  - name: "litsadmin"
    fullname: "LITS Admin"
    password: "$6$..."
    password_encrypted: true
    groups: ["sudo"]
    sudo_commands: "ALL"
    sudo_nopasswd: true
    ssh_key: "ssh-ed25519 AAAA..."
```

Generated installer media names are derived automatically from the target FQDN in `ansible_host`:
- Rocky Kickstart ISO: `ks-<ansible_host>`
- Debian Preseed ISO: `preseed-<ansible_host>`

Set `proxmox_automatic_generated_iso_name` only when you explicitly need to override that artifact name.

### GPU Passthrough Example

For workloads that require direct PCI device access, define the Proxmox `hostpci` map in host variables. When the passed-through GPU should be the only graphics device, set `proxmox_automatic_vga` to `none`.

```yaml
# host_vars/llm.example.com.yml
proxmox_automatic_memory: 36864
proxmox_automatic_vcpu: 12
proxmox_automatic_vga: "none"
proxmox_automatic_hostpci:
  hostpci0: "0000:04:00.0,pcie=1,x-vga=1"
  hostpci1: "0000:05:00.0,pcie=1"
```

From a DevOps and platform architecture perspective, keep the PCI mapping host-specific. PCI addresses are node-local hardware facts and should not be modeled as shared group defaults unless every target node is intentionally identical.

### Installation Source Modes

```yaml
proxmox_automatic_install_source: "online"  # or "cdrom"
```

From a DevOps operating model, this variable controls whether the installer should use external repositories or only the attached installation media.

- `online`
  - Rocky uses the configured `url` and `repo` sources.
  - Debian uses the configured HTTP mirrors plus security repositories.
- `cdrom`
  - Rocky installs only from the attached installer ISO.
  - Debian disables mirrors and keeps the installer CD as the only package source.
  - Debian also reduces the installer package set to packages that are expected on the attached medium. Components such as `qemu-guest-agent` should be applied afterwards through Ansible if the chosen ISO does not ship them.

For Debian, `cdrom` mode is only robust when the source ISO already contains the required packages.

- A plain `netinst` ISO is usually too small for a true offline workflow.
- Use a full Debian DVD image or a curated custom ISO if Dominik needs deterministic CD-only provisioning.

### IPv6 Configuration Example

```yaml
# group_vars/all.yml
proxmox_automatic_ipv6_enabled: true
proxmox_automatic_ipv6_dns_servers:
  - "2606:4700:4700::1111"
  - "2606:4700:4700::1001"

# host_vars/web01.yml
proxmox_automatic_networks:
  - name: net0
    bridge: vmbr0
    ip: "192.168.1.10"
    netmask: "255.255.255.0"
    gateway: "192.168.1.1"
    ipv6: "2001:db8:1::10/64"
    ipv6_gateway: "2001:db8:1::1"
```

### High Availability Configuration Example

#### Option 1: Automatic HA Rule per Hypervisor (Default & Recommended)

```yaml
# group_vars/all.yml or defaults
proxmox_automatic_ha_enabled: true          # Default: true
proxmox_automatic_ha_group_auto: true       # Default: true
proxmox_automatic_ha_group_manage: true     # Default: true

# Inventory
[proxmox_vms]
web01 proxmox_automatic_hypervisor=srv-hyp-01.local
web02 proxmox_automatic_hypervisor=srv-hyp-01.local
db01 proxmox_automatic_hypervisor=srv-hyp-02.local
db02 proxmox_automatic_hypervisor=srv-hyp-03.local
```

**Result:**
- HA rule `ha-group-srv-hyp-01` created with shared node-affinity for `srv-hyp-01`
  - VMs: web01, web02
- HA rule `ha-group-srv-hyp-02` created with shared node-affinity for `srv-hyp-02`
  - VMs: db01
- HA rule `ha-group-srv-hyp-03` created with shared node-affinity for `srv-hyp-03`
  - VMs: db02

**Benefits:**
- ✅ VMs prefer their hypervisor (priority 0)
- ✅ Automatic failover to other cluster nodes
- ✅ Zero manual configuration needed
- ✅ One shared HA rule per hypervisor

#### Option 2: Manual HA Rule Assignment

```yaml
# group_vars/production.yml
proxmox_automatic_ha_enabled: true
proxmox_automatic_ha_group_auto: false      # Disable auto mode
proxmox_automatic_ha_group_manage: true
proxmox_automatic_ha_group: "ha-group-production"
proxmox_automatic_ha_group_nodes: "pve-node1:0,pve-node2:1,pve-node3:2"
proxmox_automatic_ha_group_nofailback: false
proxmox_automatic_ha_group_comment: "Production HA rule - shared across all VMs"

# host_vars/db01.yml
proxmox_automatic_ha_comment: "Primary database server - critical service"
```

**Use case:** All VMs in a rule should share the same HA node-affinity configuration

#### Option 3: Using Existing HA Rules

```yaml
# group_vars/production.yml
proxmox_automatic_ha_enabled: true
proxmox_automatic_ha_group_auto: false      # Disable auto mode
proxmox_automatic_ha_group_manage: false    # Use existing rule
proxmox_automatic_ha_group: "ha-group-production"

# host_vars/db01.yml
proxmox_automatic_ha_comment: "Primary database server"
```

**Note:** The HA rule must already exist in your Proxmox cluster:
- Proxmox Web UI: Datacenter → HA
- CLI/API: create a node-affinity rule that already contains the intended resources and nodes

#### Option 4: Disable HA

```yaml
# group_vars/development.yml
proxmox_automatic_ha_enabled: false
```

## 🎯 Understanding Hierarchical Configuration

This role uses an intelligent merge system that combines configurations from different levels:

**Priority Order (highest to lowest):**
```
Playbook variables → host_vars/ → group_vars/ → defaults/main.yml
```

### Storage Configuration Hierarchy

The role supports three levels of storage configuration:

- `proxmox_automatic_storage_config_defaults` - Base configuration
- `proxmox_automatic_storage_config_group_vars` - Group-specific additions
- `proxmox_automatic_storage_config_host_vars` - Host-specific configuration

**Example:**

```yaml
# defaults/main.yml - All VMs get this base disk
proxmox_automatic_storage_config_defaults:
  - name: virtio0
    size: "20"
    disk: vda
    bootloader: true

# group_vars/databases.yml - Database servers get additional storage
proxmox_automatic_storage_config_group_vars:
  - name: virtio0
    size: "40"  # Override base size
  - name: virtio1
    size: "200"  # Additional data disk

# host_vars/db01.yml - Specific host gets backup disk
proxmox_automatic_storage_config_host_vars:
  - name: virtio2
    size: "100"  # Backup storage
```

**Result:** `db01` will have 3 disks: 40GB system, 200GB data, 100GB backup.

### Package Configuration Hierarchy

Packages are split into two categories for maximum flexibility:

- `proxmox_automatic_packages_default` - Base packages installed on **all** VMs
- `proxmox_automatic_packages_additional` - Additional packages per group/host

**Example:**

```yaml
# defaults/main.yml - Base packages on all VMs
proxmox_automatic_packages_default:
  - bash-completion
  - vim
  - git
  - htop
  - tcpdump
  # ... many more essential tools

# group_vars/webservers.yml - Web-specific packages
proxmox_automatic_packages_additional:
  - httpd
  - php
  - php-mysqlnd

# host_vars/web01.yml - Host-specific package
proxmox_automatic_packages_additional:
  - php-redis  # Only web01 needs Redis
```

**Result:** All VMs get the base tools, webservers get PHP/Apache, web01 additionally gets php-redis.

### User Configuration Hierarchy

Similar hierarchical approach for users:

```yaml
# defaults/main.yml - Standard admin user
proxmox_automatic_users_defaults:
  root:
    password: "$6$encrypted_password"
    ssh_key: "ssh-rsa AAAA..."

# group_vars/webservers.yml - Web-specific service user
proxmox_automatic_users_group_vars:
  nginx:
    shell: "/bin/false"
    home: "/var/www"

# host_vars/web01.yml - Host-specific developer access
proxmox_automatic_users_host_vars:
  developer:
    password: "$6$dev_password"
    groups: ["wheel"]
```

### Network Configuration Hierarchy

```yaml
# defaults/main.yml - Standard network
proxmox_automatic_networks_defaults:
  - name: net0
    bridge: vmbr0

# group_vars/databases.yml - Database VLAN
proxmox_automatic_networks_group_vars:
  - name: net1
    bridge: vmbr1
    vlanid: 200

# host_vars/db01.yml - Additional management interface
proxmox_automatic_networks_host_vars:
  - name: net2
    bridge: vmbr2
    vlanid: 100
```

## 🛠️ Configuration

### Variable Documentation

For a complete list of all 96 configuration variables with detailed descriptions and examples, see **[VARIABLES.md](VARIABLES.md)**.

### Most Important Variables

**Required:**
- `proxmox_automatic_api_host` - Proxmox API host
- `proxmox_automatic_api_password` - Proxmox API password
- `proxmox_automatic_hypervisor` - Target Proxmox node

**Commonly Used:**
```yaml
# VM Resources
proxmox_automatic_memory: 2048          # RAM in MB
proxmox_automatic_vcpu: 2               # CPU cores

# Network
proxmox_automatic_networks:
  - name: net0
    bridge: vmbr0
    ip: "192.168.1.10"
    netmask: "255.255.255.0"
    gateway: "192.168.1.1"
    vlanid: 100

# Storage
proxmox_automatic_storage_config:
  - name: virtio0
    size: "20"
    disk: vda
    bootloader: true

# High Availability (enabled by default)
proxmox_automatic_ha_enabled: true      # Default: true
proxmox_automatic_ha_group_auto: true   # Auto HA rule per hypervisor

# Packages
proxmox_automatic_packages_additional:
  - "httpd"
  - "php"
```

### Configuration Hierarchy

This role uses a powerful hierarchical configuration system:

```
defaults/main.yml  →  group_vars/  →  host_vars/  →  playbook vars
   (lowest)              (medium)       (higher)       (highest)
```

**Example:** Storage Configuration Merging
```yaml
# defaults/main.yml - Base disk for all VMs
proxmox_automatic_storage_config_defaults:
  - name: virtio0
    size: "20"

# group_vars/databases.yml - Additional disk for DB servers
proxmox_automatic_storage_config_group_vars:
  - name: virtio1
    size: "100"

# host_vars/db-master.yml - Extra disk for master only
proxmox_automatic_storage_config_host_vars:
  - name: virtio2
    size: "200"

# Result: db-master gets 3 disks (20GB + 100GB + 200GB)
```

## ⚙️ Quick Reference

### Variable Categories

| Category | Variables | Description |
|----------|-----------|-------------|
| **[Core](VARIABLES.md#core-configuration)** | 4 vars | Installation paths, dependencies |
| **[Proxmox API](VARIABLES.md#proxmox-api-configuration)** | 4 vars | API credentials, SSL validation |
| **[VM Config](VARIABLES.md#vm-configuration)** | 12 vars | CPU, RAM, boot settings |
| **[Storage](VARIABLES.md#storage-configuration)** | 5 vars | Disks, volumes, LVM |
| **[Network](VARIABLES.md#network-configuration)** | 10 vars | Interfaces, VLANs, IPv6 |
| **[Security](VARIABLES.md#security-configuration)** | 3 vars | Firewall, SELinux, SSH |
| **[Users](VARIABLES.md#user-management)** | 6 vars | Accounts, SSH keys |
| **[High Availability](VARIABLES.md#high-availability-configuration)** | 11 vars | HA rules, failover |
| **[Packages](VARIABLES.md#package-management)** | 7 vars | Software installation |
| **[Performance](VARIABLES.md#performance-tuning)** | 8 vars | Throttling, retries |

For detailed documentation of all variables, see **[VARIABLES.md](VARIABLES.md)**.

## 📚 Examples

### Database Server with Multiple Disks

```yaml
# host_vars/db01.yml
proxmox_automatic_memory: 8192
proxmox_automatic_vcpu: 4

# Storage hierarchy example
proxmox_automatic_storage_config_host_vars:
  - name: virtio0  # System disk
    size: "40"
    disk: vda
    bootloader: true
    vg:
      - name: system
        lv:
          - name: root
            mount: /
            size: 8192
            fstype: xfs
          - name: swap
            mount: swap
            size: 4096
            fstype: swap
          - name: var
            mount: /var
            size: 4096
            fstype: xfs
          - name: tmp
            mount: /tmp
            size: 2048
            fstype: xfs
            
  - name: virtio1  # Database storage
    size: "200"
    disk: vdb
    vg:
      - name: database
        lv:
          - name: mysql
            mount: /var/lib/mysql
            size: 1024
            grow: true
            fstype: xfs
            
  - name: virtio2  # Log storage
    size: "50"
    disk: vdc
    vg:
      - name: logs
        lv:
          - name: mysqllogs
            mount: /var/log/mysql
            size: 1024
            grow: true
            fstype: xfs

# Network configuration
proxmox_automatic_networks_host_vars:
  - name: net1
    bridge: vmbr1
    ip: "10.10.20.10"
    netmask: "255.255.255.0"
    vlanid: 200  # Database VLAN

# Database-specific packages
proxmox_automatic_packages_additional:
  - "mariadb-server"
  - "mariadb-client"
  - "python3-pymysql"
```

### Web Server Cluster

```yaml
# group_vars/webservers.yml
proxmox_automatic_memory: 4096
proxmox_automatic_vcpu: 2

# Web servers get additional storage for content
proxmox_automatic_storage_config_group_vars:
  - name: virtio1
    size: "100"
    disk: vdb
    vg:
      - name: web
        lv:
          - name: www
            mount: /var/www
            size: 1024
            grow: true
            fstype: xfs

# Web VLAN
proxmox_automatic_networks_group_vars:
  - name: net1
    bridge: vmbr1
    ip: "10.10.10.10"
    netmask: "255.255.255.0"
    vlanid: 100

# Web-specific packages
proxmox_automatic_packages_additional:
  - "httpd"
  - "php"
  - "php-mysqlnd"

# Web admin user
proxmox_automatic_users_group_vars:
  webadmin:
    password: "$6$rounds=656000$YourSaltHere$YourHashHere"
    groups: ["wheel", "apache"]
    shell: "/bin/bash"
```

### Development Environment

```yaml
# group_vars/development.yml
proxmox_automatic_selinux_mode: "permissive"
proxmox_automatic_firewall_enabled: false

# Development packages (additional to base packages)
proxmox_automatic_packages_additional:
  - "nodejs"
  - "npm"
  - "python3-pip"
  - "docker-ce"
  - "docker-ce-cli"
  - "containerd.io"

# Developer users with sudo access
proxmox_automatic_users_group_vars:
  developer:
    password: "$6$rounds=656000$DevSalt$DevHash"
    groups: ["wheel", "docker"]
    ssh_key: "ssh-rsa AAAA...developer-key"

# Additional development repositories
proxmox_automatic_repos:
  - name: "docker-ce"
    baseurl: "https://download.docker.com/linux/centos/9/$basearch/stable"
    gpgcheck: true
    gpgkey: "https://download.docker.com/linux/centos/gpg"
```

## 🚀 Advanced Usage

### Custom Installer Templates

You can extend the role by providing backend-specific templates:

```yaml
# Rocky
proxmox_automatic_kickstart_template: "custom-kickstart.cfg.j2"

# Debian
proxmox_automatic_preseed_template: "custom-preseed.cfg.j2"
```

### Backend Model

- `rocky_kickstart`: keeps the original OS ISO on `ide2` and generates a second config ISO on `ide3`.
- `debian13_preseed`: renders `preseed.cfg`, remasters the Debian installer ISO on the controller, and attaches only the generated installer ISO.
- `proxmox_automatic_install_source` selects whether package installation should use online repositories or CD-only media.
- `proxmox_automatic_source_iso_path` is recommended for Debian remastering workflows.
- `proxmox_automatic_generated_iso_name` is only an optional override. By default the role generates unique artifact names from `ansible_host`.
- `proxmox_automatic_debian_purge_microcode` defaults to `true` because Debian 13 can auto-install virtual CPU microcode packages that leave the generated `initrd.img` unusable on Proxmox/QEMU guests.

For Debian networking, the installer itself configures the primary NIC only. Additional NICs from `proxmox_automatic_networks` are rendered into the installed system during `late_command`, so static multi-NIC server builds are supported even though the installer continues to work off `net0`.

### Rocky / RHEL Source ISO Preparation

For `rocky_kickstart`, the role does **not** patch the Rocky or RHEL bootloader. As a DevOps operating model, this is intentional: the role owns the generated Kickstart payload on the second CD, while the base installer ISO remains a curated platform artifact.

That means a stock Rocky ISO is not sufficient by itself. The primary installer ISO must already contain boot parameters that tell Anaconda where to find the Kickstart file on the second CD.

Recommended boot parameter pattern:

```text
inst.ks=hd:LABEL=KS_<fqdn>:/ks.cfg
```

Example for host `swarm01.lenmail.de`:

```text
inst.ks=hd:LABEL=KS_swarm01.lenmail.de:/ks.cfg
```

The role creates the second ISO with:

- volume label `KS_<ansible_host>`
- file `/ks.cfg`

Operationally, you have two workable approaches:

1. Maintain one pre-adapted Rocky/RHEL installer ISO per host class and set the appropriate `inst.ks=` boot parameter there.
2. Maintain a small set of curated installer ISOs per environment if you want different boot defaults for lifecycle stage, platform generation, or storage policy.

Example workflow on a Linux build host to adapt a Rocky ISO:

```bash
mkdir -p /tmp/rocky-src /tmp/rocky-build
xorriso -osirrox on -indev Rocky-9.6-x86_64-dvd.iso -extract / /tmp/rocky-src
vim /tmp/rocky-src/isolinux/isolinux.cfg
vim /tmp/rocky-src/EFI/BOOT/grub.cfg
```

Add the Kickstart boot parameter to the default boot entries, for example:

```text
inst.ks=hd:LABEL=KS_swarm01.lenmail.de:/ks.cfg
```

Then rebuild the installer ISO:

```bash
xorriso -as mkisofs \
  -o /tmp/rocky-build/rocky9.6-ks.iso \
  -V ROCKY-9-6-X86_64 \
  -J -R -T \
  -b isolinux/isolinux.bin \
  -c isolinux/boot.cat \
  -no-emul-boot \
  -boot-load-size 4 \
  -boot-info-table \
  -eltorito-alt-boot \
  -e images/efiboot.img \
  -no-emul-boot \
  /tmp/rocky-src
```

From a systems architecture standpoint, validate both boot paths before using the ISO in production:

- BIOS boot path via `isolinux`
- UEFI boot path via `grub`

If either path is missing the `inst.ks=` argument, the installer will fall back to interactive mode even though the second config CD is attached correctly.

### Integration with Ansible Vault

```yaml
# group_vars/all/vault.yml (encrypted with ansible-vault)
vault_proxmox_password: "super-secret-password"
vault_smtp_password: "email-password"

# group_vars/all/main.yml
proxmox_automatic_api_password: "{{ vault_proxmox_password }}"
proxmox_automatic_smtp_password: "{{ vault_smtp_password }}"
```

### Conditional VM Creation

```yaml
# Create VMs only in production
- hosts: all
  roles:
    - role: lenmail.proxmox_automatic
      when: environment == "production"
```

## 🔧 Troubleshooting

### Common Issues

1. **ISO Creation Fails**
   ```bash
   # Check dependencies
   proxmox_automatic_install_dependencies: true
   ```

2. **VM Creation Timeout**
   ```yaml
   # Increase timeouts
   proxmox_automatic_create_vm_retries: 5
   proxmox_automatic_create_vm_retry_delay: 10
   ```

3. **Network Issues**
   ```yaml
   # Use DHCP for testing
   proxmox_automatic_dhcp_enabled: true
   ```

### Debug Mode

```bash
# Run with increased verbosity
ansible-playbook -vvv playbook.yml
```

### Log Locations

- **Kickstart Files:** `{{ proxmox_automatic_files_dir }}/`
- **Preseed Files:** `{{ proxmox_automatic_files_dir }}/`
- **ISO Files:** `{{ proxmox_automatic_iso_dir }}/`
- **VM Console:** Proxmox VE web interface → VM → Console

## Testing

For controller-side validation without touching Proxmox:

```bash
make syntax
ansible-playbook tests/render_templates.yml
```

For full local media generation tests, including ISO creation and Debian remastering, run on a controller with `xorriso`, `isolinux`, `syslinux`, and `libarchive-tools` installed:

```bash
make media
```

For real Proxmox end-to-end validation, the repository also contains a dedicated matrix:

- [tests/proxmox_e2e.yml](/Users/dominik.lenhardt/git/ansible_roles/ansible-role-proxmox-automatic/tests/proxmox_e2e.yml)
- [tests/proxmox_e2e_cleanup.yml](/Users/dominik.lenhardt/git/ansible_roles/ansible-role-proxmox-automatic/tests/proxmox_e2e_cleanup.yml)
- [tests/proxmox_e2e_matrix.yml](/Users/dominik.lenhardt/git/ansible_roles/ansible-role-proxmox-automatic/tests/proxmox_e2e_matrix.yml)

That matrix covers Rocky online/CDROM and Debian online/CDROM with incremented VMIDs and IPs. It is intentionally kept outside CI because it provisions real VMs on a real Proxmox cluster.

The CI pipeline runs the full suite automatically on Ubuntu, including linting, render checks, Python syntax validation for the custom module, and media artifact generation.

## 📄 License

MIT

## 👥 Author Information

This role is maintained by [lenmail](https://github.com/lenmail).

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## 📞 Support

- [GitHub Issues](https://github.com/lenmail/ansible-role-proxmox-automatic/issues)
- [Ansible Galaxy](https://galaxy.ansible.com/lenmail/proxmox_automatic)
