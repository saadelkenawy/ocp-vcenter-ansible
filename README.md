# OCP vCenter Ansible Deployment

Ansible playbook to deploy **8 OpenShift VMs** on VMware vCenter — no template, ISO attached from datastore.

---

## VMs Created

| Name | Role | CPUs | RAM | Disk1 | Disk2 |
|---|---|---|---|---|---|
| Enawy-OCP-Master1 | Master | 8 | 16GB | 120GB | 200GB |
| Enawy-OCP-Master2 | Master | 8 | 16GB | 120GB | 200GB |
| Enawy-OCP-Master3 | Master | 8 | 16GB | 120GB | 200GB |
| Enawy-OCP-Worker1 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Worker2 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Worker3 | Worker | 16 | 32GB | 120GB | 500GB |
| Enawy-OCP-Bootstrap | Worker | 8 | 16GB | 120GB | - |
| Enawy-OCP-Bastion | rhel9 | 4 | 8GB | 120GB | - |


---

## Project Structure

```
ocp-vcenter-ansible/
├── site.yml                      # Main playbook
├── vars.yml                      # All variables (no credentials)
├── vault.yml                     # ENCRYPTED credentials (not in git)
├── vault.yml.example             # Vault template (safe to commit)
├── ansible.cfg                   # Ansible config
├── inventory.ini                 # Localhost inventory
├── .gitignore                    # Excludes vault.yml and secrets
└── tasks/
    ├── create_vm.yml             # Phase 1: Create VM + attach ISO
    ├── add_disks.yml             # Phase 2: Add 2 disks per VM
    └── set_advanced_params.yml   # Phase 3: Advanced VM parameters
```

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
