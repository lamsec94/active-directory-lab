# Active Directory Lab

A two-domain-controller Active Directory environment built to enterprise conventions: a tiered OU model, delegated Tier 1 administration with no group membership, AGDLP group nesting, and Group Policy scoped to a dedicated computer branch.

The directory backs a GLPI service desk and a set of reproducible Tier 1 support scenarios. Where a design decision has a real-world reason behind it, that reason is documented alongside it.

## Environment

| Host | OS | Role |
|---|---|---|
| LAB-DC | Windows Server 2025 Standard | Domain controller, DNS, Global Catalog, software share |
| LAB-DC2 | Windows Server 2025 Standard | Domain controller, DNS, Global Catalog, all five FSMO roles, Enterprise Root CA |
| WORKSTATION | Windows 11 Pro (25H2) | Domain-joined client for policy validation and ticket reproduction |

Domain: `homelab.local`. Both DCs replicate cleanly across all five naming contexts and hold full replicas. Both are Global Catalogs.

Virtualized on a two-node Proxmox cluster with VLAN segmentation and an OPNsense firewall boundary. The Windows host count is capped at three by deliberate decision — see [Constraints](#constraints).

## What is here

| Document | Covers |
|---|---|
| [docs/ou-structure.md](docs/ou-structure.md) | Tiered OU model, migration from a flat structure, computer object placement |
| [docs/identity.md](docs/identity.md) | Admin account model, Tier 1 delegation, AGDLP groups, password and lockout policy |
| [docs/group-policy.md](docs/group-policy.md) | Five GPOs, loopback processing, software deployment troubleshooting |
| [docs/certificate-authority.md](docs/certificate-authority.md) | Enterprise CA migration between domain controllers, identity verification, rebuild sequencing |

## Design principles

**Named admin accounts over the built-in.** `Administrator` exists in every domain, sits at a known RID, and produces no per-person audit attribution when used for daily work. A named Tier 0 account handles administration; the built-in is reserved as break-glass.

**Delegation over group membership.** The Tier 1 help desk account holds zero group memberships. Every capability it has comes from explicit ACEs on the two OUs holding end users. That is the correct real-world boundary, and it makes the account's exact reach auditable.

**Policy scoped to a computer branch.** All machine policies link at a single computer OU rather than the domain root. Domain controllers stay in their default container so they continue to receive `Default Domain Controllers Policy`.

**Scenarios that require the right fix.** Group membership and policy are arranged so support scenarios have a correct resolution rather than a workaround. A VPN group that deliberately excludes three departments produces a "VPN won't connect" ticket whose real answer is a membership change, not a client-side fix.

**Nothing permanent on a host with an expiry date.** Both domain controllers run permanently licensed Windows Server 2025. An earlier build placed the Enterprise Root CA on a domain controller running evaluation media, which coupled a permanent, non-replicating role to a host with a hard shutdown date. Resolving that is documented in [docs/certificate-authority.md](docs/certificate-authority.md).

## Domain controller rebuild

LAB-DC was rebuilt in place on licensed media. The directory came through intact because Active Directory replicates — the second DC held a full replica throughout, and the rebuilt host pulled a fresh copy on promotion.

The Certificate Authority did not have that property, which is what made the operation a sequenced migration rather than a reinstall:

1. Verify replication health and FSMO placement before planning anything
2. Back up the CA private key, certificate database, and registry configuration
3. Migrate the CA to the surviving DC and verify by thumbprint and serial
4. Transfer all FSMO roles off the host being rebuilt
5. Demote cleanly, then rebuild
6. Rejoin, promote, verify replication in both directions

Every step was gated on verifying the previous one. Full detail in [docs/certificate-authority.md](docs/certificate-authority.md).

## Constraints

The environment is capped at three Windows hosts. There is no member server, which means printer, shared-drive, and mapped-drive scenarios are not reproducible. The supporting group model for those exists and is correctly scoped; it has no resource to point at.

That gap is documented rather than papered over. Co-locating File and Print Services on a domain controller was considered and rejected — running those roles on a DC is not enterprise practice, and practicing against a knowingly non-standard configuration is worse than acknowledging the limitation.

## Related

- [glpi-itsm-deployment](https://github.com/lamsec94/Glpi-itsm-deployment) — the service desk this directory authenticates
- [homelab-network-documentation](https://github.com/lamsec94/Network-documentation) — VLAN design and firewall policy
