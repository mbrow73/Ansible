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
│   ├── panorama_common/   # Common Panorama tasks
│   └── compliance_audit/  # Read-only compliance checking role
│
├── templates/             # Jinja2 templates (compliance reports)
├── reports/               # Generated compliance reports (gitignored)
├── .ansible-lint          # Ansible-lint configuration
├── ansible.cfg            # Ansible configuration
└── requirements.yml       # Required Ansible collections
```


## Compliance Audit

Run a read-only compliance audit to check current Panorama configuration against baselines without making any changes:

```bash
ansible-playbook playbooks/compliance_audit.yml -i inventory/production
```

### Report Output

Reports are generated as JSON files in the `reports/` directory with the naming convention:
```
reports/<hostname>_compliance_<timestamp>.json
```

Each report includes:
- **Metadata**: timestamp, host, device group, template stack, environment info
- **Summary**: total checks, pass/fail/unknown counts, compliance percentage
- **Findings**: detailed per-check results with expected vs actual values and severity

### Available Tags

Run selective audit checks using tags:

```bash
# Run only security profile checks
ansible-playbook playbooks/compliance_audit.yml -i inventory/production --tags "audit,security_profiles"

# Run only NTP and DNS checks
ansible-playbook playbooks/compliance_audit.yml -i inventory/production --tags "audit,ntp,dns"

# Generate report only (requires prior audit data)
ansible-playbook playbooks/compliance_audit.yml -i inventory/production --tags "report"
```

Available tags: `audit`, `security_profiles`, `zone_protection`, `ntp`, `dns`, `management`, `security_rules`, `session`, `report`

## Linting

This project uses `ansible-lint` for code quality. Configuration is in `.ansible-lint`.

```bash
# Install ansible-lint
pip install ansible-lint

# Run linting
ansible-lint
```
