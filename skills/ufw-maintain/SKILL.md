---
name: ufw-maintain
description: Maintain an existing UFW ruleset on a Linux desktop — list current rules, diff against the saved baseline, add or remove individual rules, dedupe redundant entries, flag overly-permissive rules (any-source allows, broad port ranges), and snapshot the new state. Triggers on "show ufw rules", "add ufw rule", "remove ufw rule", "audit ufw", "update firewall".
---

# UFW Maintain

Day-2 firewall ops. Read the current rule set, compare to the baseline `ufw-setup` saved, and let the user mutate it surgically.

## Config

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/config.json
```

Read `ufw.baseline_path` and `ufw.lan_cidrs`. If the `ufw` block is missing → tell the user to run `ufw-setup` first.

## Modes

The skill is dispatched on user intent:

### 1. Show / audit

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Then:

- **Diff against baseline.** Compare current numbered rules against the most recent baseline file. Report added / removed lines.
- **Permissiveness audit.** Flag any rule that:
  - Has no source restriction (`Anywhere` in the From column) on a non-trivial port (anything other than ICMP/established).
  - Opens a wide port range (>20 contiguous ports) without a specific source CIDR.
  - Is missing a `comment` (anonymous rules accumulate cruft over time).
  - Duplicates another rule (same proto/port/source/dest).
- **Suggest.** For each flagged rule, propose a tightened replacement — don't auto-apply.

### 2. Add a rule

Gather:
- Port (single or range) + protocol (`tcp` / `udp` / `any`).
- Source — default to LAN CIDRs from config; ask before allowing `Anywhere`.
- Comment — required. Refuse to add a rule without one (this is the desktop discipline).

Apply:

```bash
sudo ufw allow from <source> to any port <port> proto <proto> comment '<comment>'
```

Show before/after `ufw status numbered`.

### 3. Remove a rule

- Show numbered rules.
- User picks rule number(s).
- For each: `sudo ufw delete <n>`. Note that numbers shift after each delete — re-read after each removal, or sort descending and delete in that order.

### 4. Dedupe

Parse `ufw status numbered`; group by (proto, port, source, dest, action). For any group with >1 rule, propose keeping the one with the most descriptive comment and removing the rest.

### 5. Reset to baseline

If the user wants to revert: read the baseline file, do `sudo ufw --force reset`, then re-apply each line. Confirm explicitly — this is destructive of any post-baseline rules.

## After every mutation

Snapshot the new state:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/ufw/snapshot-<ISO-timestamp>.rules
```

Update `config.json`:

```json
"ufw": {
  "enabled": true,
  "baseline_path": "<original baseline, unchanged>",
  "latest_snapshot": "<new snapshot path>",
  "last_modified_at": "<ISO-8601>"
}
```

Don't overwrite the baseline — keep it as the immutable reference for future diffs.

## Notes

- Conservative-by-default extends here too: the skill never proposes opening a new port without an explicit user request, never auto-approves an `Anywhere` allow, and never removes rules without confirmation.
- If `sudo ufw status` returns "Status: inactive" → the user disabled UFW elsewhere. Surface it; don't silently re-enable.
- For complex rule sets the numbered interface gets unwieldy — for >50 rules, suggest the user move to managing `/etc/ufw/user.rules` directly with version control.
