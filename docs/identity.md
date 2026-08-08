# Identity and Access

Account model, Tier 1 delegation, group nesting, and domain password policy.

## Administrative accounts

| Account | OU | Privilege | Purpose |
|---|---|---|---|
| `Administrator` | `CN=Users` (default) | Domain Admins | Built-in. Break-glass only |
| `lamsec.admin` | `OU=Hokage,OU=Konoha` | Domain Admins | Named Tier 0 admin for daily work |
| `tier1.helpdesk` | `OU=Jonin,OU=Shinobi,OU=Konoha` | **None** | Tier 1 support; rights via delegation only |
| `svc-glpi` | `OU=ANBU,OU=Konoha` | None | LDAP bind account for the service desk |

### Why a named admin alongside the built-in

`Administrator` is present in every Active Directory domain, sits at a predictable RID of 500, and is the first account an attacker enumerates. It also produces no per-person attribution — every action in the security log reads as the same account regardless of who performed it.

`lamsec.admin` provides that attribution. The built-in stays as break-glass, which is also why it remains in the default `CN=Users` container rather than being moved into the tier structure: it is intentionally outside the model.

`Administrator` at RID 500 is exempt from account lockout on domain controllers by default, which makes it a genuine fallback if lockout policy locks everything else out.

### Why the Tier 1 account has no groups

`tier1.helpdesk` holds zero group memberships. Every capability comes from ACEs delegated on two specific OUs.

This is the correct boundary for a help desk role, and it has a practical benefit: the account's exact reach is auditable by reading two ACLs. An account in `Account Operators` or a custom group requires tracing group nesting and the group's own permissions across the directory to answer the same question.

## Tier 1 delegation

Delegated via the ADUC Delegation of Control Wizard on `OU=Chunin` and `OU=Genin` **individually** — not on the parent `OU=Shinobi`.

Delegating at the parent inherits down to `Jonin`, which would let Tier 1 reset the passwords of IT staff. That collapses the tier boundary the OU structure exists to create.

| ObjectType GUID | Right | Capability | Method |
|---|---|---|---|
| `00000000-0000-0000-0000-000000000000` | All properties | Read user information | Common task |
| `bf9679c0-0de6-11d0-a285-00aa003049e2` | `member` | Modify group membership | Common task |
| `bf967a0a-0de6-11d0-a285-00aa003049e2` | `pwdLastSet` | Force password change at next logon | Common task |
| `00299570-246d-11d0-a768-00aa006e0529` | Extended right | Reset password | Common task |
| `28630ebf-41d5-11d1-a9c1-0000f80367c1` | `lockoutTime` | **Unlock account** | Custom task |

### Account unlock is not a common task

The Delegation of Control Wizard has no "unlock account" option in its common-tasks list. Unlock requires a custom task delegation granting read and write on the `lockoutTime` property of User objects.

Account lockouts are typically the single largest Tier 1 ticket category. A delegation built entirely from the wizard's common tasks looks complete and cannot perform the most common operation the role exists to handle.

### Verifying the delegation

Read and write on the same attribute collapse into a single `ReadProperty, WriteProperty` ACE, so the expected count per OU is **5**, not 6.

```powershell
foreach ($ou in @("Genin","Chunin")) {
    (Get-Acl "AD:\OU=$ou,OU=Shinobi,OU=Konoha,DC=homelab,DC=local").Access |
        Where-Object { $_.IdentityReference -like "*tier1.helpdesk*" } |
        Select-Object ActiveDirectoryRights, ObjectType
}
```

Functional validation without RSAT:

```powershell
runas /user:HOMELAB\tier1.helpdesk "mmc dsa.msc"
```

### The wizard duplicates ACEs silently

Running the Delegation of Control Wizard twice against the same OU creates a second identical set of ACEs rather than erroring or updating in place.

An early pass produced 8 ACEs on `Chunin` and 0 on `Genin` — the wizard had been run against `Chunin` twice. Duplicates are functionally harmless but make the ACL difficult to read and audit.

Confirm the target OU in the wizard's title bar before proceeding, and verify ACE counts afterward.

## Group model — AGDLP

Accounts go into Global groups, Global groups go into Domain Local groups, Domain Local groups get Permissions.

| OU | Purpose | Scope | Naming |
|---|---|---|---|
| `Squads` | Role groups — job function | Global | `ROLE-*` |
| `Clans` | Security groups — resource access | Domain Local | `SEC-*` |

**Role groups (Global, 7)**
`ROLE-Engineering`, `ROLE-Finance`, `ROLE-HR`, `ROLE-Operations`, `ROLE-Marketing`, `ROLE-Facilities`, `ROLE-AllStaff`

**Security groups (Domain Local, 6)**
`SEC-Share-Engineering-RW`, `SEC-Share-Finance-RW`, `SEC-Share-HR-RW`, `SEC-Share-Company-RO`, `SEC-Printer-Main`, `SEC-VPN-Users`

Users are never members of a `SEC-*` group directly. They join a `ROLE-*` group; that role group is nested into whichever security groups the role requires; permissions are applied to the security group only.

The benefit shows up on a department change: move the user between two role groups and every downstream permission follows. Direct membership means auditing every ACL in the environment to find what needs changing.

### Verifying nesting

```powershell
Get-ADGroupMember -Identity "SEC-Share-Engineering-RW" -Recursive
```

Resolves to 5 users. `SEC-VPN-Users` resolves to 8, `SEC-Share-Company-RO` to 13 — with no direct user membership in any `SEC-*` group.

Confirmed in a real logon token on the workstation:

```powershell
whoami /groups
```

`ROLE-*` groups appear as type Group; `SEC-*` groups appear as Alias (Local Group). That is the expected representation for Domain Local groups in a token, and seeing it confirms the nesting resolved at logon rather than only looking correct in the directory.

### A group scoped to create a real scenario

`SEC-VPN-Users` deliberately excludes Operations, Marketing, and Facilities.

That produces a genuine "VPN won't connect" ticket whose correct resolution is a group membership change. The wrong resolutions — reinstalling the client, checking the firewall, rebuilding the profile — all fail, which is the point. A scenario where the obvious fix works teaches nothing.

## End users

13 accounts across two OUs, provisioned with `Title`, `Department`, `Company`, and `EmailAddress` populated so the service desk has real data to group and report on.

| OU | Count | Function |
|---|---|---|
| `Chunin` | 5 | Engineering |
| `Genin` | 8 | Standard users |

Departments in use: Engineering, Operations, Finance, HR, Marketing, Facilities.

Naming convention is `first.last`, lowercase.

## Password and lockout policy

Account lockout was **disabled by default** — `LockoutThreshold = 0` in the Default Domain Policy means accounts never lock regardless of failed attempts.

This is worth knowing before assuming a lockout scenario is reproducible. Check first:

```powershell
Get-ADDefaultDomainPasswordPolicy
```

Current configuration:

| Setting | Value | Effect |
|---|---|---|
| `LockoutThreshold` | 5 | Locks after 5 failed attempts |
| `LockoutDuration` | 00:30:00 | Auto-unlocks after 30 minutes |
| `LockoutObservationWindow` | 00:30:00 | Failed-attempt counter resets after 30 minutes |
| `MinPasswordLength` | 7 | |
| `ComplexityEnabled` | True | 3 of 4 character classes |

### Lockout events log on the PDC Emulator only

Event ID 4740 is written on the PDC Emulator — `LAB-DC2` here — not on whichever DC processed the failed authentication.

Querying any other DC returns nothing, which reads as "no lockouts occurred" rather than "wrong server."

```powershell
Get-WinEvent -ComputerName LAB-DC2 -FilterHashtable @{LogName='Security'; ID=4740} -MaxEvents 10
```

### Forced password change breaks non-interactive authentication

An account flagged "must change password at next logon" cannot authenticate anywhere that lacks a UI to present the change dialog. Active Directory refuses the bind outright.

This affects more than interactive logon:

- **RDP with NLA** refuses the connection with `Change user password before connecting`. Network Level Authentication validates credentials before a session exists, so there is no session in which to display a password change prompt.
- **`runas`** returns `1907: The user's password must be changed before signing in`.
- **LDAP-integrated applications** surface it as a generic bad-credential error with no indication of the real cause.

Check before troubleshooting an application's authentication configuration:

```powershell
Get-ADUser <user> -Properties pwdLastSet | Select-Object SamAccountName, @{N='MustChange';E={$_.pwdLastSet -eq 0}}
```

The workable path for a forced change on a domain-joined machine is the hypervisor console, which does not depend on RDP settings. Disabling NLA to work around it trades a real security control for convenience.
