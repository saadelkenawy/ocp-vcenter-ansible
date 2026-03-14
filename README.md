# OCP vCenter Ansible Deployment

Ansible playbook to deploy **8 OpenShift VMs** on VMware vCenter — no template, ISO attached from datastore.

---

## VMs Created

| Name | Role | CPUs | RAM | Disk1 | Disk2 |
|---|---|---|---|---|---|----|----|
| Enawy-OCP-Master1 | Master | 8 | 16GB | 120GB |  0  
| Enawy-OCP-Master2 | Master | 8 | 16GB | 120GB | 0|
| Enawy-OCP-Master3 | Master | 8 | 16GB | 120GB | 0 
| Enawy-OCP-Worker1 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Worker2 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Worker3 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-ODF-1  | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-ODF-2 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-ODF-2 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Bootstrap | Worker | 8 | 16GB | 120GB | - |
| Enawy-OCP-Bastion | rhel9 | 4 | 8GB | 120GB | - |


---

## Project Structure

.
├── ansible.cfg
├── bastion_ip.txt
├── inventory.ini
├── mac_addresses.txt
├── ocp-tasks
│   ├── bootstrap.ign
│   ├── generate_install_config.yml
│   ├── generate_manifests_ignition.yml
│   ├── install-config.yaml
│   ├── install-config.yaml.j2
│   ├── master.ign
│   └── worker.ign
├── ocp-vms
│   └── poweron_bootstrap.yml
├── README.md
├── should-run-this-on-bastion-one-reboot.txt
├── site.yml
├── ssh-gen-ansible-copy-to-bastion.txt
├── tasks
│   ├── add_disks.yml
│   ├── configure_bastion.yml
│   ├── configure_dhcp.yml
│   ├── configure_dns.yml
│   ├── configure_haproxy_keepalived.yml
│   ├── configure_ntp.yml
│   ├── create_vm.yml
│   ├── generate_ssh_key.yml
│   ├── set_advanced_params.yml
│   └── update_dns_zones.yml
├── vars.yml
├── vault.yml
└── vault.yml.example

---

## Requirements

- Ansible Core 2.12+
- Python `pyVmomi`
- Collection `community.vmware`

```bash
pip3 install pyVmomi
ansible-galaxy collection install community.vmware
```

---

## Setup

**1. Edit `vars.yml`** with your vCenter details:
```yaml
vcenter_hostname: "10.20.20.200"
vcenter_datacenter: "DC_KSA"
vcenter_datastore: "DATASTORE_4_HDD"
iso_path: "ISO/rhcos-4.14.iso"
```

**2. Create encrypted vault:**
```bash
ansible-vault create vault.yml  
Added in .gitignore
```
```yaml
vcenter_username: "administrator@vsphere.local"
vcenter_password: "YourPassword"
vcenter_validate_certs: false
```

**3. Run:**
```bash
ansible-playbook site.yml --ask-vault-pass
```

---

## Security

- `vault.yml` is **excluded from git** via `.gitignore`
- All credentials stored in Ansible Vault (AES256 encrypted)
- See `vault.yml.example` for vault structure
