---
name: check-plugin-version
enabled: true
event: bash
action: block
conditions:
  - field: command
    operator: regex_match
    pattern: git\s+commit
  - field: command
    operator: not_contains
    pattern: --no-verify
---

⛔ **Version Update Required**

You're committing changes but haven't staged version updates in `.claude-plugin/`.

**Before committing, ensure:**
- [ ] `.claude-plugin/plugin.json` version is updated and staged
- [ ] `.claude-plugin/marketplace.json` version is updated and staged
- [ ] Both files have matching version numbers

**To fix:**
```bash
# Update versions in both files, then:
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
```

**If this is intentional** (no version bump needed), bypass with:
```bash
git commit --no-verify
```

**Why this matters:**
- Keeps plugin versions synchronized
- Prevents publishing with stale version numbers
- Ensures marketplace listings stay current
