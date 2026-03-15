# OCP vCenter Ansible Deployment

Fully automated Ansible playbook to deploy an **OpenShift Container Platform (OCP)** cluster on **VMware vCenter** — from bare VMs to a running bootstrap, with no template required. The playbook provisions all infrastructure VMs, configures the Bastion server with all required services (DNS, DHCP, NTP, HAProxy, Keepalived), generates OCP ignition files, and powers on the Bootstrap VM to start cluster installation.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
   - [Install Ansible](#1-install-ansible)
   - [Install Python Dependencies](#2-install-python-dependencies)
   - [Install Ansible Collections](#3-install-ansible-collections)
4. [Project Structure](#project-structure)
5. [Configuration](#configuration)
   - [vars.yml](#varsyml)
   - [vault.yml (Encrypted Credentials)](#vaultyml-encrypted-credentials)
   - [ansible.cfg](#ansiblecfg)
6. [VM Specifications](#vm-specifications)
7. [Playbook Execution Sequence](#playbook-execution-sequence)
8. [Running the Playbook](#running-the-playbook)
9. [Interactive Prompts Reference](#interactive-prompts-reference)
10. [Post-Deployment Steps](#post-deployment-steps)
11. [Security](#security)
12. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
Ansible Controller (localhost)
        │
        │  API calls (pyVmomi)
        ▼
VMware vCenter ──────────────────────────────────────────────────────┐
        │                                                             │
        │  Creates VMs                                               │
        ▼                                                             │
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
  │  Master 1-3 │  │  Worker 1-3 │  │   ODF 1-3   │  (RHCOS ISO)  │
  └─────────────┘  └─────────────┘  └─────────────┘                │
  ┌─────────────┐  ┌──────────────────────────────────────────────┐ │
  │  Bootstrap  │  │               Bastion (RHEL9)                │ │
  └─────────────┘  │  DNS · DHCP · NTP · HAProxy · Keepalived    │ │
                   └──────────────────────────────────────────────┘ │
                                                                      │
        Ansible Controller ──── SSH ──────► Bastion (Phase 6+)      │
                                                                      └
```

---

## Prerequisites

| Requirement | Minimum Version |
|---|---|
| Ansible Core | 2.12+ |
| Python | 3.8+ |
| pip / pip3 | Any recent version |
| OS (Controller) | RHEL 8/9, CentOS, Ubuntu 20.04+ |
| vCenter | 7.0+ |
| ESXi Host | 7.0+ |
| RHCOS ISO in Datastore | 4.x (matching your OCP version) |
| RHEL 9 ISO in Datastore | 9.x (for Bastion VM) |

---

## Installation

### 1. Install Ansible

**RHEL / CentOS:**
```bash
sudo dnf install -y ansible-core
```

**Ubuntu / Debian:**
```bash
sudo apt update && sudo apt install -y ansible
```

**pip (any platform):**
```bash
pip3 install ansible
```

Verify:
```bash
ansible --version
```

---

### 2. Install Python Dependencies

The `community.vmware` collection requires the `pyVmomi` library to communicate with vCenter via its Python API, and `requests` for HTTP interactions.

```bash
pip3 install pyVmomi
pip3 install requests
```

> **Note:** If you are running on RHEL and prefer system packages:
> ```bash
> sudo dnf install -y python3-pyVmomi
> ```

---

### 3. Install Ansible Collections

The playbook uses two Ansible collections that must be installed before running:

```bash
# VMware vCenter automation modules
ansible-galaxy collection install community.vmware

# POSIX modules (sysctl, SELinux, etc.)
ansible-galaxy collection install ansible.posix
```

Verify installation:
```bash
ansible-galaxy collection list | grep -E "vmware|posix"
```

---

## Project Structure

```
ocp-vcenter-ansible/
├── ansible.cfg                        # Ansible configuration
├── inventory.ini                      # Inventory (localhost only)
├── site.yml                           # Main playbook (entry point)
├── vars.yml                           # VM definitions and vCenter settings
├── vault.yml                          # Encrypted credentials (git-ignored)
├── vault.yml.example                  # Vault structure reference
├── bastion_ip.txt                     # Auto-generated: Bastion IP/network
├── mac_addresses.txt                  # Auto-generated: VM MAC addresses
│
├── tasks/                             # Infrastructure task files
│   ├── create_vm.yml                  # Create a single VM in vCenter
│   ├── add_disks.yml                  # Add/verify disks per VM
│   ├── set_advanced_params.yml        # Apply VMX advanced parameters
│   ├── configure_bastion.yml          # Bastion OS prep (repos, packages)
│   ├── configure_dns.yml              # BIND DNS server setup
│   ├── configure_ntp.yml              # Chrony NTP server setup
│   ├── configure_dhcp.yml             # DHCP static reservations
│   ├── update_dns_zones.yml           # Add A/PTR records from DHCP
│   ├── configure_haproxy_keepalived.yml  # HAProxy LB + Keepalived VIPs
│   └── generate_ssh_key.yml           # SSH key generation on Bastion
│
├── ocp-tasks/                         # OCP-specific tasks and templates
│   ├── install-config.yaml.j2         # Jinja2 template for install-config
│   ├── install-config.yaml            # Auto-generated (backed up here)
│   ├── generate_install_config.yml    # Generate OCP install-config.yaml
│   ├── generate_manifests_ignition.yml # Download OCP tools + generate ignition
│   ├── bootstrap.ign                  # Auto-generated ignition (backed up)
│   ├── master.ign                     # Auto-generated ignition (backed up)
│   └── worker.ign                     # Auto-generated ignition (backed up)
│
└── ocp-vms/
    └── poweron_bootstrap.yml          # Power on Bootstrap VM (Beta Version)
```

---

## Configuration

### vars.yml

Edit this file before running the playbook:

```yaml
# vCenter connection
vcenter_hostname: "10.20.20.200"  (Change Per Your ENv)
vcenter_datacenter: "GTS_KSA"      (Change Per Your ENv)
vcenter_datastore: "DATASTORE_4_HDD" (Change Per Your ENv)
esxi_hostname: "10.20.20.21"   (Change Per Your ENv)
vcenter_folder: "/GTS_KSA/vm/TEST-OCP"  (Change Per Your ENv)

# ISO paths in the datastore
iso_path: "ISOs/rhcos-4.20.12-x86_64-live-iso.x86_64.iso"
iso_path_rhel9: "ISOs/rhel-9.6-x86_64-dvd.iso"

# VM name prefix
vm_name_prefix: "Enawy-OCP-Test" (Change Per Your ENv)

# Local user running Ansible (used for file paths)
local_user: "<your-user>" (Change Per Your User)
```

VM definitions are also in `vars.yml` under the `vms:` list. Each entry defines a VM role, CPU, RAM, and disk sizes.

---

### vault.yml (Encrypted Credentials)

**Step 1 — Create your vault password file:**
```bash
echo "YourVaultPassword" > /home/<your-user>/ocp-vcenter-ansible/.vault_pass
chmod 600 /home/<your-user>/ocp-vcenter-ansible/.vault_pass
```

**Step 2 — Create the encrypted vault:**
```bash
ansible-vault create vault.yml
```

Add the following content:
```yaml
vcenter_username: "administrator@vsphere.local"
vcenter_password: "YourVCenterPassword"
vcenter_validate_certs: false
bastion_ssh_password: "YourBastionRootPassword"
```

> The field `bastion_ssh_public_key` is automatically added to vault.yml by the playbook after generating the SSH key on the Bastion.

To edit the vault later:
```bash
ansible-vault edit vault.yml
```

To encrypt an existing plain-text vault file:
```bash
ansible-vault encrypt vault.yml
```

---

### ansible.cfg

The configuration file is pre-set for this project:

```ini
[defaults]
inventory           = inventory.ini
host_key_checking   = False
retry_files_enabled = False
stdout_callback     = debug
collections_path    = ~/.ansible/collections
vault_password_file = /home/<your-user>/ocp-vcenter-ansible/.vault_pass    # change per your name

[ssh_connection]
pipelining = True
```

> The `vault_password_file` path must match where you saved your `.vault_pass` file.

---

## VM Specifications

All VMs are created from ISO — no templates required. VMs are created in the vCenter folder `PRO-OCP` under the configured datacenter.

| VM Name | Role | vCPU | RAM | Disk 1 | Disk 2 | OS |
|---|---|---|---|---|---|---|
| `<prefix>-Master1` | Control Plane | 8 | 16 GB | 200 GB | — | RHCOS |
| `<prefix>-Master2` | Control Plane | 8 | 16 GB | 200 GB | — | RHCOS |
| `<prefix>-Master3` | Control Plane | 8 | 16 GB | 200 GB | — | RHCOS |
| `<prefix>-Worker1` | Compute | 8 | 16 GB | 120 GB | 300 GB | RHCOS |
| `<prefix>-Worker2` | Compute | 8 | 16 GB | 120 GB | 300 GB | RHCOS |
| `<prefix>-Worker3` | Compute | 8 | 16 GB | 120 GB | 300 GB | RHCOS |
| `<prefix>-ODF1` | ODF Storage | 8 | 24 GB | 150 GB | 500 GB | RHCOS |
| `<prefix>-ODF2` | ODF Storage | 8 | 24 GB | 150 GB | 500 GB | RHCOS |
| `<prefix>-ODF3` | ODF Storage | 8 | 24 GB | 150 GB | 500 GB | RHCOS |
| `<prefix>-bootstrap1` | Bootstrap | 8 | 16 GB | 120 GB | — | RHCOS |
| `<prefix>-Bastion1` | Services | 4 | 8 GB | 120 GB | — | RHEL 9 |

**Total: 11 VMs**

All VMs use:
- Network adapter: `vmxnet3`
- Disk type: `thin provisioned`
- Firmware: `EFI` (Secure Boot disabled)
- Hardware version: `20`  #VMware Version : VMware ESXi, 8.0.3, 24280767

---

## Playbook Execution Sequence

The playbook runs in two plays: the first executes on **localhost** (Ansible controller communicating with vCenter), and the second SSH-connects to the **Bastion** VM once it is up.

---

### Play 1 — vCenter Provisioning (localhost)

#### Phase 0 — Create VM Folder
Creates the `Pro-OCP` VM folder in the vCenter datacenter if it does not already exist.

#### Phase 1 — Create All VMs
Iterates over the `vms` list in `vars.yml` and creates each VM using `community.vmware.vmware_guest`. For each VM, the task:
- Sets CPU, RAM, hardware version, and guest OS type
- Attaches the appropriate ISO (RHCOS or RHEL9) from the datastore as a CD-ROM
- Configures the `vmxnet3` network adapter on the target network
- Provisions Disk 1 (and Disk 2 if size > 0) as thin-provisioned VMDK

#### Phase 2 — Add / Verify Disks
Runs `community.vmware.vmware_guest_disk` per VM to ensure both disks are attached at the correct SCSI controller positions. This task is **idempotent** — safe to re-run without duplicating disks.

#### Phase 3 — Apply Advanced VM Parameters
Sets VMX-level `extraConfig` entries on each VM required for OpenShift:

| Parameter | Value | Purpose |
|---|---|---|
| `disk.EnableUUID` | TRUE | Required for OCP persistent volumes (PVs) |
| `guestinfo.ignition.config.data.encoding` | base64 | CoreOS ignition data encoding |
| `tools.syncTime` | FALSE | OCP manages its own time via Chrony |
| `mem.hotadd` | FALSE | Disabled for stability |
| `cpu.hotadd` | FALSE | Disabled for stability |
| `hpet0.present` | FALSE | Remove HPET timer (not needed) |
| `stealclock.enable` | TRUE | Accurate CPU steal time reporting |

#### Phase 4 — Collect MAC Addresses
Queries `vmware_guest_info` for each VM and collects the MAC address of the primary NIC (`hw_eth0`). Saves the MAC-to-VM mapping to `mac_addresses.txt` on the Ansible controller. This file is later used by the DHCP task to create static reservations.

#### Phase 5 — Power On Bastion and Establish Connectivity
1. Checks if the Bastion VM is already powered on.
2. Prompts the operator to confirm powering on the Bastion.
3. Powers on the Bastion VM via `vmware_guest_powerstate` and waits up to 100 seconds for it to reach `poweredOn` state.
4. **Pauses and prompts for the Bastion IP address** (entered after completing the RHEL9 OS installation via the vCenter console).
5. Saves the Bastion IP and derived `/24` network to `bastion_ip.txt`.
6. Waits for SSH port 22 to become available on the Bastion (timeout: 6000 s).
7. Clears any old SSH host key and scans the new one into `~/.ssh/known_hosts`.
8. Adds the Bastion to the in-memory Ansible inventory for Play 2.

---

### Play 2 — Bastion Configuration (SSH to Bastion)

The second play connects to the Bastion over SSH using the credentials from `vault.yml`.

#### configure_bastion.yml
Prepares the RHEL9 Bastion operating system:
1. Stops and disables `firewalld` permanently.
2. Sets SELinux to `permissive` mode.
3. Adds the internal RHEL golden image host to `/etc/hosts`.
4. Creates a local YUM/DNF repository file pointing to an internal mirror (BaseOS, AppStream, CodeReady, HA).  # if you don't Have a synced repo with Redhat | use redhat  Subscription before proceed 
5. Runs `dnf update` to fully patch the system.
6. Installs all required service packages:
   - `chrony` — NTP server
   - `bind` + `bind-utils` — DNS (BIND/named)
   - `dhcp-server` — DHCP server
   - `haproxy` — Load balancer
   - `keepalived` — VRRP / Virtual IP management
   - `ipcalc` — IP address calculation utility
   - `httpd` — Apache web server (serves ignition files)

#### configure_dns.yml
Installs and configures a BIND DNS server:
1. Prompts for the network CIDR (e.g., `192.168.110.0/24`).
2. Prompts for two upstream DNS forwarder IPs.
3. Detects the domain name from the Bastion's FQDN (`hostname -f`).
4. Creates the log and zone directories under `/var/log/named` and `/var/named/zones`.
5. Generates the `rndc.key` for BIND administration.
6. Writes `/etc/named.conf` with:
   - Listener on Bastion IP and `127.0.0.1`
   - Forward-only to the two upstream forwarders
   - Recursion restricted to the local subnet
7. Creates the domain zone config file at `/etc/named/<domain>.conf`.
8. Creates a forward zone file with a placeholder bastion `A` record.
9. Creates a reverse zone file with placeholder `PTR` records.
10. Validates config with `named-checkconf` and starts/enables the `named` service.
11. Tests DNS resolution (forward, reverse, and external).

#### configure_ntp.yml
Configures Chrony as both an NTP client and server:
1. Prompts for one or more NTP server addresses (space-separated).
2. Prompts for the list of client networks allowed to use this bastion as an NTP source.
3. Writes `/etc/chrony.conf` with all upstream servers and `allow` directives.
4. Restarts and enables `chronyd`.
5. Waits 10 seconds and verifies sync using `chronyc tracking` and `chronyc sources`. Fails if not synchronized.

#### configure_dhcp.yml
Configures a DHCP server with static IP reservations for all OCP VMs:
1. Loads the Bastion IP from `bastion_ip.txt`.
2. Reads the MAC addresses from `mac_addresses.txt` (collected in Phase 4).
3. Detects the default gateway and broadcast address from the Bastion's routing table.
4. Prompts for the DHCP domain name (e.g., `ocp.gts`).
5. Generates static DHCP host entries for all VMs except Bastion, assigning sequential IPs starting from `.50`.
6. Writes `/etc/dhcp/dhcpd.conf` with the subnet block (dynamic range `.100`–`.200`) and all static reservations.
7. Restarts and enables `dhcpd`.

#### update_dns_zones.yml
Populates the DNS zone files with A and PTR records for all OCP nodes:
1. Reads the static IP assignments from `/etc/dhcp/dhcpd.conf`.
2. Inserts an `A` record for each VM into the forward zone file.
3. Inserts a `PTR` record for each VM into the reverse zone file.
4. Increments the SOA serial number in both zone files.
5. Validates zone syntax with `named-checkzone` and restarts `named`.
6. Tests DNS `A` record resolution for each host.

#### configure_haproxy_keepalived.yml
Sets up virtual IPs (VIPs) and load balancing for the OCP API and application ingress:
1. Detects the Bastion's primary network interface.
2. Derives VIP addresses automatically from the Bastion IP (`.230` for API, `.235` for Ingress).
3. Prompts for the OCP cluster name.
4. Parses the DHCP configuration to extract master, worker, and bootstrap IPs.
5. Writes `/etc/keepalived/keepalived.conf` with two VRRP instances (API VIP and Ingress VIP).
6. Writes `/etc/haproxy/haproxy.cfg` with four frontends/backends:
   - **API** — port `6443` → Bootstrap + Masters
   - **MCS** — port `22623` → Bootstrap + Masters (Machine Config Server)
   - **Ingress HTTP** — port `80` → Workers
   - **Ingress HTTPS** — port `443` → Workers
7. Sets `net.ipv4.ip_nonlocal_bind=1` via sysctl so HAProxy can bind to VIPs before they are locally assigned.
8. Starts/enables `keepalived` and `haproxy`.
9. Waits 15 seconds and verifies both VIPs (`.230` and `.235`) are assigned to the interface.
10. Updates the DNS forward and reverse zones with `api.<cluster>`, `api-int.<cluster>`, and `*.apps.<cluster>` records pointing to the VIPs.

#### generate_ssh_key.yml
Generates the SSH key pair that will be embedded in OCP ignition files:
1. Checks if `/root/.ssh/id_ed25519` already exists and optionally overwrites.
2. Generates an `ed25519` key pair on the Bastion under `/root/.ssh/`.
3. Sets correct file permissions (`0700` on dir, `0640` on private key, `0644` on public key).
4. Reads the public key content.
5. Temporarily decrypts `vault.yml` on the Ansible controller, injects the `bastion_ssh_public_key` field, and re-encrypts the vault — making the key available for the `install-config.yaml` template.

#### generate_install_config.yml (ocp-tasks)
Generates the OCP cluster installation configuration:
1. Skips entirely if ignition files are already present (idempotent).
2. Loads the Bastion IP and network from `bastion_ip.txt`.
3. Reads the `bastion_ssh_public_key` from the vault.
4. Prompts for the OCP **base domain** (e.g., `bank.gts`).
5. Prompts for the Red Hat **pull secret** (from console.redhat.com).
6. Creates the `/opt/ocp-install/` directory on the Bastion.
7. Renders `install-config.yaml.j2` into `/opt/ocp-install/install-config.yaml` using all collected facts.
8. Fetches a backup of the generated file back to the Ansible controller at `ocp-tasks/install-config.yaml`.

#### generate_manifests_ignition.yml (ocp-tasks)
Downloads OCP tools and generates ignition files:
1. Reconfigures Apache `httpd` to listen on port `8888` (to avoid conflict with Ingress port 80) and starts it.
2. Verifies `install-config.yaml` exists.
3. Downloads `openshift-install` binary from the OpenShift mirror (stable-4.20) if not present.
4. Downloads `oc` and `kubectl` client binaries if not present.
5. Cleans up downloaded tarballs.
6. Skips manifest generation if the `/opt/ocp-install/auth` directory already exists.
7. Runs `openshift-install create manifests --dir /opt/ocp-install/` to generate cluster manifests.
8. Sets `mastersSchedulable: false` in `cluster-scheduler-02-config.yml` so workloads do not land on control plane nodes.
9. Runs `openshift-install create ignition-configs --dir /opt/ocp-install/` to produce `bootstrap.ign`, `master.ign`, and `worker.ign`.
10. Copies ignition files to `/var/www/html/ocp/` for HTTP serving to RHCOS nodes during boot.
11. Fetches all three ignition files back to `ocp-tasks/` on the Ansible controller as a backup.

#### poweron_bootstrap.yml (ocp-vms)
Starts the Bootstrap VM to begin the OCP cluster bootstrapping process:
1. Queries the Bootstrap VM's current power state.
2. Powers it on if not already running.
3. Waits (retrying up to 10 times with 10-second delays) until the VM reaches `poweredOn` state.

#### Phase 7 — Deployment Summary
Prints a formatted summary of all VMs created with their CPU, RAM, and disk configuration.

---

## Running the Playbook

**Prerequisite checklist before running:**

- [ ] `vars.yml` updated with your vCenter hostname, datacenter, datastore, ESXi host, and ISO paths
- [ ] `vault.yml` created and encrypted with vCenter and Bastion credentials
- [ ] `.vault_pass` file created at the path configured in `ansible.cfg`
- [ ] RHCOS ISO uploaded to the vCenter datastore at the path set in `vars.yml`
- [ ] RHEL 9 ISO uploaded to the vCenter datastore at the path set in `vars.yml`
- [ ] Ansible and all collections installed (see [Installation](#installation))

**Run the playbook:**

```bash
ansible-playbook site.yml
```

If you did not configure a vault password file and prefer to be prompted:

```bash
ansible-playbook site.yml --ask-vault-pass
```

To run with increased verbosity for debugging:

```bash
ansible-playbook site.yml -v
```

---

## Interactive Prompts Reference

The playbook pauses at several points to collect information. Below is a reference of every prompt in execution order:

| # | Prompt | Example Input | When |
|---|---|---|---|
| 1 | `Power on Bastion VM now? (yes/no)` | `yes` | Phase 5 — Bastion is off |
| 2 | `Enter Bastion IP address` | `192.168.110.10` | After RHEL9 OS install on Bastion console |
| 3 | `Enter bastion network in CIDR format` | `192.168.110.0/24` | DNS configuration |
| 4 | `Enter DNS forwarder 1 IP` | `8.8.8.8` | DNS configuration |
| 5 | `Enter DNS forwarder 2 IP` | `8.8.4.4` | DNS configuration |
| 6 | `Enter NTP servers space-separated` | `192.168.200.100 0.africa.pool.ntp.org` | NTP configuration |
| 7 | `Enter networks allowed to use this NTP server` | `192.168.110.0/24` | NTP configuration |
| 8 | `Enter domain name (e.g. ocp.gts)` | `ocp.gts` | DHCP configuration |
| 9 | `Enter OCP cluster name (e.g. ocp)` | `ocp` | HAProxy / Keepalived |
| 10 | `SSH key already exists. Overwrite? (yes/no)` | `yes` or `no` | SSH key generation (if key exists) |
| 11 | `Enter base domain (e.g. bank.gts)` | `bank.gts` | OCP install-config |
| 12 | `Paste your pull secret` | `{"auths":{...}}` | OCP install-config |

> **Tip:** Have all values ready before running the playbook to avoid delays during interactive pauses.

---

## Post-Deployment Steps

Once the playbook completes:

1. **Power on Master VMs** manually from vCenter (or add an automation task).
   - Masters will boot from the RHCOS ISO and fetch `master.ign` from `http://<bastion-ip>:8888/ocp/master.ign`.

2. **Monitor Bootstrap progress** from the Bastion:
   ```bash
   openshift-install wait-for bootstrap-complete --dir /opt/ocp-install/ --log-level=info
   ```

3. **Approve pending CSRs** once Masters have joined:
   ```bash
   export KUBECONFIG=/opt/ocp-install/auth/kubeconfig
   oc get csr
   oc get csr -o name | xargs oc adm certificate approve
   ```

4. **Power on Worker and ODF VMs** from vCenter — they will fetch `worker.ign` automatically.

5. **Complete cluster installation:**
   ```bash
   openshift-install wait-for install-complete --dir /opt/ocp-install/ --log-level=info
   ```

6. **Destroy Bootstrap VM** once installation is complete (Bootstrap is no longer needed):
   ```bash
   # Remove from HAProxy by editing /etc/haproxy/haproxy.cfg on Bastion
   # Power off and delete Bootstrap VM from vCenter
   ```

---

## Security

| Item | Detail |
|---|---|
| `vault.yml` | AES-256 encrypted via `ansible-vault`. Excluded from git via `.gitignore` |
| `.vault_pass` | Store securely. Never commit to git |
| `vcenter_validate_certs` | Set to `true` in production with a valid TLS certificate |
| Bastion credentials | Stored only in the encrypted vault |
| SSH key | Generated on Bastion at runtime; public key injected into vault and ignition files |

Reference `vault.yml.example` for the expected vault structure without real credentials.

---

## Troubleshooting

**`pyVmomi` import error:**
```bash
pip3 install --upgrade pyVmomi requests
```

**Collection not found:**
```bash
ansible-galaxy collection install community.vmware --force
ansible-galaxy collection install ansible.posix --force
```

**SSH connection to Bastion fails:**
- Verify the Bastion IP entered at the prompt is correct.
- Confirm RHEL9 OS installation is fully complete and SSH is running.
- Check the `bastion_ssh_password` in `vault.yml` matches the root password set during OS install.

**DHCP not assigning IPs to VMs:**
- Verify MAC addresses in `mac_addresses.txt` match the VMs.
- Check `dhcpd` service: `systemctl status dhcpd`
- Ensure no other DHCP server is active on the same subnet.

**HAProxy fails to start:**
- Confirm `net.ipv4.ip_nonlocal_bind=1` is set: `sysctl net.ipv4.ip_nonlocal_bind`
- Check that Keepalived has assigned the VIPs before HAProxy starts: `ip addr show`

**`openshift-install` download fails (no internet on Bastion):**
- Download the binary on a machine with internet access, then transfer it to `/usr/local/bin/openshift-install` on the Bastion manually before running the playbook.

**Vault decryption error:**
```bash
# Verify your vault password file contains the correct password
cat /home/<your-user>/ocp-vcenter-ansible/.vault_pass

# Test decryption manually
ansible-vault view vault.yml
```
