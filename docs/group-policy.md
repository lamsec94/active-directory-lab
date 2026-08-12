# Group Policy

Five GPOs, all linked at `OU=Compounds,OU=Konoha,DC=homelab,DC=local`.

| GPO | Purpose | Status |
|---|---|---|
| `Security - Screen Lock 5 Minutes` | Idle session lock (user-side settings, loopback processed) | Applying |
| `Deploy 7-Zip` | Software deployment via MSI | Applying, source missing — see [below](#deploy-7-zip-source-package-missing) |
| `WSUS Client Settings` | Windows Update targeting | Applying, target decommissioned — see [below](#wsus-client-settings-points-at-a-role-that-no-longer-exists) |
| `Security - Block USB Storage` | Removable media restriction | Applying, broader than intended — see [below](#removable-storage-policy-is-broader-than-its-name) |
| `Optional Features - Direct from Windows Update` | Features on Demand source override | Applying |

Linking at a dedicated computer OU rather than the domain root keeps machine policy off domain controllers, which stay in their default container.

Verify what actually applied:

```powershell
gpresult /r /scope:computer
gpresult /r /scope:user
```

All five GPOs plus `Default Domain Policy` were confirmed applying on the workstation after a domain controller rebuild, sourced from the rebuilt DC — policy survives a DC rebuild because it lives in the directory and SYSVOL, both of which replicate.

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

## Removable storage policy is broader than its name

`Security - Block USB Storage` denies read, write, and execute on device class `{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}`. That class covers WPD and removable disks — **and CD/DVD drives**.

The policy name describes USB. The policy blocks optical media as well.

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices\*'
```

```
Deny_Read    : 1
Deny_Write   : 1
Deny_Execute : 1
PSChildName  : {53f5630d-b6bf-11d0-94f2-00a0c91efb8b}
```

### How it presented

Driver installation from mounted ISO media failed on the workstation. The symptom pointed everywhere except Group Policy:

| Observation | Suggested |
|---|---|
| `Get-CimInstance Win32_CDROMDrive` → `MediaLoaded: False`, blank drive letter | Hypervisor attachment problem or ghost device |
| `Get-PnpDevice -Class CDROM` → three entries, one `Status: Unknown` | Stale device enumeration |
| `Test-Path D:\` → Access is denied | Ambiguous — could be missing or blocked |
| `msiexec` → error `1619`, `-2147287035` (`STG_E_ACCESS_DENIED`) in verbose log | Package unreadable |

Considerable time went into removing ghost devices, re-attaching media at the hypervisor, and power-cycling the guest before the policy was checked.

`diskpart` gave the accurate picture the whole time:

```
DISKPART> list volume

  Volume ###  Ltr  Label        Fs     Type        Size     Status
  ----------  ---  -----------  -----  ----------  -------  ---------
  Volume 0     D   virtio-win-  CDFS   CD-ROM       753 MB  Healthy
```

Media loaded, drive letter assigned, filesystem readable, volume healthy. The disk was fine; access to it was denied by policy.

> **When `Win32_CDROMDrive` and `diskpart` disagree, trust `diskpart`.** WMI can report `MediaLoaded: False` on healthy mounted media, which sends troubleshooting toward the hypervisor when the problem is inside Windows.

> **`Test-Path` returns `False` for both "does not exist" and "access denied."** That conflation is what made the failure look like a missing device rather than a blocked one. For shares, `net use` returns a specific error code instead; for local paths, check the policy before the hardware.

### Resolution and status

The immediate workaround was to bypass optical media entirely — the installers were copied over SMB from a share on a domain controller instead.

> **Status: open.** The GPO needs narrowing so it denies USB mass storage without covering the CD/DVD device class. Blocking optical media was not the intent, and a policy whose name understates its reach is exactly the kind of configuration that produces confusing help desk tickets.

The generalizable point: a removable-storage restriction written against a broad device class will catch media the author was not thinking about. Enumerate the classes a policy actually covers rather than trusting its display name — including your own.

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

### `Deploy 7-Zip`: source package missing

> **Status: open.** The GPO's source path points at a share hosted on a domain controller that was subsequently rebuilt. The share was recreated after the rebuild; the MSI in it was not.
>
> Machines that already installed 7-Zip are unaffected. Any newly joined machine will fail this policy until the package is restored.

This is a second-order consequence worth stating plainly: rebuilding a host removes every file role it was quietly serving, not just the roles anyone thought to inventory. A Software Installation source path is exactly the kind of dependency that lives in a GPO nobody re-reads.

### Two things this generalizes to

**A `Test-Path` against a share tests the calling user, not the machine account.** Software Installation runs as SYSTEM and authenticates as `DOMAIN\MACHINE$`. An interactive `Test-Path` failure does not prove a permissions problem, and success does not prove the machine account has access.

**Absence of an error is diagnostic information.** Before investigating why an operation failed, establish whether it ran at all. `Deployment Count: 0` and 0 ms processing time answered that in this case, and pointed somewhere completely different from where the symptom suggested.

## `WSUS Client Settings` points at a role that no longer exists

> **Status: open.** WSUS was hosted on the domain controller that was rebuilt and was not reinstalled. The GPO still targets it.
>
> Clients apply the policy successfully and then fail to reach an update server. Resolution is to reinstall WSUS, repoint the GPO at a different server, or unlink it and let clients go direct to Windows Update.

A GPO that applies cleanly while pointing at a decommissioned target produces no policy-side error at all — `gpresult` reports success, and the failure surfaces later and elsewhere as clients silently stop patching.

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
>
> Worth retrying after WSUS is either reinstated or removed from the picture entirely, since the client is now pointed at an update server that does not answer.

**Diagnose FoD failures from CBS, not DISM.** `dism.log` reports only the propagated HRESULT; `CBS.log` names the actual condition.

## Quick reference

| Symptom | Check |
|---|---|
| GPO listed as applied, settings absent | Which config branch the settings live on; loopback if user-side on a computer-linked GPO |
| User policy shows `N/A` | Loopback processing; log off and back on, not `gpupdate` |
| Software Installation never runs | `Deployment Count`; force with `NoGPOListChanges`; reboot, not `gpupdate` |
| Software Installation fails only on new machines | Source package still present on the share it points at |
| Freshly joined machine gets no policy | Object in `CN=Computers`; move it or use `redircmp` |
| DC not receiving `Default Domain Controllers Policy` | Computer object moved out of the default DC container |
| `0x800f0954` on RSAT install | FoD GPO; not a WSUS fault |
| Optical or removable media inaccessible | `HKLM\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices` device class denials |
| `MediaLoaded: False` on media known to be attached | `diskpart list volume` before suspecting the hypervisor |
