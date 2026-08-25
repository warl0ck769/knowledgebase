---
title: Well-known SIDs
aliases: ["well-known SID", "built-in SID", "S-1-5-18", "NT AUTHORITY"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers"
verified: true
related: ["[[SID]]", "[[RID]]", "[[Important Users]]", "[[Privileged AD Groups]]", "[[Security Principal]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Well-known SIDs

> [!summary] One-liner
> Predefined SIDs with fixed values across all Windows systems — they identify built-in accounts, groups, and special identities that always exist regardless of the domain.

## Key well-known SIDs

### Universal (same on every Windows system)

| SID | Name | Notes |
|---|---|---|
| `S-1-0-0` | Nobody / Null | No security principal |
| `S-1-1-0` | Everyone | All users (including anonymous on older systems) |
| `S-1-2-0` | Local | Users logged on locally |
| `S-1-3-0` | Creator Owner | Placeholder; resolves to the object creator's SID |
| `S-1-5-7` | Anonymous Logon | Unauthenticated connections |
| `S-1-5-11` | Authenticated Users | All domain/local authenticated users |
| `S-1-5-18` | `NT AUTHORITY\SYSTEM` | LocalSystem — highest-privilege local account |
| `S-1-5-19` | `NT AUTHORITY\LOCAL SERVICE` | Reduced-privilege service account |
| `S-1-5-20` | `NT AUTHORITY\NETWORK SERVICE` | Like LocalService but uses machine creds on network |

### Domain-relative (RID appended to domain SID)

| RID | Name | Full SID example |
|---|---|---|
| **500** | Administrator | `S-1-5-21-<domain>-500` |
| **501** | Guest | `S-1-5-21-<domain>-501` |
| **502** | [[krbtgt account|krbtgt]] | `S-1-5-21-<domain>-502` |
| **512** | Domain Admins | `S-1-5-21-<domain>-512` |
| **513** | Domain Users | `S-1-5-21-<domain>-513` |
| **514** | Domain Guests | `S-1-5-21-<domain>-514` |
| **515** | Domain Computers | `S-1-5-21-<domain>-515` |
| **516** | Domain Controllers | `S-1-5-21-<domain>-516` |
| **518** | Schema Admins | Forest root domain only |
| **519** | Enterprise Admins | Forest root domain only |
| **520** | Group Policy Creator Owners | |
| **521** | Read-only Domain Controllers | |

### Built-in local groups

| SID | Name |
|---|---|
| `S-1-5-32-544` | BUILTIN\Administrators |
| `S-1-5-32-545` | BUILTIN\Users |
| `S-1-5-32-548` | BUILTIN\Account Operators |
| `S-1-5-32-549` | BUILTIN\Server Operators |
| `S-1-5-32-550` | BUILTIN\Print Operators |
| `S-1-5-32-551` | BUILTIN\Backup Operators |

## Red-team relevance
- **RID 500**: The built-in Administrator account bypasses many security filters (e.g., `LocalAccountTokenFilterPolicy` doesn't apply to RID 500 by default) — always a high-value target.
- **RID 502** ([[krbtgt account]]): Its hash creates [[Golden Ticket]]s.
- **`S-1-5-18` (SYSTEM)**: The target of most local privilege escalation; equivalent to root.
- **SID filtering**: [[Forest Trust Abuse]] relies on the fact that SID filtering blocks foreign SIDs with RID < 1000 across forest trusts — well-known SIDs from a trusted forest are stripped.

## Commands
```powershell
# Translate SID to name
([System.Security.Principal.SecurityIdentifier]"S-1-5-21-<domain>-500").Translate([System.Security.Principal.NTAccount])

# Translate name to SID
(Get-ADUser Administrator).SID

# List all well-known SIDs on a system
whoami /all
```

## Related
- Structure: [[SID]], [[RID]].
- Key accounts: [[Important Users]], [[krbtgt account]].
- Key groups: [[Privileged AD Groups]].
- Trust filtering: [[Forest Trust Abuse]], [[SID History Abuse]].
