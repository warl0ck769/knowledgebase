---
title: FSMO Roles
aliases: ["FSMO", "Flexible Single Master Operations", "Operations Masters"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/fsmo-roles"
verified: true
related: ["[[Domain Controller (DC)]]", "[[Forest]]", "[[Domain]]", "[[AD Replication]]", "[[RID]]"]
created: 2026-06-07
updated: 2026-06-07
---

# FSMO Roles

> [!summary] One-liner
> Five special single-master roles assigned to specific DCs to handle operations that must be performed by exactly one DC to avoid conflicts — two are forest-wide, three are per-domain.

## Forest-wide roles (one per forest)

| Role | Holder | Purpose |
|---|---|---|
| **Schema Master** | One DC in the forest root domain | Only DC allowed to write to the AD Schema partition (attribute/class definitions) |
| **Domain Naming Master** | One DC in the forest root domain | Handles adding/removing domains from the [[Forest]] |

## Domain-wide roles (one per domain)

| Role | Holder | Purpose |
|---|---|---|
| **PDC Emulator** | One DC per [[Domain]] | Authoritative for password changes (urgent replication), time sync source, GPO editing target, account lockout processing |
| **RID Master** | One DC per [[Domain]] | Allocates [[RID]] pools to DCs so each can create new objects with unique SIDs |
| **Infrastructure Master** | One DC per [[Domain]] | Updates cross-domain group-to-user references (phantom records); less relevant if all DCs are also [[Global Catalog|GCs]] |

## Red-team relevance
- **PDC Emulator** is the first DC contacted for password validation failures (password changes replicate to PDCe urgently). Targeting it disrupts authentication.
- **RID Master** exhaustion: if the RID pool is exhausted and the RID Master is unavailable, no new objects can be created in that domain.
- **Schema Master**: compromising this DC allows schema modification — adding attributes to store backdoor data (extremely stealthy, extremely impactful).
- Finding FSMO holders reveals the most critical DCs to target (or avoid alerting):

```powershell
# Query all FSMO role holders
netdom query fsmo

# PowerShell
Get-ADForest | Select SchemaMaster, DomainNamingMaster
Get-ADDomain | Select PDCEmulator, RIDMaster, InfrastructureMaster
```

## Related
- Hosted on: [[Domain Controller (DC)]].
- RID allocation: [[RID]], [[SID]].
- Replication: [[AD Replication]].
