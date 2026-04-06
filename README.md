# Active Directory Lab

Windows Server 2022 Active Directory homelab demonstrating enterprise OU design, GPO management, PowerShell provisioning, and dual-identity integration with FreeIPA.

---

## Environment

| Component | Details |
|---|---|
| Domain Controller | Windows Server 2022 |
| Domain | `homelab.local` |
| Workstation | Windows 11 Pro (domain-joined) |
| Linux Identity | FreeIPA (AlmaLinux) — parallel identity provider |
| Virtualization | Proxmox VE, two-node cluster |

---

## OU Structure

Designed to reflect a realistic enterprise identity hierarchy with separation between user types and computer roles.

```
homelab.local
├── Domain Controllers
├── Corp-Computers
│   ├── Servers
│   └── Workstations
├── Corp-Users
│   ├── IT
│   └── Standard-Users
├── Engineering
├── IT Staff
```

### Design Rationale

- **Corp-Computers** separates servers from workstations — allows GPOs to target machine types independently
- **Corp-Users** separates IT staff from standard users — different sudo/access policies per group
- **Engineering** isolated for future role-specific policy application
- **IT Staff** at top level — help desk and admin accounts with distinct policy scope
- Domain Controllers in default OU — managed separately from general computer policy

---

## User Accounts

| User | OU | Role |
|---|---|---|
| IT Admin | Corp-Users → IT | IT administrator account |
| Help Desk Tech | IT Staff | Help desk access |
| Engineer User | Engineering | Engineering role account |
| John Smith | Corp-Users → Standard-Users | Standard user |
| Sarah Johnson | Corp-Users → Standard-Users | Standard user |
| Mike Davis | Corp-Users → Standard-Users | Standard user |
| Lisa Martinez | Corp-Users → Standard-Users | Standard user |
| Guest User | Corp-Users → Standard-Users | Restricted guest account |

---

## GPO Configuration

### Linked GPOs

| GPO | Linked To | Purpose |
|---|---|---|
| Default Domain Policy | Domain root | Password policy, account lockout |
| Security - Screen Lock 5 Minutes | Corp-Computers | Idle screen lock enforcement |
| Deploy 7-Zip | Corp-Computers | Software deployment via GPO |
| WSUS Client Settings | Corp-Computers | Windows Update server targeting |
| Security - Block USB Storage | Corp-Computers | Removable media restriction |

### GPO Design Notes

- All computer policies scoped to `Corp-Computers` — DCs are excluded from workstation/server GPOs
- `Default Domain Policy` is the only domain-root linked GPO — keeps root clean
- USB storage blocked at the computer level — applies regardless of logged-in user
- Screen lock enforced via computer policy — not user policy, so it applies to all sessions

---

## PowerShell Provisioning

### OU Structure

```powershell
# Create top-level OUs
New-ADOrganizationalUnit -Name "Corp-Computers" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Corp-Users" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Engineering" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "IT Staff" -Path "DC=homelab,DC=local"

# Create sub-OUs
New-ADOrganizationalUnit -Name "Servers" -Path "OU=Corp-Computers,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Corp-Computers,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "IT" -Path "OU=Corp-Users,DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Standard-Users" -Path "OU=Corp-Users,DC=homelab,DC=local"
```

### User Placement

```powershell
# Move users to correct OUs
Move-ADObject -Identity "CN=IT Admin,DC=homelab,DC=local" `
  -TargetPath "OU=IT,OU=Corp-Users,DC=homelab,DC=local"

Move-ADObject -Identity "CN=Engineer User,DC=homelab,DC=local" `
  -TargetPath "OU=Engineering,DC=homelab,DC=local"

Move-ADObject -Identity "CN=Guest User,CN=Users,DC=homelab,DC=local" `
  -TargetPath "OU=Standard-Users,OU=Corp-Users,DC=homelab,DC=local"
```

### Computer Placement

```powershell
# Move workstation to correct OU
Move-ADObject -Identity "CN=WINDOWS11-VM,CN=Computers,DC=homelab,DC=local" `
  -TargetPath "OU=Workstations,OU=Corp-Computers,DC=homelab,DC=local"
```

### GPO Links

```powershell
# Scope computer GPOs to Corp-Computers only
New-GPLink -Name "Deploy 7-Zip" -Target "OU=Corp-Computers,DC=homelab,DC=local"
New-GPLink -Name "WSUS Client Settings" -Target "OU=Corp-Computers,DC=homelab,DC=local"
New-GPLink -Name "Security - Block USB Storage" -Target "OU=Corp-Computers,DC=homelab,DC=local"

# Remove broad domain-root links
Remove-GPLink -Name "Deploy 7-Zip" -Target "DC=homelab,DC=local"
Remove-GPLink -Name "WSUS Client Settings" -Target "DC=homelab,DC=local"
```

### Verify Final State

```powershell
# Confirm OU structure
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize

# Confirm user placement
Get-ADUser -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize

# Confirm computer placement
Get-ADComputer -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize

# Confirm GPO links
Get-GPInheritance -Target "DC=homelab,DC=local" | Select-Object -ExpandProperty GpoLinks
Get-GPInheritance -Target "OU=Corp-Computers,DC=homelab,DC=local" | Select-Object -ExpandProperty GpoLinks
```

---

## Dual Identity — AD + FreeIPA

The lab runs two parallel identity providers targeting different host types.

### Architecture

```
Windows hosts  →  Active Directory (homelab.local)
Linux hosts    →  FreeIPA (ipa.homelab.local)
```

### Why Dual Identity

Enterprise environments rarely run a single identity provider across all platforms. This lab reflects that reality:

- Windows workstations and the DC are managed entirely through AD — GPOs, software deployment, and authentication
- Linux VMs are enrolled in FreeIPA — HBAC rules control SSH access, sudo rules are centrally managed
- Both systems are independent — no AD-IPA trust configured, intentionally kept separate to demonstrate both stacks

### FreeIPA Configuration

| Component | Configuration |
|---|---|
| HBAC | `allow_admins` rule — admin account only, sshd service |
| HBAC | `allow_all` default rule disabled |
| Sudo | `allow_admin_sudo` — full sudo with password required |
| Enrolled hosts | IPA server, Ansible controller |

### DNS

- AD DNS handles `homelab.local` resolution — conditional forward configured on AdGuard Home
- FreeIPA DNS handles `ipa.homelab.local`
- All VLANs resolve through AdGuard Home as primary DNS

---

## Key Takeaways

- GPO scope matters — linking computer policies at domain root applies them to DCs, which is incorrect and a common mistake
- OU design should reflect policy boundaries, not just org chart structure
- Dual identity stacks are common in hybrid environments — AD for Windows, LDAP/IPA for Linux is a standard enterprise pattern
- PowerShell provisioning makes AD configuration reproducible and auditable
