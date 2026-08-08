# Group Policy

Five GPOs, all linked at `OU=Compounds,OU=Konoha,DC=homelab,DC=local`.

| GPO | Purpose |
|---|---|
| `Security - Screen Lock 5 Minutes` | Idle session lock (user-side settings, loopback processed) |
| `Deploy 7-Zip` | Software deployment via MSI |
| `WSUS Client Settings` | Windows Update targeting |
| `Security - Block USB Storage` | Removable media restriction |
| `Optional Features - Direct from Windows Update` | Features on Demand source override |

Linking at a dedicated computer OU rather than the domain root keeps machine policy off domain controllers, which stay in their default container.

Verify what actually applied:

```powershell
gpresult /r /scope:computer
gpresult /r /scope:user
```

## Loopback processing

`Security - Screen Lock 5 Minutes` contains **User Configuration settings only**. It is linked on the computer branch at `Compounds`, while user accounts live on the user branch at `OU=Shinobi`.

The result was that it applied to nothing. `gpresult /r /scope:user` returned `Applied Group Policy Objects: N/A` with no error anywhere.

Confirm which side a GPO's settings live on:

```powershell
Get-GPOReport -Name "Security - Screen Lock 5 Minutes" -ReportType Xml
```

Computer extension data empty, User extension data `Registry` — user-side only.

### Fix

```powershell
Set-GPRegistryValue -Name "Security - Screen Lock 5 Minutes" `
    -Key "HKLM\Software\Policies\Microsoft\Windows\System" `
    -ValueName "UserPolicyMode" -Type DWord -Value 1
```

| Mode | Value | Behavior |
|---|---|---|
| Merge | 1 | User's own GPOs apply first, then computer-branch user settings layer on top |
| Replace | 2 | User's own GPOs discarded entirely |

Merge was chosen deliberately. The environment currently has no user-linked GPOs, so both modes behave identically today — but Merge stays safe once user-branch policy is added, where Replace would silently discard it.

The design rationale for putting screen lock on the computer branch at all: it is a machine security baseline. Any workstation in the fleet locks after five minutes regardless of who signs in. Treating it as a user preference would let it follow a person onto unmanaged devices and would let a user without the policy sit unlocked on a corporate machine.

### Verification

Signed in as a standard user:

```powershell
Get-ItemProperty "HKCU:\Software\Policies\Microsoft\Windows\Control Panel\Desktop"
```

```
ScreenSaveTimeOut  : 300
ScreenSaverIsSecure: 1
ScreenSaveActive   : 1
```

> Loopback evaluates at **logon**. `gpupdate /force` alone will not surface the change — log off and back on.

Loopback is now active for every machine under `Compounds`, including future hosts in `Towers`, `Branches`, and `Academy`. If user policy ever fails to apply unexpectedly on those machines, check loopback first.

## Software Installation: a GPO that applies but does nothing

`Deploy 7-Zip` appeared under Applied Group Policy Objects on the workstation. 7-Zip never installed. No error appeared in any log.

Everything checked out individually:

| Checked | Result |
|---|---|
| Deployment type | Assigned, not Published — correct |
| Source path | `\\LAB-DC\Software$\7z2501-x64.msi` — present, 1,996,800 bytes |
| Share ACL | `HOMELAB\Domain Computers` = Read |
| NTFS ACL | `HOMELAB\Domain Computers` = ReadAndExecute, Synchronize |
| SYSTEM-context share access | Confirmed via scheduled task |
| Transforms | None |
| **Deployment Count** | **0** |

`Deployment Count: 0` was the tell. No client had ever processed the package — this was not a failure to install, it was a failure to attempt.

The Group Policy operational log confirmed it:

```
Event 4016: Starting Software Installation Extension Processing.
            List of applicable Group Policy objects: (No changes were detected.)
            Deploy 7-Zip
Event 5016: Completed Software Installation Extension Processing in 0 milliseconds.
```

### Root cause

Software Installation is a **change-driven client-side extension**. It compares the GPO version against its cached state and no-ops when they match.

The GPO had been evaluated while the workstation still sat in `CN=Computers` receiving no policy at all. After the move into `Training-Grounds`, the GPO's own version number had not changed — only the machine's location had. The CSE compared versions, found no change, and skipped processing entirely. It never read the share and never invoked the MSI.

There was never an error to find, because nothing was attempted.

### Fix

Force reprocessing regardless of change detection, on the client:

```powershell
$cse = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Group Policy\{c6dc5466-785a-11d2-84d0-00c04fb169f7}"
New-Item -Path $cse -Force | Out-Null
Set-ItemProperty -Path $cse -Name "NoGPOListChanges" -Value 0 -Type DWord
```

`{c6dc5466-785a-11d2-84d0-00c04fb169f7}` is the Software Installation CSE GUID.

Reboot afterward. **Software Installation processes only at boot** — `gpupdate /force` will never trigger an MSI install, which is its own trap when testing.

Confirmed installed: `7-Zip 25.01 (x64 edition)`. Post-fix CSE processing time moved from 0 ms to 31 ms — the clearest signal that it was now doing work rather than skipping.

### Two things this generalizes to

**A `Test-Path` against a share tests the calling user, not the machine account.** Software Installation runs as SYSTEM and authenticates as `DOMAIN\MACHINE$`. An interactive `Test-Path` failure does not prove a permissions problem, and success does not prove the machine account has access.

**Absence of an error is diagnostic information.** Before investigating why an operation failed, establish whether it ran at all. `Deployment Count: 0` and 0 ms processing time answered that in this case, and pointed somewhere completely different from where the symptom suggested.

## Features on Demand and WSUS

RSAT installation on the workstation fails with `0x800f0954`.

WSUS does not distribute Features on Demand. A fully healthy WSUS still produces this error, so the error should not be read as a WSUS fault — that misreading costs a lot of time.

The correct configuration is a GPO overriding the FoD source:

```
Computer Configuration → Administrative Templates → System
→ Specify settings for optional component installation and component repair → Enabled
   ☑ Download repair content and optional features directly from Windows Update instead of WSUS
   Alternate source path: (empty)
```

Verify on the client:

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Servicing" |
    Select-Object RepairContentServerSource
```

`RepairContentServerSource = 2` confirms the policy applied.

> **Status: unresolved.** The GPO is confirmed applied and the failure persists. Share permissions, NTFS permissions, DNS, firewall path, WSUS service health, delivery endpoint reachability, and reboot-cycle policy caching have all been eliminated. Additional policy blockers documented in community fixes (`DisableWindowsUpdateAccess`, `DoNotConnectToWindowsUpdateInternetLocations`) were checked and are absent. A build-mismatch theory was investigated and disproved — 24H2 and 25H2 share an identical set of system files, so a `26100` FoD package is correct for a `26200` client.
>
> The offline media path requires `FoD_Part1` ISO media, distributed only through volume licensing, and is not obtainable here.
>
> RSAT is not required to validate the Tier 1 delegation — `runas /user:HOMELAB\tier1.helpdesk "mmc dsa.msc"` on a DC proves it equally well. Documented as an open item rather than worked around.

**Diagnose FoD failures from CBS, not DISM.** `dism.log` reports only the propagated HRESULT; `CBS.log` names the actual condition.

## Quick reference

| Symptom | Check |
|---|---|
| GPO listed as applied, settings absent | Which config branch the settings live on; loopback if user-side on a computer-linked GPO |
| User policy shows `N/A` | Loopback processing; log off and back on, not `gpupdate` |
| Software Installation never runs | `Deployment Count`; force with `NoGPOListChanges`; reboot, not `gpupdate` |
| Freshly joined machine gets no policy | Object in `CN=Computers`; move it or use `redircmp` |
| DC not receiving `Default Domain Controllers Policy` | Computer object moved out of the default DC container |
| `0x800f0954` on RSAT install | FoD GPO; not a WSUS fault |
