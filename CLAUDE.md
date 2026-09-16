# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of PowerShell scripts that deploy and keep the **Acium Sensor** agent up to date on Windows endpoints via various RMM (Remote Monitoring and Management) platforms. There is no build system, package manager, or test suite — each subfolder is a self-contained `.ps1` script meant to be pasted into (or run directly on) a Windows target.

| Folder | Platform | Script |
|---|---|---|
| [`dattormm/`](dattormm/) | Datto RMM | `dattormm-acium.ps1` |
| [`generic/`](generic/) | Any RMM / scheduled task / manual | `generic-acium.ps1` |
| [`ninjaone/`](ninjaone/) | NinjaOne | `ninjaone-acium.ps1` |
| [`connectwise-rmm/`](connectwise-rmm/) | ConnectWise RMM | `connectwise-rmm-acium.ps1` |
| [`intune/`](intune/) | Microsoft Intune | `intune-acium-detection.ps1` + `intune-acium-remediation.ps1` (Intune Remediations pair, not a single script — see the folder's readme) |
| [`syncro/`](syncro/) | Syncro | `syncro-acium.ps1` |
| [`n-able/`](n-able/) | N-able | `n-able-acium.ps1` |

## Remotes — two repos, different purposes

```
origin -> Ben-Acium/acium-rmm-deployment   (work-in-progress / beta)
all    -> pushes to BOTH Ben-Acium/acium-rmm-deployment AND Acium-Inc/rmm-scripts
```

- `Ben-Acium/acium-rmm-deployment` is for work-in-progress/beta scripts.
- `Acium-Inc/rmm-scripts` is the shared org production repo. **Only push tested, production-ready content there** — never beta/WIP scripts. `main` on `Acium-Inc/rmm-scripts` is protected by an org-level ruleset; changes go through a PR.
- To publish a promoted change to both repos, push to the `all` remote (or push to `origin` first, then separately to `Acium-Inc/rmm-scripts` via a PR).

## Script architecture

`generic-acium.ps1`, `dattormm-acium.ps1`, `ninjaone-acium.ps1`, `connectwise-rmm-acium.ps1`, `syncro-acium.ps1`, `n-able-acium.ps1`, and `intune-acium-remediation.ps1` are all near-duplicates of each other — same install logic, same numbered `SECTION` structure — differing mainly in **where configuration comes from**:

- `generic-acium.ps1` / `intune-acium-remediation.ps1`: all config is hardcoded in SECTION 1 (`$DownloadUrl`, `$Organization`, `$ExpectedPublisherCN`) between `EDIT THESE VALUES TO CONFIGURE A DEPLOYMENT` markers. `generic-acium.ps1` is drop-in usable on any platform with no variable system required; `intune-acium-remediation.ps1` is hardcoded because Intune Remediations have no per-deployment variable system at all (see `intune/readme.md`).
- `dattormm-acium.ps1`, `ninjaone-acium.ps1`, `connectwise-rmm-acium.ps1`, `syncro-acium.ps1`, `n-able-acium.ps1`: config is read from that platform's script-variable feature, exposed to the script as environment variables (`$env:AgentDownloadUrl`, `$env:AgentOrganization`, `$env:ExpectedPublisherCN`), configured in that platform's console rather than in the script. Each platform's readme documents its own variable UI and any platform-specific caveats.

Because these scripts duplicate their install logic, **a fix or behavioral change almost always needs to be made in every one of them** (adjusted for the config-source difference), with a matching version bump and CHANGELOG entry for each. `intune-acium-detection.ps1` is the one exception to the "near-duplicate" pattern — it's a lightweight, read-only companion to `intune-acium-remediation.ps1` (Intune's detect/remediate model), not a full copy of the install logic; see `intune/readme.md` for why.

Both scripts follow the same section layout — useful for finding where to make a change:

1. **CONFIG** — `$ScriptVersion`, config values (hardcoded or from `$env:`), file paths, `$ProductCode`/`$DisplayNamePattern` for install-state detection.
2. **SETUP** — creates/locks down working directories (custom ACLs, since `C:\ProgramData`'s default DACL lets standard users create subfolders — if hardening a directory fails, the script fails closed with exit `2` rather than proceeding against a directory that may still be junction/write-attackable), rotates the log, defines `Write-Log`, takes a machine-wide mutex so overlapping scheduled runs don't race.
3. **SHARED HELPERS** — `Set-SecurityProtocol` (TLS), `Invoke-Download` (WebClient-based, retries, honors system proxy), `Get-SensorInstallState` (service presence, not process), `Find-InstalledProduct` (registry cross-check by `$DisplayNamePattern`).
4. **CHANGE CHECK** — decides whether to skip this run: compares the server's ETag against the last successful run's saved ETag, AND confirms the sensor is actually still installed. Both must agree to skip.
5. **DOWNLOAD** — downloads the `.zip`, then does a second, server-independent change check via SHA256 of the downloaded bytes (catches a server that stops sending ETag headers).
6. **PREREQUISITE CHECK** — installs the ASP.NET Core 8.0 Hosting Bundle if missing (hard requirement for the sensor service to start). If the bundle install itself needs a reboot (exit 3010), the script **defers and exits 0** rather than proceeding to install the sensor MSI against a runtime that can't start yet — doing otherwise reproduces a real MSI 1603/Windows Installer 1920 failure that was hit in testing.
7. **EXTRACT** — unzips, locates `AciumSensorInstall.msi`, checks its Authenticode signature (enforced only if `$ExpectedPublisherCN` is set).
8. **INSTALL** — decides first-time-install vs. reinstall by querying Windows Installer's `ProductState` for `$ProductCode`, cross-checked against the registry. Getting this branch wrong is a known failure mode: passing `REINSTALL=ALL`/`REINSTALLMODE=vomus` on a genuine first-time install makes every MSI component resolve to `Action: Null` — a silent no-op install that still reports exit code 0. Runs `msiexec` (built as one explicit string, not a PowerShell array, so a config value containing a space quotes correctly), retrying on 1618 ("another install in progress"). Verifies post-install that the sensor is actually present rather than trusting the exit code alone.
9. **SAVE STATE** — writes `last-installed.json` (ETag, SHA256, ProductCode, agent version, `$ScriptVersion`) only after a verified successful install, so a failed/no-op run never poisons the next run's change check.
10. **CLEANUP** — removes the downloaded zip and extracted files.

### Versioning convention

- Each script tracks its own revision in `$ScriptVersion` (SECTION 1) — **separate from** the deployed agent version (`$DownloadUrl` / `AgentDownloadUrl`).
- Bump `$ScriptVersion` whenever a script's logic changes, and add a dated entry under the matching folder's heading in [`CHANGELOG.md`](CHANGELOG.md) at the repo root describing what changed and why.
- `$ScriptVersion` is logged at deployment start and recorded in the SECTION 9 saved state JSON.

### Exit codes (shared by both scripts)

| Code | Meaning |
|---|---|
| `0` | Success — installed, already up to date (no-op), or deferred pending a reboot |
| `1` | Download failed |
| `2` | Could not secure the working/log directory (ACL hardening failed) — refused to proceed with a privileged install |
| `3` | Install failed |
| `4` | Missing/invalid required config (`$DownloadUrl` empty, or `AgentDownloadUrl` Component Variable missing) |
| `5` | Zip extraction failed / MSI not found inside package |
| `6` | MSI failed Authenticode signature verification |

## Adding a new script (from the README)

1. Create a subfolder named for the target platform (e.g. `ninjaone/`, `n-able/`).
2. Add the script(s) plus a `readme.md` documenting what it does, prerequisites, required variables/inputs, and exit codes.
3. Link the new folder from the table in the root [`README.md`](README.md).

## Testing

There is no automated test suite — these scripts require a real (or VM) Windows endpoint with SYSTEM/Administrator privileges to exercise meaningfully. Verify changes by running the script on a Windows test machine and checking `C:\ProgramData\AciumSensor\Logs\deploy.log` and `msi-install.log`, per the Troubleshooting sections in each folder's `readme.md`.
