# OU Structure

The directory uses a tiered OU model that separates administrative accounts, service accounts, groups, users, and computers into distinct branches. The naming is themed; the structure underneath is conventional.

## Layout

```
Konoha (domain root container)
├── Hokage (Admins)
│   └── Sannin (Specialized Admins)
├── ANBU (Service Accounts)
├── Council (Groups)
│   ├── Clans (Security Groups)
│   └── Squads (Role Groups)
├── Shinobi (Users)
│   ├── Jonin (IT Staff)
│   ├── Chunin (Engineering)
│   └── Genin (Standard Users)
├── Fallen-Shinobi (Disabled / Terminated Users)
└── Compounds (Computers)      ← GPOs link here
    ├── Academy (New Computer Staging)
    ├── Towers (Servers)
    │   └── Branches (Member Servers)
    └── Training-Grounds (Workstations)
```

| Function | OU | Notes |
|---|---|---|
| Tier 0 admins | `Hokage` | Top-tier administrative accounts |
| Specialized admins | `Sannin` | Child of Hokage; inherits Hokage-linked policy |
| Service accounts | `ANBU` | Non-interactive accounts |
| Groups (parent) | `Council` | |
| Security groups | `Clans` | Domain Local, resource ACLs |
| Role groups | `Squads` | Global, job function |
| Users (parent) | `Shinobi` | |
| IT staff | `Jonin` | Tier 1 help desk account lives here |
| Engineering | `Chunin` | |
| Standard users | `Genin` | |
| Offboarded | `Fallen-Shinobi` | Staging for disabled accounts |
| Computers (parent) | `Compounds` | Single link point for all machine policy |
| New computer staging | `Academy` | |
| Servers | `Towers` | |
| Member servers | `Branches` | Child of Towers |
| Workstations | `Training-Grounds` | |

## Why tiered

The separation between `Hokage`, `Jonin`, and the end-user OUs is what makes delegation possible without over-granting. Tier 1 support is delegated on `Chunin` and `Genin` individually — never on the parent `Shinobi` — so help desk staff cannot reset the passwords of IT staff sitting in `Jonin`.

A flat model makes that boundary difficult to express. Delegating on a parent OU inherits downward, and once IT staff and standard users share a container there is no clean place to draw the line.

`Compounds` exists so every machine policy has one link point. Linking at the domain root would apply computer policy to domain controllers, which is rarely what you want.

## Migration from a flat model

The environment previously used a flat structure: `Corp-Computers`, `Corp-Users`, `Engineering`, and `IT Staff` directly under the domain root. That model was fully retired.

The migration was destructive and order-dependent. What made it work:

**Move objects before deleting the old containers.** The one live service account was moved into its new OU rather than recreated, preserving its SID and group memberships. Test and dummy objects were deleted outright.

**Unprotect before removing.** OUs with `ProtectedFromAccidentalDeletion` enabled refuse `Remove-ADOrganizationalUnit` until the flag is cleared.

**Verify GPO application at each step**, not at the end. Relinking policy and confirming with `gpresult /r` after each move catches problems while there is still a small change set to inspect.

### Domain controllers were not migrated

LAB-DC and LAB-DC2 remain in the default `OU=Domain Controllers,DC=homelab,DC=local`. Moving a DC's computer object out of that container breaks `Default Domain Controllers Policy` application.

This was discovered the hard way. Before the rebuild, LAB-DC's computer object had drifted into `OU=Servers,OU=Corp-Computers` — outside the default container — and had silently stopped receiving DC-tier policy. There is no error for this; the policy simply does not apply.

Verify placement:

```powershell
(Get-ADDomainController -Identity LAB-DC).ComputerObjectDN
```

Then confirm policy application:

```powershell
gpresult /r
```

Worth checking after any OU restructuring, and worth checking on an inherited environment before assuming DC policy is intact.

## Computer object placement on domain join

A freshly domain-joined machine lands in `CN=Computers`, not in the OU structure. `CN=Computers` is a **container**, not an OU, and containers cannot have GPOs linked to them.

The practical effect: the machine joins successfully, appears in the directory, and receives none of the policy linked at `Compounds`. Nothing errors.

Resolved by moving the object:

```powershell
Move-ADObject -Identity "CN=WORKSTATION,CN=Computers,DC=homelab,DC=local" `
    -TargetPath "OU=Training-Grounds,OU=Compounds,OU=Konoha,DC=homelab,DC=local"
```

All linked policies applied on the next refresh.

The permanent fix is to redirect the default container so joins land somewhere policy can reach:

```powershell
redircmp "OU=Academy,OU=Compounds,OU=Konoha,DC=homelab,DC=local"
```

`Academy` exists as a staging OU for exactly this purpose — new machines land there, get baseline policy, and move to `Training-Grounds` or `Towers` once configured.

## Downstream systems hold literal DNs

An OU restructure breaks every external system that stores a distinguished name, and those breakages surface long after the migration looks complete.

The service account move from `OU=IT Staff` to `OU=ANBU,OU=Konoha` was assessed as safe for the GLPI integration, on the basis that GLPI's user lookup is filter-based on `sAMAccountName` and therefore indifferent to OU placement. That assessment was correct about user lookup and wrong overall — the *bind account* is a separate setting stored as a literal DN, and it pointed at a deleted container for a month before anyone logged in against that source.

The error surfaced as `Unable to connect to the LDAP directory`, which reads as a network or service fault and sends you looking in entirely the wrong place.

Two takeaways:

- Prefer UPN form (`account@domain`) over full DN for LDAP bind accounts. Active Directory accepts it, and it survives arbitrary OU moves.
- After any OU restructuring, explicitly re-test every integrated application rather than reasoning about which ones should be affected.

Full writeup in the [GLPI repo](https://github.com/lamsec94/Glpi-itsm-deployment/blob/main/docs/ldap-auth.md).
