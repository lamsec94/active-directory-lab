# Certificate Authority

Enterprise Root CA design, migration between domain controllers, and the rebuild sequencing that migration forces.

## Current state

| Component | Value |
|---|---|
| CA type | Enterprise Root CA |
| Common name | `homelab-CA` |
| Host | LAB-DC2 |
| Service | `CertSvc` — running |
| Root validity | 5 years |
| Issued wildcard | `*.homelab.local` |
| Algorithm | SHA256, 2048-bit RSA |
| Client trust | Domain members via Active Directory; non-domain hosts by manual import |

The CA issues the wildcard certificate that fronts every internal service through a reverse proxy. Domain-joined machines trust it automatically through AD; the Linux daily driver imports the root manually, and Firefox requires a separate import from the system store.

## The problem this section exists to document

**Active Directory replicates between domain controllers. A Certificate Authority does not.**

That asymmetry is easy to miss because both roles run on the same host and both feel like "directory infrastructure." They behave completely differently under failure:

- A domain controller can be demoted, wiped, and rebuilt with no data loss, because a second DC holds a full replica and the rebuilt host pulls a fresh copy on promotion.
- A CA has no equivalent mechanism. Destroy the host and the private key goes with it. Every certificate that CA ever issued becomes untrusted, and every client that trusted it needs a new root imported by hand.

The earlier build had the Enterprise Root CA on a domain controller running evaluation media with a fixed expiry date. That is an architectural problem rather than an operating system problem: a permanent, non-replicating, hard-to-rebuild role was pinned to a host that was guaranteed to need replacing.

Rebuilding that host therefore could not be a reinstall. It had to be a migration, sequenced so that at no point did the only copy of the CA key live on a machine that was about to be wiped.

## Migration procedure

### 1. Back up the CA

On the source host:

```powershell
New-Item -Path C:\CA-Backup -ItemType Directory -Force
Backup-CARoleService -Path C:\CA-Backup -Password (Read-Host -AsSecureString "Backup password")
reg export "HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration" C:\CA-Backup\CA-Config.reg
```

Three artifacts, all of which matter:

| Artifact | Contents |
|---|---|
| `<CAName>.p12` | The CA private key — the irreplaceable piece |
| `DataBase\<CAName>.edb` and logs | The issued-certificate database |
| `CA-Config.reg` | CRL paths, validity periods, extension configuration |

> `Backup-CARoleService` does not capture the registry configuration. Without the separate `reg export`, a restored CA comes up with default CRL distribution points and validity periods and behaves subtly wrong — issuing certificates that clients cannot revocation-check. Both artifacts must move together.

Copy the backup off the source host before proceeding. A backup that only exists on the machine being wiped is not a backup.

### 2. Remove the role from the source

```powershell
Remove-WindowsFeature ADCS-Cert-Authority
```

This leaves a pending reboot. That matters later — see [Sequencing](#sequencing).

### 3. Configure from restore on the destination

```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools

Install-AdcsCertificationAuthority -CAType EnterpriseRootCA `
    -CertFile "C:\CA-Backup\<CAName>.p12" `
    -CertFilePassword (Read-Host -AsSecureString "P12 password") -Force
```

Supplying `-CertFile` is what makes this a restore rather than a new CA. Without it, the cmdlet generates a fresh key pair and produces a CA with the same name and a completely different identity — which silently invalidates every certificate previously issued.

> The parameter is `-CertFile`, not `-CertFilePath`. And `-CACommonName` must **not** be supplied alongside it: when restoring from a `.p12` the name comes from the certificate, and passing both produces `Parameter set cannot be resolved using the specified named parameters`.

> `ErrorId 0` with an empty `ErrorString` is the success signature for this cmdlet. It does not look like success.

### 4. Restore the database and registry configuration

```powershell
Stop-Service certsvc
Restore-CARoleService -Path C:\CA-Backup -DatabaseOnly -Force
reg import C:\CA-Backup\CA-Config.reg
Start-Service certsvc
```

> `Restore-CARoleService -DatabaseOnly` does not accept `-Password`. That parameter belongs only to the `-KeyOnly` and full-restore parameter sets. `Get-Command Restore-CARoleService -Syntax` resolves this faster than guessing.

> The database restore requires `certsvc` stopped. Running it against a live service fails with `0x80070020` — the process cannot access the file because it is in use.

### 5. Verify identity, not availability

This is the step that actually proves the migration worked.

```powershell
Get-ChildItem Cert:\LocalMachine\CA |
    Where-Object { $_.Subject -like "*<CAName>*" } |
    Select-Object Subject, Thumbprint, SerialNumber, NotBefore, NotAfter
```

Run on both hosts and compare thumbprint and serial number. An exact match is definitive proof that the destination runs *the same CA identity* rather than a new CA that happens to share a name.

A service check does not prove this. `Get-Service certsvc` returning `Running` is equally true for a correctly restored CA and for a brand-new one built by accident — and the second case looks fine until certificates start failing validation across the environment.

```powershell
certutil -ping
```

`ICertRequest2 interface is alive` confirms the service answers enrollment requests.

> `certutil -ping` with no arguments tests for a *local* CA. On a host without one it correctly returns `No local Certification Authority; use -config option`. To test a remote CA: `certutil -config "<host>\<CAName>" -ping`.

### 6. Confirm downstream trust is unaffected

Because the CA identity was preserved, no client-side change was required. The reverse proxy serving every internal service continued presenting the same wildcard certificate, and every domain member continued trusting it.

That is only true because of step 5. Had the migration produced a new CA, every certificate in the environment would have needed reissuing and every trust store updating.

The reverse proxy in this environment uses statically uploaded certificate files with no live connection to the CA, so it required no reconfiguration at all — it neither knows nor cares which host runs `certsvc`. That only becomes relevant at the next certificate renewal.

## Sequencing

The order is not arbitrary. Each step depends on the one before it.

| # | Step | Why it is here and not elsewhere |
|---|---|---|
| 0 | Hypervisor snapshots of both DCs | The real undo button for every step that follows |
| 1 | Verify replication health and FSMO placement | Never begin a DC operation on broken replication |
| 2 | Back up the CA | Before anything destructive touches the source host |
| 3 | Restore and verify the CA on the destination | The source host is not disposable until this passes |
| 4 | Transfer FSMO roles off the source | A DC holding FSMO roles cannot be cleanly demoted |
| 5 | Demote the source | Requires the reboot left pending by the role removal |
| 6 | Rebuild, rejoin, promote | Directory data returns by replication |
| 7 | Verify replication in both directions | The operation is not finished until this is clean |

### FSMO roles must move before demotion

```powershell
netdom query fsmo
```

Run this before planning any DC removal. The assumption that "the other DC holds everything important" was wrong here — the host slated for rebuild still held the two forest-level roles, Schema Master and Domain Naming Master.

```powershell
Move-ADDirectoryServerOperationMasterRole -Identity "<target-dc>" `
    -OperationMasterRole SchemaMaster,DomainNamingMaster
```

Demoting a DC that still holds FSMO roles strands them, forcing an `ntdsutil` seizure and metadata cleanup afterward. A graceful transfer while the source is healthy costs one command.

### A pending role-removal reboot blocks demotion

Demotion failed on the first attempt with:

```
Verification of prerequisites for Domain Controller promotion failed.
Role change is in progress or this computer needs to be restarted.
```

The message names *promotion* during a *demotion*, which is misleading. The actual cause was the pending reboot left by the ADCS role removal, which had been deliberately deferred to keep the source host rollback-able. Once the CA was confirmed live on the destination, rebooting cleared it and demotion succeeded.

### Demotion

```powershell
Uninstall-ADDSDomainController -DemoteOperationMasterRole -RemoveApplicationPartitions -Force -Confirm:$false
```

Verified clean afterward from the surviving DC: `Get-ADDomainController -Filter *` returned only that host, and `repadmin /showrepl` showed zero inbound neighbours. No orphaned metadata, no cleanup required.

### In-place edition upgrade is not available

DISM edition conversion is blocked on a domain controller. Converting evaluation media to a licensed edition in place is not an option while the AD DS role is installed — the host must be demoted first, or rebuilt outright.

This is worth knowing before planning around it, because it removes the option that looks easiest.

## Post-promotion replication

```powershell
repadmin /replsummary
repadmin /showrepl
```

Immediately after the promotion reboot, `/replsummary` reported failures with error `1908` (could not find the domain controller for this domain). Fifteen minutes later it was clean in both directions across all five naming contexts.

> A trailing `N CONSECUTIVE FAILURES since <timestamp>` counter persists in `/showrepl` output after the underlying problem clears. Read the per-naming-context "Last attempt ... was successful" timestamps rather than the summary counter. A recent successful timestamp alongside a stale failure counter means the environment is healthy.

## Operational notes

**Stray VPN clients on a domain controller break DC location.** A Tailscale client installed on one DC advertised its CGNAT address to Active Directory, causing `Get-ADDomainController` to report that address instead of the host's real LAN IP. Both a CA restore and a DC promotion depend on correct DC location and SRV records, so this was found and removed during the pre-migration health check rather than discovered mid-operation.

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress
ipconfig /registerdns
Restart-Service netlogon
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address
```

Check every interface, not just the primary NIC, before assuming a real network fault.

**A DNS server with no forwarders fails only for external names.** The destination DC had no forwarders configured at all and was falling back to failing root hints. Internal resolution worked normally because the DC is authoritative for its own zone, so nothing appeared broken — the fault only surfaced when an unrelated operation needed internet access.

```powershell
Get-DnsServerForwarder
Add-DnsServerForwarder -IPAddress <resolver>
```

Check this on every DNS server after any rebuild.

**Certificate enrollment errors on RDP to a freshly joined machine are not CA faults.** *"A certification authority could not be contacted for authentication"* reflects incomplete certificate auto-enrollment on a machine that has just joined the domain. It resolves on its own once the machine settles, and entirely after DC promotion.

## What this demonstrates

The technically interesting part of this operation is not the commands. It is recognizing before starting that two roles living on the same host have completely different survivability properties, and sequencing the work so the non-replicating one is never the thing at risk.

The verification step matters for the same reason. Confirming a service is running proves the migration did *something*. Confirming the thumbprint and serial match proves it did the *right* thing — and that distinction is the difference between a working PKI and one that silently invalidates every certificate it ever issued.
