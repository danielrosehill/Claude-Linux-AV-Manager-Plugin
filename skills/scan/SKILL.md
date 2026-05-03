---
name: scan
description: Run an on-demand security scan with one or more installed tools — ClamAV (clamscan / clamdscan against home or a chosen path), rkhunter, chkrootkit, Lynis (system audit), AIDE (integrity check). User picks scope (quick / deep / specific path) and which scanners to run. Reports go to the user-defined scan-results folder, organised per tool with timestamped filenames. Triggers on "scan my system", "run clamav", "rkhunter scan", "lynis audit".
---

# Scan

Single on-demand scan run. Reads `installed.*` to know what's available; asks the user which scanners + scope.

## Config

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/linux-av-manager/config.json
```

`scans_dir` is the report root. Per-tool subfolder; one file per scan named `<ISO-timestamp>.txt` (or `.json` where the tool supports it).

## Scope choices

- **Quick** — `~` only, ClamAV + rkhunter.
- **Deep** — full filesystem (`/`) excluding pseudo-FS (`/proc /sys /dev /run`), all installed scanners.
- **Path** — user-supplied path; ClamAV only.
- **Custom** — let the user pick scanners + path.

Default if not asked: quick.

## Per-tool commands

| Tool | Command | Output |
|---|---|---|
| ClamAV | `clamdscan --multiscan --fdpass <path>` (if `clamav-daemon` is up) else `clamscan -r <path>` | `<scans_dir>/clamav/<timestamp>.txt` |
| rkhunter | `sudo rkhunter --check --skip-keypress --report-warnings-only` | `<scans_dir>/rkhunter/<timestamp>.txt` |
| chkrootkit | `sudo chkrootkit -q` (`-q` = quiet, infections only) | `<scans_dir>/chkrootkit/<timestamp>.txt` |
| Lynis | `sudo lynis audit system --quick --no-colors` | `<scans_dir>/lynis/<timestamp>.txt` (also leaves `/var/log/lynis-report.dat`) |
| AIDE | `sudo aide --check` | `<scans_dir>/aide/<timestamp>.txt` |

Run in parallel only if independent and the host has the headroom — these are I/O heavy. Default to sequential.

## Reporting

After all scanners complete, write a `<scans_dir>/_summary/<timestamp>.md` with one section per tool:

- Tool, scanner version, signature/db version (if applicable).
- Files / paths scanned.
- Counts: infected / suspect / warnings / clean.
- The actual findings (full lines, not just counts).
- A `verdict` field: `clean`, `noisy` (warnings only), `findings` (real hits).

Highlight the **real findings** at the top of the summary — bury the `clean` entries.

## False-positive handling

- chkrootkit's `bindshell INFECTED` on port 465 is a known false positive (Postfix submissions). Annotate, don't alarm.
- rkhunter `Hidden files found` warnings are usually benign system files (`.gitignore`, `.cache`). Annotate.
- AIDE deltas after a system update are expected. Suggest `update-definitions` first if a recent apt upgrade ran.

## Notes

- Don't auto-quarantine. Surface findings; let the user act.
- Don't email results; just write to disk.
- Long ClamAV scans of `/` can take an hour+ — warn before starting deep mode.
