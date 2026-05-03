---
name: schedule
description: Set up periodic AV scans on a Linux desktop using systemd timers (preferred) or cron. Schedules definition updates plus scans (ClamAV, rkhunter, Lynis) at user-chosen cadences (e.g. daily definitions refresh, weekly quick scan, monthly deep scan). Wires desktop notifications on findings via notify-send / libnotify. Triggers on "schedule scans", "set up periodic AV", "automate clamav".
---

# Schedule

Create systemd timer units (or cron jobs) that run `update-definitions` and `scan` at a user-chosen cadence. Notify on findings.

## Config

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/config.json
```

Persist the chosen schedule in `config.json` under `schedule`:

```json
"schedule": {
  "definitions_update": "daily",
  "quick_scan": "weekly",
  "deep_scan": "monthly",
  "notify": true
}
```

## Mechanism

Default to **systemd timers** — robust, handles missed runs (`Persistent=true`), integrates with journalctl. Fall back to cron only if the host isn't running systemd.

## Files

Create three timer + service unit pairs under `/etc/systemd/system/`:

- `linux-av-manager-definitions.{service,timer}` — runs the equivalent of the `update-definitions` skill via a small shell wrapper.
- `linux-av-manager-quick-scan.{service,timer}` — quick scan (~ only, ClamAV + rkhunter).
- `linux-av-manager-deep-scan.{service,timer}` — deep scan (full FS, all installed scanners).

Each service writes its report to the configured `scans_dir` and, if `notify == true` and a finding is non-clean, calls `notify-send` as the desktop user (use `loginctl show-user` / `XDG_RUNTIME_DIR` to send a notification from a system-scope unit, or install timers as user units under `~/.config/systemd/user/` and use `systemctl --user enable --now`).

For the desktop-notification path, **prefer user-scope units** — they run as the user already, so `notify-send` works without extra plumbing. Recommend this path; only use system-scope when the user explicitly wants root-owned timers.

### Wrapper scripts

Place small wrapper scripts under `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/bin/`:

- `run-update.sh` — invokes the same commands as `update-definitions`.
- `run-quick-scan.sh` — invokes `clamdscan ~` + `rkhunter --check --rwo` and writes to `<scans_dir>/clamav/` and `<scans_dir>/rkhunter/`.
- `run-deep-scan.sh` — full-FS variant; warns in the report header about expected runtime.

Each wrapper ends with: if the report contains findings, call `notify-send -u critical "Linux AV Manager" "Findings in <tool> — see <path>"`.

## Cadence translation

| Cadence | systemd `OnCalendar` |
|---|---|
| daily | `daily` |
| weekly | `weekly` (Mon 00:00) |
| monthly | `monthly` (first of month) |
| custom | accept a literal `OnCalendar` spec from the user |

Always set `Persistent=true` so missed runs (laptop asleep) catch up on next boot.

## Apply protocol

1. Show the user the proposed timer files and cadence.
2. Confirm once before writing.
3. Write units, run `systemctl daemon-reload` (or `--user`), then `enable --now` each timer.
4. Verify with `systemctl list-timers | grep linux-av-manager`.
5. Update `config.json` `schedule` block.

## Cron fallback

If systemd isn't available, drop the same wrappers and add `/etc/cron.d/linux-av-manager`:

```
0 3 * * *    root  /path/to/run-update.sh
0 4 * * 1    root  /path/to/run-quick-scan.sh
0 5 1 * *    root  /path/to/run-deep-scan.sh
```

Notifications won't reach the desktop from cron without extra plumbing — surface that limitation.

## Notes

- Don't enable timers without confirmation — these run as root for the system-scope path.
- Always offer to disable with `systemctl disable --now linux-av-manager-*.timer`.
