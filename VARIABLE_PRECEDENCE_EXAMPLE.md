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

### How This Works in Practice

Baseline variables are defined in `group_vars/all.yml`, giving them the LOWEST precedence:

1. **group_vars/all.yml** - Baseline defaults for all DGs
2. **group_vars/internet_facing.yml** - Overrides for internet-facing DGs
3. **host_vars/pano1-dg-gcp-internet-e1.yml** - Overrides for specific DG (HIGHEST)

This means:
-  Every DG gets sensible defaults from `all.yml`
-  Groups of DGs can share settings via `group_vars/<group>.yml`
-  Individual DGs can fully customize via `host_vars`
-  Command-line `--extra-vars` can override everything for emergencies


