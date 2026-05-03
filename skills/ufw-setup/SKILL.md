---
name: ufw-setup
description: Set up UFW (Uncomplicated Firewall) on a Linux desktop with conservative, desktop-appropriate defaults — deny incoming, allow outgoing, allow established/related, allow common LAN-only services if present (mDNS, KDE Connect, syncthing) only after asking. Does NOT impose aggressive rules that break common desktop workflows. Triggers on "set up ufw", "configure firewall", "first-time ufw".
---

# UFW Setup

First-time UFW configuration tuned for a desktop, not a server. The principle: a sensible default that doesn't surprise the user by killing their printer / file sharing / smart-home discovery on day one. Add narrow allows only for services the user actually runs.

## Pre-flight

1. **Detect existing state.**
   - `command -v ufw` — install with `sudo apt install ufw` if missing.
   - `sudo ufw status verbose` — if already active and has non-default rules, this is **not** a first-time setup. Surface that and route the user to `ufw-maintain` instead.
2. **Detect services on the host that the user might want to keep reachable on LAN:**
   - `ss -tulnp 2>/dev/null` — sockets currently listening.
   - Cross-reference with these well-known desktop services and confirm with the user before allowing each:
     - **mDNS / Avahi** — UDP 5353 (printer/AirPlay/Chromecast discovery)
     - **KDE Connect** — UDP 1714-1764, TCP 1714-1764
     - **Syncthing** — TCP 22000, UDP 22000, UDP 21027
     - **Samba (file sharing)** — TCP 139, 445, UDP 137, 138
     - **CUPS (printer sharing)** — TCP 631
     - **SSH** — TCP 22 (only if the user actually runs sshd)
     - **VNC / RDP** — case-by-case
3. **Ask about scope.** Should the allows be LAN-only (`from 192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`) or wider? Default LAN-only.

## Apply

Build the rule set in this order, then activate:

```bash
sudo ufw --force reset                       # clean slate (only if user confirms; surfaces this is destructive)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw logging low                         # low = blocked packets only; 'medium'/'high' is too noisy for desktops
```

For each opt-in service the user accepted, add a LAN-scoped allow:

```bash
# Examples — only add the ones the user picked
sudo ufw allow from 192.168.0.0/16 to any port 5353 proto udp comment 'mDNS LAN'
sudo ufw allow from 192.168.0.0/16 to any port 1714:1764 proto udp comment 'KDE Connect LAN'
sudo ufw allow from 192.168.0.0/16 to any port 1714:1764 proto tcp comment 'KDE Connect LAN'
sudo ufw allow from 192.168.0.0/16 to any port 22000 proto tcp comment 'Syncthing LAN'
sudo ufw allow from 192.168.0.0/16 to any port 22000 proto udp comment 'Syncthing LAN'
sudo ufw allow from 192.168.0.0/16 to any port 21027 proto udp comment 'Syncthing discovery LAN'
sudo ufw allow from 192.168.0.0/16 to any port 631  proto tcp comment 'CUPS LAN'
```

Always tag rules with `comment '<purpose> <scope>'` so future audits make sense.

Then enable:

```bash
sudo ufw --force enable
sudo ufw status verbose
```

## Persist a snapshot

Write the resulting ruleset to:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/ufw/baseline-<ISO-timestamp>.rules
```

Source: `sudo ufw status numbered > <path>`. This baseline is the reference `ufw-maintain` diffs against later.

Also write to `config.json`:

```json
"ufw": {
  "enabled": true,
  "baseline_path": "<path>",
  "lan_cidrs": ["192.168.0.0/16", "10.0.0.0/8"],
  "configured_at": "<ISO-8601>"
}
```

## Notes

- Don't `ufw allow ssh` unless the host actually has sshd running. Most desktops don't.
- Don't enable IPv6 rules separately — Ubuntu's UFW config has IPv6 on by default (`/etc/default/ufw` `IPV6=yes`); rules apply to both stacks.
- Don't enable rate limiting (`ufw limit`) on a desktop — it's a server pattern.
- Don't open ports the user hasn't agreed to. The principle of this skill is *conservative*.
- If the user runs Docker, mention that Docker bypasses UFW by default — surface this as a known caveat, link to the `ufw-docker` workaround, but don't auto-apply it.
