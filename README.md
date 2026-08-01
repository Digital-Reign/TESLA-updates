# TESLA-updates — Windows 11 Pro STIG / Hardening Script

A modernized fork of [Angry-Joe/Standalone-Windows-STIG-Script](https://github.com/Angry-Joe/Standalone-Windows-STIG-Script) (originally by [@SimeonOnSecurity](https://github.com/simeononsecurity)), updated for hardening **Windows 11 Pro** client machines.

**Note:** Review and test this script before running it on production hardware. It makes over a thousand configuration changes, and neither the original authors nor this fork's maintainers take responsibility for broken systems. Take a backup (the script also attempts a System Restore point) and keep BitLocker suspended until the post-run reboot completes.

## What changed in this fork (2026-07)

### Windows 11 Pro focus
- **OS/edition detection at startup.** The script now reports the detected OS build and warns when running on anything below Windows 11 (build 22000). On non-Enterprise editions it explains which STIG features simply cannot apply to **Pro** — Credential Guard, AppLocker enforcement, and Application Guard are Enterprise/Education-only. Pro still gets BitLocker, WDAC, memory integrity (HVCI), LSA protection, and Defender ASR hardening, so the bulk of the baseline applies cleanly.
- **Retired-technology sections now default to off** (each can be re-enabled with its parameter):
  - `-IE11` — Internet Explorer 11 was retired in June 2022 and is disabled on Windows 11.
  - `-java` — Oracle JRE 8 with its deployment.config mechanism is rarely present on modern clients.
  - `-horizon` — VMware Horizon STIGs don't apply to a typical client laptop.
- **Legacy Edge (EdgeHTML) registry tweaks removed.** The old `HKLM\...\MicrosoftEdge` policies targeted the discontinued UWP Edge. The section now sets the equivalent Chromium Edge policies (`InPrivateModeAvailability`, `ClearBrowsingDataOnExit`) alongside the DoD Edge GPO import.
- **New Windows 11 STIG items added** to the `-windows` section:
  - Remove/disable SMBv1 (WN11-00-000160/165/170)
  - Remove Windows PowerShell 2.0 engine (WN11-00-000155)
  - Enable LSA protection (`RunAsPPL`)
  - Restrict print driver installation to administrators (PrintNightmare mitigation)

### Bug fixes to the original script
- `If ($i = 1)` **assignments** used instead of `-eq` comparisons in the unquoted-service-path section — both registry passes ran on every iteration.
- The **NetBIOS-disable block never actually disabled NetBIOS** — it only read the value back. It now sets `NetbiosOptions = 2` on every interface.
- The **untrusted-fonts mitigation set the wrong bit**: the hex value `0x1000000000000` was passed as decimal `"1000000000000"`. Fonts were never blocked.
- Registry paths corrected: `Current Version` → `CurrentVersion` (publisher certificate revocation) and `Internet Explorer\Main Criteria` → `Internet Explorer\Main` (form autocomplete) — the originals wrote to nonexistent keys.
- The WPAD HKCU key was created nested one level too deep (`Wpad\Wpad`).
- The ECC-curve priority key is now created before it is written to.
- The "no options selected" guard referenced undefined variables and never fired; `$horizon` was missing from the parameter check.
- The System Restore point job referenced variables outside its scope — the description was always blank.

### Modernized PowerShell practice
- `Get-WmiObject` (removed in PowerShell 7) replaced with `Get-CimInstance`.
- `PSWindowsUpdate` now installs from PowerShell Gallery when online (bundled copy remains as an offline fallback, installed to the proper Program Files module path instead of System32).
- Comment-based help header and top-of-file `#Requires -RunAsAdministrator`.

### Still bundled from the original (know what you're running)
- The DoD GPO backups under `Files\GPOs` date from the original repo's last update. DISA has since released newer baselines (e.g., **Microsoft Windows 11 STIG V2R6, April 2026**). The GPO import mechanism is unchanged, so you can refresh the contents of `Files\GPOs\DoD\` with the latest [DISA GPO package](https://public.cyber.mil/stigs/gpo/) at any time — the folder layout just needs to match (one GPO backup folder per subdirectory, imported via `Files\LGPO\LGPO.exe`).
- The `mitigations` section still disables Windows Script Host and hibernation, and blocks untrusted fonts. These are sound for a hardened laptop but can surprise users (no `cscript`/`wscript` logon scripts, no hibernate/fast-startup). Run with `-mitigations $false` if that's a problem.

## Requirements
- Windows 11 Pro or better, fully updated. (Windows 10 and Enterprise editions still work, but this fork tests against 11 Pro.)
- Run from an **elevated** PowerShell prompt, from the repo root (the script needs `Files\`).
- **Suspend BitLocker before the first run**; re-enable after the post-run reboot.
- PowerShell execution policy that allows local scripts (`Set-ExecutionPolicy RemoteSigned -Scope Process`).
- ~10 GB free disk space for the restore point.

## How to run

```powershell
git clone https://github.com/Digital-Reign/TESLA-updates.git
cd TESLA-updates
.\secure-standalone.ps1
```

All parameters are optional booleans. Current defaults:

| Parameter | Default | Purpose |
|---|---|---|
| `cleargpos` | `$true` | Wipe and rebuild local group policy before importing |
| `installupdates` | `$true` | Install latest Windows updates (PSWindowsUpdate) |
| `adobe` | `$true` | Adobe Acrobat/Reader DC STIG |
| `firefox` | `$true` | Firefox STIG |
| `chrome` | `$true` | Google Chrome STIG |
| `IE11` | **`$false`** | IE11 STIG (retired browser) |
| `edge` | `$true` | Microsoft Edge (Chromium) STIG |
| `dotnet` | `$true` | .NET Framework 4 STIG |
| `office` | `$true` | Microsoft Office STIG |
| `onedrive` | `$true` | OneDrive STIG |
| `java` | **`$false`** | Oracle JRE 8 STIG (legacy) |
| `windows` | `$true` | Windows 10/11 STIG + audit policy + Win11 additions |
| `defender` | `$true` | Microsoft Defender STIG |
| `firewall` | `$true` | Windows Firewall STIG |
| `mitigations` | `$true` | Spectre/Meltdown, LLMNR, NetBIOS, WPAD, WDigest, WSH, fonts, etc. |
| `nessusPID` | `$true` | Fix unquoted service paths (Nessus 63155) |
| `horizon` | **`$false`** | VMware Horizon STIG |

Example — skip browser STIGs on a machine that only uses Edge:

```powershell
.\secure-standalone.ps1 -firefox $false -chrome $false
```

**A reboot is required after the script completes.**

## Windows 11 Pro caveats
| Feature | Pro support |
|---|---|
| BitLocker, TPM 2.0, Secure Boot | ✅ |
| Memory integrity (HVCI) / VBS | ✅ |
| LSA protection (RunAsPPL) | ✅ |
| WDAC (App Control for Business) | ✅ |
| Defender ASR rules | ✅ |
| Credential Guard | ❌ Enterprise/Education only |
| AppLocker enforcement | ❌ Enterprise/Education only |
| Application Guard | ❌ Deprecated by Microsoft |

GPO settings for unsupported features import without error — they just have no effect on Pro. Document them as "not applicable — edition" in any compliance checklist.

## Editing policies after the fact
- Import the ADMX policy definitions from [STIG-Compliant-Domain-Prep](https://github.com/simeononsecurity/STIG-Compliant-Domain-Prep/tree/master/Files/PolicyDefinitions) into `C:\Windows\PolicyDefinitions`.
- Open `gpedit.msc`.

## Sources and credits
- Original script: [simeononsecurity/Standalone-Windows-STIG-Script](https://github.com/simeononsecurity/Standalone-Windows-STIG-Script)
- Forked from: [Angry-Joe/Standalone-Windows-STIG-Script](https://github.com/Angry-Joe/Standalone-Windows-STIG-Script)
- [DoD Cyber Exchange — STIG downloads](https://public.cyber.mil/stigs/downloads/) and [GPO packages](https://public.cyber.mil/stigs/gpo/)
- [Microsoft Security Compliance Toolkit / LGPO](https://www.microsoft.com/en-us/download/details.aspx?id=55319)
- [NSACyber — Windows Secure Host Baseline](https://github.com/nsacyber/Windows-Secure-Host-Baseline)
