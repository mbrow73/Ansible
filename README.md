# Ansible
This repository holds ansible repository structure.
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

