# Managing Multiple Device Groups with Different Baselines

## The Problem
You have multiple Panoramas, each with multiple Device Groups (DGs). Each DG needs **different baseline configurations** based on its purpose:
- Internet-facing: Strict security, high thresholds
- Partner networks: Moderate security, trusted sources
- DMZ: Strictest security, explicit rules
- Internal: Relaxed security, broader access

## The Solution: One Playbook, Multiple Configurations

### How It Works

```
Single Playbook (baseline_compliance.yml)
    ↓
    Runs against specific host (e.g., pano1-dg-gcp-internet-e1)
    ↓
    Loads host-specific variables (host_vars/pano1-dg-gcp-internet-e1.yml)
    ↓
    Variables override defaults (group_vars/all.yml)
    ↓
    Role executes with custom baseline for that DG
```

### Variable Precedence (Low to High)
1. **Default baseline** - `group_vars/all.yml` (applies to all)
2. **Group variables** - `group_vars/panorama.yml`, `group_vars/internet_facing.yml` (applies to groups)
3. **Host variables** - `host_vars/pano1-dg-gcp-internet-e1.yml` (specific DG)

**Later variables override earlier ones!**

## Directory Structure

```
inventory/production/
├── hosts.yml                              # Defines all DGs as hosts
├── group_vars/
│   ├── all.yml                           # Applies to EVERYTHING
│   └── panorama.yml                      # Applies to all Panoramas
└── host_vars/                            # DG-specific configs
    ├── pano1-dg-gcp-internet-e1.yml     # Internet DG baseline
    ├── pano1-dg-partner-gcp-e1.yml      # Partner GCP baseline
    ├── pano1-dg-partner-aws-e1.yml      # Partner AWS baseline
    └── pano2-dg-dmz-e1.yml              # DMZ baseline
```

## Usage Examples

### Run Against ALL Device Groups
```bash
ansible-playbook playbooks/baseline_compliance.yml
```
This runs against ALL DGs defined in the `panorama` group, each using its own host_vars!

### Run Against Specific Device Group
```bash
# Single DG
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1

# Multiple specific DGs
ansible-playbook playbooks/baseline_compliance.yml --limit "pano1-dg-gcp-internet-e1,pano1-dg-partner-gcp-e1"
```

### Run Against Logical Groups
```bash
# All internet-facing DGs
ansible-playbook playbooks/baseline_compliance.yml --limit internet_facing

# All partner networks
ansible-playbook playbooks/baseline_compliance.yml --limit partner_networks

# All cloud environments (GCP + AWS)
ansible-playbook playbooks/baseline_compliance.yml --limit cloud_environments

# All DGs on Panorama 1
ansible-playbook playbooks/baseline_compliance.yml --limit panorama_1
```

### Run Against Multiple Panoramas
```bash
# All DGs on both Panoramas
ansible-playbook playbooks/baseline_compliance.yml --limit "panorama_1,panorama_2"
```

### Check Mode (Test First!)
```bash
# See what would change for all DGs
ansible-playbook playbooks/baseline_compliance.yml --check

# See what would change for internet-facing only
ansible-playbook playbooks/baseline_compliance.yml --check --limit internet_facing
```

## Creating a New Device Group

### Step 1: Add to inventory/production/hosts.yml
```yaml
panorama_1:
  hosts:
    pano1-dg-new-environment:
      ansible_host: 192.168.1.10
      device_group: "New-Environment-DG"
      template_stack: "New-Template"
      environment: "production"
      network_type: "custom"
```

### Step 2: Create host_vars file (if different baseline needed)
```bash
# Create file: inventory/production/host_vars/pano1-dg-new-environment.yml
```

Copy an existing host_vars file as a template:
```bash
cp inventory/production/host_vars/pano1-dg-gcp-internet-e1.yml \
   inventory/production/host_vars/pano1-dg-new-environment.yml
```

Edit the new file with your custom baseline settings.

### Step 3: Run the playbook
```bash
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-new-environment --check
```

## Common Scenarios

### Scenario 1: Update baseline for ALL internet-facing DGs
1. Edit `group_vars/internet_facing.yml` (create if doesn't exist)
2. Add shared variables for all internet-facing DGs
3. Run: `ansible-playbook playbooks/baseline_compliance.yml --limit internet_facing`

### Scenario 2: Update baseline for ONE specific DG
1. Edit `host_vars/pano1-dg-gcp-internet-e1.yml`
2. Run: `ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1`

### Scenario 3: Emergency change to ALL DGs
1. Edit `inventory/production/group_vars/all.yml` (the default baseline)
2. Run: `ansible-playbook playbooks/baseline_compliance.yml`

### Scenario 4: Roll out new baseline gradually
```bash
# Test in staging first
ansible-playbook playbooks/baseline_compliance.yml -i inventory/staging --check

# Apply to one production DG
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1

# If successful, expand to similar DGs
ansible-playbook playbooks/baseline_compliance.yml --limit internet_facing

# Finally, roll out to all
ansible-playbook playbooks/baseline_compliance.yml
```

## Variable Override Examples

### Default (group_vars/all.yml)
```yaml
baseline_session:
  tcp_timeout: 3600  # 1 hour
```

### Internet DG Override (host_vars/pano1-dg-gcp-internet-e1.yml)
```yaml
baseline_session:
  tcp_timeout: 1800  # 30 minutes - stricter!
```

When you run the playbook:
- **Internet DG** uses `1800` (from host_vars)
- **Other DGs** use `3600` (from defaults)

## Best Practices

### 1. Use Descriptive Host Names
```yaml
# Good - tells you what it is
pano1-dg-gcp-internet-e1:

# Bad - unclear
fw-group-1:
```

### 2. Group Logically
Create groups for:
- Network types (internet, partner, dmz, internal)
- Cloud providers (gcp, aws, azure)
- Environments (prod, staging, dev)
- Panoramas (panorama_1, panorama_2)

### 3. Share Common Settings
Put shared settings in group_vars:
```yaml
# group_vars/internet_facing.yml
baseline_management:
  idle_timeout: 10  # All internet DGs get short timeout
```

### 4. Document Differences
Add comments in host_vars explaining why settings differ:
```yaml
# DMZ requires stricter timeouts due to compliance requirements
baseline_session:
  tcp_timeout: 900  # PCI-DSS requirement
```

### 5. Test Incrementally
Always test changes:
```bash
# 1. Check syntax
ansible-playbook playbooks/baseline_compliance.yml --syntax-check

# 2. Test one DG in check mode
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1 --check

# 3. Apply to one DG
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1

# 4. Expand to group
ansible-playbook playbooks/baseline_compliance.yml --limit internet_facing

# 5. Apply to all
ansible-playbook playbooks/baseline_compliance.yml
```

## Troubleshooting

### "Which baseline is this DG using?"
```bash
# See all variables for a specific host
ansible-inventory --host pano1-dg-gcp-internet-e1 --yaml

# Show where each variable comes from (requires -vvv)
ansible-playbook playbooks/baseline_compliance.yml --limit pano1-dg-gcp-internet-e1 -vvv
```

### "I changed host_vars but nothing happened"
Check variable names match exactly. Remember:
- `baseline_session.tcp_timeout` (correct structure)
- vs `tcp_timeout` (wrong - won't override)

### "How do I see all hosts in a group?"
```bash
# List all internet-facing hosts
ansible-inventory --graph internet_facing

# List all hosts
ansible-inventory --list
```

## Pro Tips

### Run in Parallel
Ansible runs sequentially by default. For faster execution:
```bash
# Run against 5 DGs at once
ansible-playbook playbooks/baseline_compliance.yml -f 5
```

### See What Changed
```bash
# Show diff of changes
ansible-playbook playbooks/baseline_compliance.yml --diff
```

### Tag Specific Tasks
Run only NTP updates across all DGs:
```bash
ansible-playbook playbooks/baseline_compliance.yml --tags ntp
```

### Skip Commits for Testing
```bash
ansible-playbook playbooks/baseline_compliance.yml --skip-tags commit
```

## Summary

✅ **One playbook** handles all Device Groups
✅ **host_vars/** defines DG-specific baselines
✅ **--limit** controls which DGs to target
✅ **Logical groups** enable batch operations
✅ **Variable precedence** allows flexible overrides

The same [playbooks/baseline_compliance.yml](playbooks/baseline_compliance.yml) works for every DG - the magic is in the inventory structure!
