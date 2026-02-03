# Quick Start Guide for Beginners

## What is Ansible?
Ansible is an automation tool that helps you manage servers and network devices by writing simple YAML files (called "playbooks") that describe what you want to do. Think of it like writing a recipe that Ansible follows to configure your firewalls.

## First-Time Setup

### Step 1: Install Ansible
```bash
# On Windows (using WSL) or Linux
pip install ansible

# Verify installation
ansible --version
```

### Step 2: Install Palo Alto Collection
```bash
cd c:\Users\maxwb\OneDrive\Documents\VSCODE\Ansible
ansible-galaxy collection install -r requirements.yml
```

### Step 3: Configure Your Panorama Connection

Edit [inventory/production/hosts.yml](inventory/production/hosts.yml):
```yaml
panorama-primary:
  ansible_host: YOUR_PANORAMA_IP_HERE  # Change this!
  device_group: "YOUR_DEVICE_GROUP"     # Change this!
  template_stack: "YOUR_TEMPLATE"       # Change this!
```

### Step 4: Set Up Credentials (IMPORTANT - Don't skip!)

**Option A: Environment Variables (Easiest for testing)**
```bash
export PANORAMA_USERNAME='your-username'
export PANORAMA_PASSWORD='your-password'
```

**Option B: Ansible Vault (Most secure for production)**
```bash
# Create encrypted file
# Note: panorama.yml is a file, not a directory, so vault goes in group_vars root
ansible-vault create inventory/production/group_vars/vault.yml

# Add these lines when the editor opens:
vault_panorama_username: "your-username"
vault_panorama_password: "your-password"

# Then update inventory/production/group_vars/panorama.yml to reference vault:
username: "{{ vault_panorama_username }}"
password: "{{ vault_panorama_password }}"
```

### Step 5: Customize Your Baseline Standards

Edit [inventory/production/group_vars/all.yml](inventory/production/group_vars/all.yml) to match your organization's security requirements. This file contains the default baseline configuration that applies to all Device Groups.

## Running Your First Playbook

### Test Connection (Check Mode - No Changes)
```bash
cd c:\Users\maxwb\OneDrive\Documents\VSCODE\Ansible
ansible-playbook playbooks/baseline_compliance.yml --check
```

The `--check` flag means "show me what would change, but don't actually change anything."

### Apply Baseline Configuration
```bash
ansible-playbook playbooks/baseline_compliance.yml
```

This will actually make changes to Panorama!

### Run Against Staging First
```bash
ansible-playbook playbooks/baseline_compliance.yml -i inventory/staging
```

## Understanding the Output

Ansible shows you:
- **Green (ok)**: Task ran, no changes needed
- **Yellow (changed)**: Task made changes
- **Red (failed)**: Task encountered an error
- **Purple (skipped)**: Task was skipped based on conditions

## Common Commands

```bash
# Test connectivity to Panorama
ansible panorama -i inventory/production -m ping

# Check syntax of a playbook
ansible-playbook playbooks/baseline_compliance.yml --syntax-check

# See what hosts are in your inventory
ansible-inventory -i inventory/production --list

# Run only specific tags
ansible-playbook playbooks/baseline_compliance.yml --tags ntp

# Skip specific tags
ansible-playbook playbooks/baseline_compliance.yml --skip-tags commit
```

## Troubleshooting

### "Authentication failed"
- Check your username/password in environment variables or vault
- Verify you can log into Panorama web UI with same credentials

### "Connection timeout"
- Verify `ansible_host` IP address in hosts.yml
- Check firewall rules allow HTTPS (443) to Panorama
- Ping the Panorama IP to verify network connectivity

### "Module not found"
- Run `ansible-galaxy collection install -r requirements.yml` again
- Check that the collection installed: `ansible-galaxy collection list`

### "Device group not found"
- Verify the device_group name matches exactly in Panorama
- Check spelling and capitalization

## Next Steps

1. **Understand the structure**: Read through the files to see how they connect
2. **Customize variables**: Edit group_vars/all.yml for baseline defaults
3. **Add per-DG overrides**: Create host_vars files for specific Device Groups
4. **Add security rules**: Expand the baseline_security_rules section in group_vars/all.yml
5. **Create new playbooks**: Copy baseline_compliance.yml and modify for new use cases

## Learning Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [PAN-OS Ansible Collection Docs](https://paloaltonetworks.github.io/pan-os-ansible/)
- [Ansible for Network Automation](https://docs.ansible.com/ansible/latest/network/index.html)

## File Navigation Guide

```
Start here: README.md (overview)
     ↓
Then: QUICK_START.md (this file)
     ↓
Configure: inventory/production/hosts.yml (your Panorama details)
     ↓
Customize: inventory/production/group_vars/all.yml (baseline defaults)
     ↓
Override: inventory/production/host_vars/*.yml (per-DG customization)
     ↓
Run: playbooks/baseline_compliance.yml (the actual automation)
```
