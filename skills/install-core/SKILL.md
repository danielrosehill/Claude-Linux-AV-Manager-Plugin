---
name: install-core
description: Install the core AV tool set on Linux — ClamAV (clamscan + clamd + freshclam), ClamTk (GUI front-end), rkhunter. Uses the package manager recorded by `onboard`. Configures freshclam to run as a service, runs an initial definitions update, and updates `installed` flags in plugin config. Triggers on "install AV core", "install clamav", "set up antivirus".
---

# Install Core

Installs the baseline antivirus + rootkit detection layer.

## Config

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/config.json
```

If missing → call `onboard` first. Use `system.package_manager` to pick the right install command.

## Packages

| Tool | Debian/Ubuntu (apt) | Fedora (dnf) | Arch (pacman) |
|---|---|---|---|
| ClamAV CLI + daemon | `clamav clamav-daemon clamav-freshclam` | `clamav clamav-update clamd` | `clamav` |
| ClamTk GUI | `clamtk` | `clamtk` | `clamtk` (AUR) |
| rkhunter | `rkhunter` | `rkhunter` | `rkhunter` (AUR) |

Install in one batched command per package manager.

## Post-install

1. **Stop freshclam if running**, run an initial sync, then re-enable as a service:
   ```
   sudo systemctl stop clamav-freshclam
   sudo freshclam
   sudo systemctl enable --now clamav-freshclam
   ```
2. **Enable clamd** (the daemon — needed for fast on-demand scans via `clamdscan`):
   ```
   sudo systemctl enable --now clamav-daemon
   ```
   On Fedora the unit is `clamd@scan`. On Arch the user wires it manually — flag if so.
3. **Initialise rkhunter database**:
   ```
   sudo rkhunter --update
   sudo rkhunter --propupd
   ```
   `--propupd` baselines current file properties — only run on a *known-clean* system. Surface this caveat to the user before running and let them defer if they're unsure.
4. **Update `config.json`** — set `installed.clamav`, `installed.clamtk`, `installed.rkhunter` to true.
5. **Suggest next steps** — `scan` for an on-demand run, or `schedule` to set up periodic runs.

## Notes

- Don't run scans here — installation only.
- If a package is already installed (apt-get says "0 newly installed"), don't re-install; just update the flag.
