# Ansible Concepts Explained Simply

## How Everything Connects Together

```
┌─────────────────────────────────────────────────────────────┐
│  YOU run a command:                                         │
│  ansible-playbook playbooks/baseline_compliance.yml         │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  ANSIBLE reads configuration from:                          │
│  • ansible.cfg (how to behave)                             │
│  • inventory/production/hosts.yml (what devices to manage) │
│  • group_vars/ (variables for those devices)               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  PLAYBOOK (baseline_compliance.yml) does:                   │
│  1. Variables auto-loaded from inventory (group_vars/all.yml) │
│  2. Runs tasks in order (like following a recipe)          │
│  3. Calls the "baseline_config" role                       │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  ROLE (roles/baseline_config/) executes:                    │
│  • Reads tasks/main.yml                                     │
│  • Each task calls a module (panos_security_rule, etc.)    │
│  • Modules connect to Panorama API                         │
│  • Makes the actual changes on Panorama                    │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│  PANORAMA receives API calls and:                           │
│  • Creates/updates security profiles                        │
│  • Configures zone protection                              │
│  • Sets NTP, DNS, etc.                                     │
│  • Commits changes (if commit_changes = true)              │
└─────────────────────────────────────────────────────────────┘
```

## Key Files Explained

### 1. Inventory Files (Who to manage)
**Location**: `inventory/production/hosts.yml`

Think of this as your phonebook - it lists all the devices Ansible will manage.

```yaml
panorama-primary:
  ansible_host: 192.168.1.10  # IP address to connect to
  device_group: "DG-Production"  # Which device group in Panorama
```

### 2. Variables (Configuration values)
**Locations**:
- `group_vars/` - Variables for groups of devices
- `vars/` - Variables for specific purposes

Variables are like fill-in-the-blank values. Instead of hardcoding "192.168.1.10" everywhere, you use `{{ ansible_host }}`.

### 3. Playbooks (What to do)
**Location**: `playbooks/`

A playbook is like a recipe. It says:
1. Do this first
2. Then do this
3. Finally do this

```yaml
- name: Configure firewalls
  tasks:
    - name: Set NTP
    - name: Create security rule
    - name: Commit
```

### 4. Roles (Reusable components)
**Location**: `roles/`

A role bundles together:
- Tasks (steps to perform)
- Variables (default values)
- Templates (config file templates)

Instead of copying the same tasks into every playbook, you create a role once and reuse it.

### 5. Modules (The actual work)
Modules are pre-built tools that do specific things:
- `panos_security_rule` - Creates security rules
- `panos_nat_rule2` - Creates NAT rules
- `panos_commit_panorama` - Commits changes

You don't write modules - they come from the `paloaltonetworks.panos` collection.

## Variable Precedence (What overrides what)

From lowest to highest priority:
1. Role defaults (`roles/baseline_config/defaults/main.yml`)
2. Group vars all (`inventory/production/group_vars/all.yml`) - Baseline defaults
3. Group vars specific (`inventory/production/group_vars/panorama.yml`, `internet_facing.yml`)
4. Host vars (`inventory/production/host_vars/pano1-dg-*.yml`) - Per-DG overrides
5. Extra vars (command line: `-e "variable=value"`) - Emergency overrides

**Later ones override earlier ones.**

✅ **This means**: Baseline defaults in `group_vars/all.yml` can be overridden by specific group settings, which can be overridden by per-DG `host_vars`. This is the correct structure for multi-DG management!

## Common Workflow

### For Baseline Compliance:
1. Edit `inventory/production/group_vars/all.yml` with your baseline standards
2. Edit `inventory/production/hosts.yml` with your Panorama IP
3. Test: `ansible-playbook playbooks/baseline_compliance.yml --check`
4. Apply: `ansible-playbook playbooks/baseline_compliance.yml`

### For Adding NAT Rules (future):
1. Create `vars/nat_rules.yml` with your NAT rules
2. Customize `playbooks/nat_onboarding_EXAMPLE.yml`
3. Test in staging first: `ansible-playbook playbooks/nat_onboarding.yml -i inventory/staging --check`
4. Apply to production: `ansible-playbook playbooks/nat_onboarding.yml -i inventory/production`

## Idempotency - A Key Concept

**Idempotent** means you can run the same playbook multiple times and it will:
- Only make changes if needed
- Not break if run twice
- Always result in the same state

Example:
- First run: Creates security rule (CHANGED)
- Second run: Rule already exists (OK - no change)
- Third run: Rule already exists (OK - no change)

This is why Ansible is safe for automation!

## Tags - Run Specific Parts

Tags let you run only certain tasks:

```bash
# Only run NTP configuration
ansible-playbook playbooks/baseline_compliance.yml --tags ntp

# Run everything except commit
ansible-playbook playbooks/baseline_compliance.yml --skip-tags commit

# Run multiple tags
ansible-playbook playbooks/baseline_compliance.yml --tags "ntp,dns"
```

## Check Mode - Dry Run

Always test first with `--check`:

```bash
ansible-playbook playbooks/baseline_compliance.yml --check
```

This shows what WOULD change without actually changing anything.

## Common Patterns

### Pattern: Loop Through Items
```yaml
- name: Create multiple security rules
  panos_security_rule:
    name: "{{ item.name }}"
    action: "{{ item.action }}"
  loop:
    - { name: "Rule1", action: "allow" }
    - { name: "Rule2", action: "deny" }
```

### Pattern: Conditional Execution
```yaml
- name: Commit changes
  panos_commit_panorama:
    provider: "{{ provider }}"
  when: commit_changes | default(true)
```

### Pattern: Register and Use Results
```yaml
- name: Get system info
  panos_op:
    cmd: "show system info"
  register: system_info

- name: Display version
  debug:
    msg: "Version: {{ system_info.stdout }}"
```

## Directory Organization Best Practices

```
inventory/          # Environment-specific (prod vs staging)
├── production/    # Production devices and variables
└── staging/       # Test environment

playbooks/         # Task-specific (baseline vs NAT vs URL)
├── baseline_compliance.yml
├── nat_onboarding.yml
└── url_management.yml

roles/             # Function-specific (reusable components)
├── baseline_config/
└── panorama_common/

inventory/production/group_vars/  # Variable files (what values to use)
├── all.yml                       # Baseline defaults for all DGs
├── internet_facing.yml           # Internet-facing DG overrides
└── panorama.yml                  # Panorama connection settings
```

## Next Steps in Your Learning

1. **Week 1**: Run baseline_compliance.yml in check mode, understand the output
2. **Week 2**: Modify group_vars/all.yml, add your own security rules
3. **Week 3**: Create host_vars for specific DG customizations
4. **Week 4**: Learn Ansible Vault for encrypting passwords
5. **Week 5**: Expand to NAT rule onboarding

## Helpful Commands Reference

```bash
# Syntax check (does the YAML parse correctly?)
ansible-playbook playbooks/baseline_compliance.yml --syntax-check

# List all tasks that would run
ansible-playbook playbooks/baseline_compliance.yml --list-tasks

# See which hosts would be affected
ansible-playbook playbooks/baseline_compliance.yml --list-hosts

# Run with extra verbosity (for troubleshooting)
ansible-playbook playbooks/baseline_compliance.yml -vvv

# Run against specific host
ansible-playbook playbooks/baseline_compliance.yml --limit panorama-primary
```
