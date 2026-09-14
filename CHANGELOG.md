# Changelog

All notable changes to the scripts in this repo are documented here, grouped by folder. Each script tracks its own version in a `$ScriptVersion` variable in SECTION 1 (CONFIG) of the file — bump it whenever the script's logic changes, and add an entry below.

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
