---
name: onboard
description: First-run setup for linux-av-manager. Picks a scan-results folder (default ~/linux-av-manager/scans/), records distro/package-manager info, detects which AV tools are already installed, and offers to install the core set (ClamAV, ClamTk, rkhunter). Triggers on "set up linux-av-manager", "first run", or any scan/install request when no config exists.
---

# Onboard

One-time setup. Establishes a scan-results store the user owns and primes the plugin so other skills know what's installed and where to write reports.

## Config + storage

Plugin config lives at:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/config.json
```

Scan results (user-owned — they may want to inspect or archive them) live at a user-chosen path. Default suggestion: `~/linux-av-manager/scans/`. Store **only the pointer** in `config.json`:

```json
{
  "scans_dir": "/home/<user>/linux-av-manager/scans",
  "system": { "distro": "...", "package_manager": "apt|dnf|pacman|...", "init": "systemd" },
  "installed": { "clamav": false, "clamtk": false, "rkhunter": false, "lynis": false, "chkrootkit": false, "aide": false, "debsecan": false },
  "first_run_at": "<ISO-8601>"
}
```

If `config.json` already exists, this skill is a no-op — surface that and suggest `scan` or `install-advanced` instead.

## Steps

1. **Pick the scan folder.** Confirm or override the default. Create it with subfolders `clamav/`, `rkhunter/`, `lynis/`, `chkrootkit/`, `aide/` (made on demand by `scan`).
2. **Detect distro + package manager.** `cat /etc/os-release` → set `package_manager` (apt for Debian/Ubuntu/Mint, dnf for Fedora, pacman for Arch, zypper for openSUSE).
3. **Detect already-installed tools** — for each of `clamscan clamtk rkhunter lynis chkrootkit aide debsecan`, run `command -v <bin>` and record true/false.
4. **Write `config.json`** with detected state.
5. **Offer to install the core set** via `install-core`. Don't auto-install — let the user decide. If they accept, hand off; otherwise stop.

## Notes

- Don't write to `~/.claude/`.
- Scan results are user-owned — never put them under `$CLAUDE_USER_DATA`.
- Only ask one question (the folder path); detection handles the rest.
