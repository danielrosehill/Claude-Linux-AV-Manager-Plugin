# Linux AV Manager

Claude Code plugin for managing antivirus, rootkit-detection, and host firewall on a Linux desktop. Layered approach: install a small "core" set first (ClamAV, ClamTk, rkhunter), add advanced tools (Lynis, chkrootkit, AIDE) on top when wanted, set up UFW with desktop-appropriate defaults, keep definitions current, run scans, and schedule periodic runs.

## Skills

### Setup + tooling

- `onboard` — first-run setup. Picks the scan-results folder, records system info, and offers to install the core tool set.
- `install-core` — install ClamAV (CLI + daemon + freshclam), ClamTk (GUI front-end), rkhunter.
- `install-advanced` — install optional layer: Lynis, chkrootkit, AIDE, debsecan.

### Maintenance + runs

- `update-definitions` — refresh signature databases for every installed scanner.
- `scan` — on-demand scan with one or more installed scanners; report to scan-results folder.
- `schedule` — periodic runs via systemd timers (preferred) or cron, with desktop notifications on findings.

### Firewall

- `ufw-setup` — first-time UFW setup tuned for desktops: deny incoming, allow outgoing, opt-in LAN allows for mDNS / KDE Connect / Syncthing / Samba / CUPS only after asking. Conservative — won't break common desktop workflows.
- `ufw-maintain` — list, diff against baseline, add / remove / dedupe rules, flag overly-permissive entries, snapshot after every change.

## Installation

```bash
claude plugins install linux-av-manager@danielrosehill
```

## Notes

This plugin manages defensive AV and host-firewall tooling for personal/desktop use. It does not perform offensive scans, exploit testing, or evade detection.
