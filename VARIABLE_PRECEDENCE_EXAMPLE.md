# Variable Precedence - Real Example

## The Question
"I run one playbook against 3 different DGs - how does each get different config?"

## The Answer: Variable Precedence

### Example: TCP Timeout Setting

Let's trace how `tcp_timeout` is determined for 3 different DGs.

#### 1️⃣ Default Baseline (group_vars/all.yml)
```yaml
baseline_session:
  tcp_timeout: 3600  # 1 hour - the default
  udp_timeout: 30
```

#### 2️⃣ Internet DG Override (host_vars/pano1-dg-gcp-internet-e1.yml)
```yaml
baseline_session:
  tcp_timeout: 1800  # 30 minutes - OVERRIDES default
  udp_timeout: 30    # Same as default
```

#### 3️⃣ Partner DG - No Override (host_vars/pano1-dg-partner-gcp-e1.yml)
```yaml
# baseline_session not defined - uses default
```

#### 4️⃣ DMZ Override (host_vars/pano2-dg-dmz-e1.yml)
```yaml
baseline_session:
  tcp_timeout: 900   # 15 minutes - OVERRIDES default
  udp_timeout: 15    # Also overridden
```

### What Happens When Playbook Runs?

```bash
ansible-playbook playbooks/baseline_compliance.yml --limit "pano1-dg-gcp-internet-e1,pano1-dg-partner-gcp-e1,pano2-dg-dmz-e1"
```

**Result:**
| Device Group | tcp_timeout Used | Source |
|-------------|------------------|---------|
| pano1-dg-gcp-internet-e1 | 1800 | host_vars/pano1-dg-gcp-internet-e1.yml |
| pano1-dg-partner-gcp-e1 | 3600 | group_vars/all.yml (default) |
| pano2-dg-dmz-e1 | 900 | host_vars/pano2-dg-dmz-e1.yml |

### Visual Flow

```
┌──────────────────────────────────────────────────────────────┐
│ Run: ansible-playbook playbooks/baseline_compliance.yml     │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌───────────────────┐    ┌────────────────────┐
│ Internet DG Host  │    │ Partner DG Host    │
└───────┬───────────┘    └─────────┬──────────┘
        │                          │
        ▼                          ▼
┌──────────────────────────────────────────────┐
│ 1. Load Default (group_vars/all.yml)        │
│    tcp_timeout: 3600                         │
└────────────────┬─────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
┌────────────────┐  ┌──────────────────┐
│ Internet DG:   │  │ Partner DG:      │
│ 2. Load        │  │ 2. Load          │
│    host_vars   │  │    host_vars     │
│    ↓           │  │    ↓             │
│ OVERRIDES to   │  │ No override      │
│ tcp_timeout:   │  │ KEEPS:           │
│ 1800           │  │ tcp_timeout:     │
│                │  │ 3600             │
└────────────────┘  └──────────────────┘
```

## Full Precedence Order (Low to High)

When Ansible looks for a variable, it checks in this order:

```
1. role defaults (roles/baseline_config/defaults/main.yml) - LOWEST
   ↓ (can be overridden by)

2. group_vars/all.yml                   (baseline defaults for all hosts)
   ↓ (can be overridden by)

3. group_vars/panorama.yml              (all panorama hosts)
4. group_vars/internet_facing.yml       (internet DG group)
   ↓ (can be overridden by)

5. host_vars/pano1-dg-gcp-internet-e1.yml  (specific DG)
   ↓ (can be overridden by)

6. --extra-vars on command line         (HIGHEST priority!)
```

**The last one wins!**

### ✅ How This Works in Practice

Baseline variables are defined in `group_vars/all.yml`, giving them the LOWEST precedence:

1. **group_vars/all.yml** - Baseline defaults for all DGs
2. **group_vars/internet_facing.yml** - Overrides for internet-facing DGs
3. **host_vars/pano1-dg-gcp-internet-e1.yml** - Overrides for specific DG (HIGHEST)

This means:
- ✅ Every DG gets sensible defaults from `all.yml`
- ✅ Groups of DGs can share settings via `group_vars/<group>.yml`
- ✅ Individual DGs can fully customize via `host_vars`
- ✅ Command-line `--extra-vars` can override everything for emergencies

## Practical Example: Security Profile

### Scenario
You want:
- **Internet DGs**: Use "strict" URL filtering
- **Partner DGs**: Use "alert-only" URL filtering
- **DMZ**: Use "strict" URL filtering
- **Default**: Use "default" URL filtering

### Implementation

**Step 1: Set Default** - `group_vars/all.yml`
```yaml
baseline_security_profile_group:
  url_filtering: "default"  # Most DGs use this
```

**Step 2: Override for Internet** - `host_vars/pano1-dg-gcp-internet-e1.yml`
```yaml
baseline_security_profile_group:
  url_filtering: "strict"  # Stricter for internet
```

**Step 3: Override for Partners** - `host_vars/pano1-dg-partner-gcp-e1.yml`
```yaml
baseline_security_profile_group:
  url_filtering: "alert-only"  # Lenient for partners
```

**Step 4: Override for DMZ** - `host_vars/pano2-dg-dmz-e1.yml`
```yaml
baseline_security_profile_group:
  url_filtering: "strict"  # Strictest for DMZ
```

### Result Table
| DG | url_filtering | Source |
|---|---|---|
| pano1-dg-gcp-internet-e1 | strict | host_vars override |
| pano1-dg-internet-e1 | default | default (no host_vars file) * |
| pano1-dg-partner-gcp-e1 | alert-only | host_vars override |
| pano1-dg-partner-aws-e1 | alert-only | host_vars override |
| pano2-dg-dmz-e1 | strict | host_vars override |
| pano2-dg-internal-e1 | default | default (no host_vars file) * |

**Note**: * These hosts exist in inventory but intentionally have **no host_vars file**. This is by design - they use group_vars and default baseline values. Not every host needs custom host_vars overrides.

## Using Group Variables

Instead of setting the same value in multiple host_vars files, use group_vars:

### Create group_vars/internet_facing.yml
```yaml
# This applies to ALL hosts in the internet_facing group
baseline_security_profile_group:
  url_filtering: "strict"
```

Now you don't need to define it in each internet DG's host_vars!

```
inventory/production/
├── hosts.yml
├── group_vars/
│   ├── all.yml
│   ├── panorama.yml
│   └── internet_facing.yml          ← Applies to group!
└── host_vars/
    ├── pano1-dg-gcp-internet-e1.yml ← Can still override if needed
    └── pano1-dg-internet-e1.yml
```

## Override Priority Example

Let's say you have:

**group_vars/internet_facing.yml**
```yaml
baseline_session:
  tcp_timeout: 1800
```

**host_vars/pano1-dg-gcp-internet-e1.yml**
```yaml
baseline_session:
  tcp_timeout: 900  # Even stricter for this one DG
```

**Result:**
- `pano1-dg-gcp-internet-e1` gets `900` (host_vars wins)
- `pano1-dg-internet-e1` gets `1800` (from group_vars)

## Command-Line Overrides (Highest Priority!)

Emergency change needed? Override everything:

```bash
ansible-playbook playbooks/baseline_compliance.yml \
  --extra-vars "baseline_session={tcp_timeout: 600}"
```

This forces ALL DGs to use 600, regardless of what's in files!

**Use case:** Emergency security response - need to enforce setting NOW across everything.

## Debugging: "Which Value Is Being Used?"

### Method 1: Check Inventory
```bash
ansible-inventory --host pano1-dg-gcp-internet-e1 --yaml
```

Shows ALL variables for that host (after precedence applied).

### Method 2: Debug in Playbook
Add to your playbook:
```yaml
- name: Debug baseline values
  ansible.builtin.debug:
    msg:
      - "DG: {{ device_group }}"
      - "TCP Timeout: {{ baseline_session.tcp_timeout }}"
      - "URL Filtering: {{ baseline_security_profile_group.url_filtering }}"
```

### Method 3: Very Verbose
```bash
ansible-playbook playbooks/baseline_compliance.yml -vvv
```

Shows where each variable comes from.

## Key Takeaways

✅ **Default first** - Set sensible defaults in `group_vars/all.yml`

✅ **Override selectively** - Only define host_vars for DGs that differ

✅ **Group similar DGs** - Use group_vars for DGs that share settings

✅ **Host_vars wins** - Most specific override always wins

✅ **Command line trumps all** - Use --extra-vars for emergency overrides

This is how ONE playbook serves ALL device groups with different configs! 🎯
