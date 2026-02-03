# Palo Alto Firewall Ansible Automation

This project contains Ansible playbooks for managing Palo Alto firewalls through Panorama.

## Project Structure

```
ansible/
├── inventory/              # Where we define our firewalls and Panorama
│   ├── production/        # Production environment
│   │   ├── hosts.yml      # List of production devices
│   │   ├── group_vars/    # Variables for groups of devices
│   │   └── host_vars/     # Variables for specific devices
│   └── staging/           # Staging/test environment
│
├── playbooks/             # The actual automation scripts
│   ├── baseline_compliance.yml      # Check/enforce baseline config
│   ├── nat_onboarding_EXAMPLE.yml   # (Example) Add NAT rules
│   └── url_management_EXAMPLE.yml   # (Example) Manage URL categories
│
├── roles/                 # Reusable automation components
│   ├── baseline_config/   # Baseline configuration role
│   └── panorama_common/   # Common Panorama tasks
│
├── templates/             # Jinja2 templates (currently empty - for future use)
├── ansible.cfg            # Ansible configuration
└── requirements.yml       # Required Ansible collections
```

## Getting Started

### 1. Install Required Collections
```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Your Inventory
Edit `inventory/production/hosts.yml` with your Panorama details.

Edit `inventory/production/group_vars/panorama.yml` with credentials.

### 3. Customize Baseline Standards
Edit `inventory/production/group_vars/all.yml` with your baseline security requirements.

### 3. Run Baseline Compliance Check
```bash
# Check only (no changes)
ansible-playbook playbooks/baseline_compliance.yml -i inventory/production --check

# Apply baseline configuration
ansible-playbook playbooks/baseline_compliance.yml -i inventory/production
```

## Managing Multiple Device Groups

This project supports managing **multiple Panoramas** with **multiple Device Groups (DGs)**, each with **different baseline configurations**:

- **One playbook** works for all DGs
- **host_vars/** defines DG-specific baselines
- **Variable precedence** allows flexible overrides

See [MULTI_DG_GUIDE.md](MULTI_DG_GUIDE.md) for detailed examples of managing multiple DGs.

Example commands:
```bash
# Run against all DGs
ansible-playbook playbooks/baseline_compliance.yml

# Run against specific DG
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1

# Run against all internet-facing DGs
ansible-playbook playbooks/baseline_compliance.yml --limit internet_facing
```

## Key Concepts for Ansible Beginners

- **Playbook**: A YAML file that describes automation steps (like a recipe)
- **Role**: A reusable collection of tasks, variables, and templates
- **Inventory**: List of devices to manage (your Panorama/firewalls)
- **Variables**: Configuration values (like IP addresses, passwords, settings)
- **Tasks**: Individual automation steps (like "create security rule")
- **Templates**: Files with placeholders that get filled in with variables

**New to Ansible?** Read these in order:
1. [QUICK_START.md](QUICK_START.md) - Installation and setup
2. [ANSIBLE_CONCEPTS.md](ANSIBLE_CONCEPTS.md) - How Ansible works
3. [MULTI_DG_GUIDE.md](MULTI_DG_GUIDE.md) - Managing multiple device groups
4. [VARIABLE_PRECEDENCE_EXAMPLE.md](VARIABLE_PRECEDENCE_EXAMPLE.md) - How variables override each other

## Important Notes

### Baseline Implementation Status
The baseline configuration role is **partially implemented**. Currently implemented:
- ✅ Security profile groups
- ✅ Zone protection profiles
- ✅ NTP configuration
- ✅ Basic DNS configuration

**Not yet implemented** (defined in vars but not enforced):
- ⏳ Logging configuration (`baseline_logging`)
- ⏳ Update schedules (`baseline_updates`)
- ⏳ Password policy (`baseline_password_policy`)
- ⏳ SNMP configuration (`baseline_snmp`)
- ⏳ Management access restrictions (`baseline_management.permitted_ips`, `services`)
- ⏳ Specific security rules creation

These variables are placeholders for future expansion. You can customize the role in `roles/baseline_config/tasks/main.yml` to implement additional baseline checks based on your requirements.

### Templates Directory
The `templates/` directory is currently empty and reserved for future use. It's intended for Jinja2 configuration templates that can be rendered with variables and deployed to Panorama.

## Security Notes

- Never commit passwords to git!
- Use Ansible Vault to encrypt sensitive variables
- Consider using environment variables for credentials

## Next Steps

1. Fill in your Panorama details in inventory files
2. Customize baseline standards in `inventory/production/group_vars/all.yml`
3. Create per-DG overrides in `inventory/production/host_vars/` as needed
4. Test with `--check` mode first
5. Expand with additional playbooks for NAT, URL management, etc.
