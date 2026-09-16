# Changelog

All notable changes to the scripts in this repo are documented here, grouped by folder. Each script tracks its own version in a `$ScriptVersion` variable in SECTION 1 (CONFIG) of the file — bump it whenever the script's logic changes, and add an entry below.

## ninjaone/

### 1.0.0 — 2026-09-16

Initial version of `ninjaone-acium.ps1`, adapted from `generic-acium.ps1`/`dattormm-acium.ps1` (both at 1.1.0) for NinjaOne. Reads its configuration (`AgentDownloadUrl`, `AgentOrganization`, `ExpectedPublisherCN`) from NinjaOne Automation Script Variables, injected into the script's environment for the run — the same shape as Datto RMM's Component Variables. Install/change-check/prerequisite/logging logic is otherwise identical to the other platform scripts.

## connectwise-rmm/

### 1.0.0 — 2026-09-16

Initial version of `connectwise-rmm-acium.ps1`, adapted from `generic-acium.ps1`/`dattormm-acium.ps1` (both at 1.1.0) for ConnectWise RMM (the Asio-based SaaS product, not ConnectWise Automate/LabTech, which uses a different scripting engine). ConnectWise RMM's exact mechanism for injecting a named script variable into a PowerShell script's environment could not be confirmed from public documentation at the time of writing, so configuration is **hardcoded** in SECTION 1 (same pattern as `generic-acium.ps1`) — see the `UNVERIFIED` note in the script's `.NOTES` header and the readme's opening section for what was and wasn't confirmed, and how to switch to variable-based config if you confirm the mechanism in your own tenant. Install/change-check/prerequisite/logging logic is otherwise identical to the other platform scripts.

## syncro/

### 1.0.0 — 2026-09-16

Initial version of `syncro-acium.ps1`, adapted from `generic-acium.ps1`/`dattormm-acium.ps1` (both at 1.1.0) for Syncro. Reads its configuration (`AgentDownloadUrl`, `AgentOrganization`, `ExpectedPublisherCN`) from Syncro Script Variables — confirmed via Syncro's own documentation to be injected as bare top-level PowerShell variables (e.g. `$AgentDownloadUrl`), not `$env:`-prefixed environment variables like Datto RMM/NinjaOne use; `Agent`-prefixed variable names avoid colliding with this script's own internal `$DownloadUrl`/`$Organization` variables. Install/change-check/prerequisite/logging logic is otherwise identical to the other platform scripts.

## n-able/

### 1.0.0 — 2026-09-16

Initial version of `n-able-acium.ps1`, adapted from `generic-acium.ps1`/`dattormm-acium.ps1` (both at 1.1.0), targeting **N-central** (Automation Manager's "Run PowerShell Script" Object) rather than N-sight RMM — N-central has documented Input/Output Parameters built specifically for PowerShell, while N-sight RMM's custom-script docs only show argument passing for batch/bash/VBScript. N-central's exact parameter-injection mechanism and default execution context could not be confirmed from public documentation, so the script reads its configuration via a standard `param()` block (the safest documented option) and flags both points as `UNVERIFIED` in the script's `.NOTES` header and the readme, with a test procedure to confirm SYSTEM context and parameter delivery before trusting it in production. Install/change-check/prerequisite/logging logic is otherwise identical to the other platform scripts.

## intune/

### 1.0.0 — 2026-09-16

Initial version, split into `intune-acium-detection.ps1` (read-only) and `intune-acium-remediation.ps1` (does the install), deployed as an Intune Remediation rather than a single script — Intune's plain "platform script" mechanism runs once per device and never recurs, which doesn't fit this repo's recurring/idempotent design, and Remediations have no per-deployment variable system, so configuration is hardcoded in both scripts (same pattern as `generic-acium.ps1`) between `EDIT THESE VALUES` markers. `$DownloadUrl` must be kept identical across both files — see the `.NOTES` in `intune-acium-remediation.ps1`. `intune-acium-remediation.ps1`'s install logic is otherwise adapted directly from `generic-acium.ps1` 1.1.0.

## generic/

### 1.1.0 — 2026-09-14

Promoted from `generic-acium-beta.ps1` to production as `generic-acium.ps1` after testing. Changes versus the prior production script:

- **ProductCode self-check.** The hardcoded `$ProductCode` is now cross-checked against the Uninstall registry. If Windows Installer says "not installed" but a matching product IS registered, the script logs the real ProductCode and uses it, instead of silently taking the first-time-install branch on an existing install (which produced error 1638).
- **Reboot gate.** If the ASP.NET Core hosting bundle install returns 3010 (reboot required), the script now stops instead of running the sensor MSI whose service can't yet start — that ordering was reproducing the exact 1603/1920 failure the prereq check exists to prevent. The next scheduled run completes the install.
- **SHA256 change detection in addition to ETag.** If the server ever stops returning an ETag header, the old script would reinstall the MSI on every scheduled run, fleet-wide, forever — and report success every time. The package hash now catches that.
- **Install state is read from the service, not a running process.** A stopped-but-installed service no longer looks like "not installed" and triggers a reinstall on every run.
- **`$WorkDir`/`$LogDir` ACLs are hardened.** `C:\ProgramData`'s default DACL lets standard users create directories in inherited subfolders; this script stages an MSI there and executes it as SYSTEM.
- **Authenticode check** on the MSI before running it as SYSTEM.
- **`Write-Log` can no longer kill the script.** Under `$ErrorActionPreference='Stop'`, a locked log file was an unhandled terminating error — exiting 1, which the exit-code table reads as "download failed."
- **msiexec 1618** ("another install in progress") is now retried.
- **TLS is OR'd into the existing protocol set** rather than replacing it, so TLS 1.3 isn't disabled on newer OSes.
- **Log files rotate** instead of growing unbounded; the MSI log appends instead of truncating every run.
- **Downloads retry, use the system proxy, and MSI property values are quoted correctly** (an `$Organization` containing a space used to silently break the old argument-array approach).
- **Added `$ScriptVersion`** tracking (this changelog).

## dattormm/

### readme.md — 2026-09-16 (docs only, no script change)

Rewrote the "Setup in Datto RMM" section with the actual current console flow, confirmed against a live tenant, and added screenshots for each step (`dattormm/images/`): Component Library's "Create Component" button (the button is labeled "Create Component", not "New Component"), the Create Component form fields (Name/Description/Category/Script type), setting Sites and adding Variables, the filled-in Component Variables, and the full Job-creation flow (Automation > Jobs > Create Job > Add Component > confirm variable values and Targets > Execution: Run as system account > Create Job). Removed the old standalone "Set execution context" step — execution context (System vs. logged-in user) is actually set per-Job during Job creation, not on the Component itself; there's no such setting on the Component form.

### 1.1.0 — 2026-09-14

Promoted from `dattormm-acium-beta.ps1` to production as `dattormm-acium.ps1` after testing. Changes versus the prior production script:

- **ORGANIZATION support**, via the new `AgentOrganization` Component Variable. The previous Datto script had no way to set it at all, so Datto-deployed sensors installed with no organization ID while the generic script set one — a silent divergence between the two copies.
- **ProductCode self-check.** The hardcoded `$ProductCode` is now cross-checked against the Uninstall registry. If Windows Installer says "not installed" but a matching product IS registered, the script logs the real ProductCode and uses it, instead of silently taking the first-time-install branch on an existing install (which produced error 1638).
- **Reboot gate.** If the ASP.NET Core hosting bundle install returns 3010 (reboot required), the script now stops instead of running the sensor MSI whose service can't yet start — that ordering was reproducing the exact 1603/1920 failure the prereq check exists to prevent. The next scheduled run completes the install.
- **SHA256 change detection in addition to ETag.** If the server ever stops returning an ETag header, the old script would reinstall the MSI on every scheduled run, fleet-wide, forever — and report success every time. The package hash now catches that.
- **Install state is read from the service, not a running process.** A stopped-but-installed service no longer looks like "not installed" and triggers a reinstall on every run.
- **`$WorkDir`/`$LogDir` ACLs are hardened.** `C:\ProgramData`'s default DACL lets standard users create directories in inherited subfolders; this script stages an MSI there and executes it as SYSTEM.
- **Authenticode check** on the MSI before running it as SYSTEM.
- **`Write-Log` can no longer kill the script.** Under `$ErrorActionPreference='Stop'`, a locked log file was an unhandled terminating error — exiting 1, which the exit-code table reads as "download failed."
- **msiexec 1618** ("another install in progress") is now retried.
- **TLS is OR'd into the existing protocol set** rather than replacing it, so TLS 1.3 isn't disabled on newer OSes.
- **Log files rotate** instead of growing unbounded; the MSI log appends instead of truncating every run.
- **Downloads retry, use the system proxy, and MSI property values are quoted correctly** (an `AgentOrganization` containing a space used to silently break the old argument-array approach).
- **Added `$ScriptVersion`** tracking (this changelog).
