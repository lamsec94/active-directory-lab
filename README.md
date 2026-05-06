# Active Directory Lab

Windows Server Active Directory lab documenting enterprise-style identity design,
Group Policy scoping, PowerShell-based provisioning, multi-DC operations, and
internal certificate services. This repository focuses on how Active Directory
was structured and operated as part of a broader Windows and Linux homelab.

> **Built by:** Lamar Scott | **GitHub:** [lamsec94](https://github.com/lamsec94) | **Last updated:** May 2026

---

## Environment

| Component | Details |
|---|---|
| Primary Domain Controller | Windows Server 2022 |
| Secondary Domain Controller | Windows Server 2025 |
| Domain | `homelab.local` |
| Workstation | Windows 11 Pro (domain-joined) |
| Certificate Services | ADCS Enterprise Root CA |
| Linux Identity | FreeIPA (parallel identity provider) |
| Virtualization | Proxmox VE, two-node cluster |

This environment was designed to go beyond a basic single-DC lab. In addition to
centralized authentication and GPO management, it includes domain replication,
internal PKI, and LDAP-backed service authentication for selected applications.

---

## Domain Architecture

The Active Directory environment is built around a primary and secondary domain controller
to support replication validation, operational resilience, and more realistic Windows
infrastructure workflows.

| Component | Role |
|---|---|
| LAB-DC | Primary domain controller, DNS, ADCS Enterprise Root CA |
| LAB-DC2 | Secondary domain controller for replication and directory redundancy |
| Windows 11 Pro | Domain-joined workstation for user policy and software deployment testing |

This design better reflects real-world operations than a single-controller lab and creates
a platform for testing identity, policy, DNS, certificate trust, and service integration together.

---

## OU Structure

Designed to reflect a realistic enterprise identity hierarchy with separation between user types and computer roles.

```text
homelab.local
├── Domain Controllers
├── Corp-Computers
│   ├── Servers
│   └── Workstations
├── Corp-Users
│   ├── IT
│   └── Standard-Users
├── Engineering
└── IT Staff
```

### Design Rationale

- **Corp-Computers** separates servers from workstations so computer policies can be scoped cleanly.
- **Corp-Users** separates IT staff from standard users to support different access and policy requirements.
- **Engineering** is isolated for role-specific policy targeting.
- **IT Staff** remains distinct for administrative and support workflows.
- **Domain Controllers** stay in the default OU so they are not affected by workstation-oriented computer policy.

---

## GPO Configuration

### Linked GPOs

| GPO | Linked To | Purpose |
|---|---|---|
| Default Domain Policy | Domain root | Password policy and account lockout baseline |
| Security - Screen Lock 5 Minutes | Corp-Computers | Idle session lock enforcement |
| Deploy 7-Zip | Corp-Computers | Software deployment through Group Policy |
| WSUS Client Settings | Corp-Computers | Windows Update targeting and patch policy |
| Security - Block USB Storage | Corp-Computers | Removable media restriction |

### GPO Design Notes

- All computer policies are scoped to `Corp-Computers`, which keeps them off domain controllers.
- `Default Domain Policy` remains the only domain-root linked GPO.
- USB storage blocking is enforced as a computer policy so it applies consistently.
- Screen lock is applied at the computer level rather than depending on user context.

This keeps policy scope intentional and avoids the common mistake of linking workstation controls too broadly.

---

## PowerShell Provisioning

PowerShell was used to make the directory layout reproducible and easier to validate after changes.

### OU Structure

```powershell
New-ADOrganizationalUnit -Name "Corp-Computers" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Corp-Users" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Engineering" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "IT Staff" -Path "DC=homelab,DC=local"

New-ADOrganizationalUnit -Name "Servers" -Path "OU=Corp-Computers,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Corp-Computers,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "IT" -Path "OU=Corp-Users,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Standard-Users" -Path "OU=Corp-Users,DC=homelab,DC=local"
```

### GPO Links

```powershell
New-GPLink -Name "Deploy 7-Zip" -Target "OU=Corp-Computers,DC=homelab,DC=local"
New-GPLink -Name "WSUS Client Settings" -Target "OU=Corp-Computers,DC=homelab,DC=local"
New-GPLink -Name "Security - Block USB Storage" -Target "OU=Corp-Computers,DC=homelab,DC=local"

Remove-GPLink -Name "Deploy 7-Zip" -Target "DC=homelab,DC=local"
Remove-GPLink -Name "WSUS Client Settings" -Target "DC=homelab,DC=local"
```

### Validation

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize
Get-ADUser -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize
Get-ADComputer -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize
Get-GPInheritance -Target "DC=homelab,DC=local" | Select-Object -ExpandProperty GpoLinks
Get-GPInheritance -Target "OU=Corp-Computers,DC=homelab,DC=local" | Select-Object -ExpandProperty GpoLinks
```

---

## Certificate Services

Active Directory Certificate Services was deployed on the primary domain controller as an
Enterprise Root CA named `homelab-CA`. This CA issues trust used across internal services
and supports the broader move to HTTPS across the homelab.

| PKI Component | Purpose |
|---|---|
| Enterprise Root CA | Internal trust anchor for Windows and internal services |
| Wildcard certificate | Shared certificate for internal reverse-proxied services |
| CA trust distribution | Certificate trust validation across administrative endpoints and browsers |

This adds meaningful enterprise depth to the AD deployment because certificate trust,
service identity, and internal HTTPS are now tied back to Windows infrastructure rather than
being handled as isolated one-off tasks.

---

## LDAP Service Integration

Active Directory is also used as an identity backend for internal applications.
A dedicated read-only LDAP bind account was created for GLPI so the ITSM platform could
authenticate domain users without requiring elevated directory permissions.

| Integration | Purpose |
|---|---|
| GLPI LDAP bind account | Read-only directory queries for service authentication |
| Imported domain users | Domain-backed authentication and access inside GLPI |

This demonstrates that the directory is being used as a real shared identity service,
not just a login source for a single Windows client.

---

## Dual Identity

The lab runs two parallel identity systems:

```text
Windows hosts  →  Active Directory (homelab.local)
Linux hosts    →  FreeIPA
```

This reflects a more realistic mixed-platform environment where Windows and Linux systems
do not always rely on a single identity provider. Active Directory handles Windows-centric
policy and authentication, while FreeIPA manages Linux host enrollment, HBAC, and sudo policy.

---

## Operational Lessons

- GPO scope matters; broad links at the domain root create avoidable policy risk.
- OU design should follow management and policy boundaries, not just naming preference.
- A second domain controller adds real operational value by forcing replication validation and healthier directory design.
- PKI depends on accurate time; certificate issuance and trust can fail immediately when the CA clock is incorrect.
- Least-privilege LDAP service accounts are a better pattern than reusing administrative credentials for application binds.

These are the kinds of details that make the lab more representative of real Windows infrastructure work.

---

## Skills Demonstrated

`Active Directory` `Windows Server 2022` `Windows Server 2025` `OU Design`
`Group Policy` `PowerShell` `ADCS` `PKI` `DNS` `LDAP`
`Identity architecture` `Windows administration` `Infrastructure documentation`
