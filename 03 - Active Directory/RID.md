---
title: RID
aliases: ["Relative Identifier", "RID pool", "RID Master"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers"
verified: true
related: ["[[SID]]", "[[Well-known SIDs]]", "[[FSMO Roles]]", "[[Domain Controller (DC)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# RID (Relative Identifier)

> [!summary] One-liner
> The final component of a [[SID]] that uniquely identifies a security principal within its domain — allocated in pools by the [[FSMO Roles|RID Master]] to each DC.

## Structure within a SID
```
S-1-5-21-<domain identifier>-<RID>
                                ^^^
                                This is the RID
```
Example: `S-1-5-21-3623811015-3361044348-30300820-1104` → RID = **1104**

## RID allocation
- The [[FSMO Roles|RID Master]] (one per domain) allocates **pools of RIDs** (default: 500 per batch) to each DC.
- When a DC creates a new object (user, group, computer), it assigns the next RID from its pool.
- This avoids collisions: no two DCs assign the same RID because they draw from non-overlapping pools.

## Well-known RIDs
RIDs below 1000 are reserved for built-in accounts/groups — see [[Well-known SIDs]].

| RID | Account |
|---|---|
| 500 | Administrator |
| 501 | Guest |
| 502 | krbtgt |
| 512 | Domain Admins |
| 513 | Domain Users |
| 515 | Domain Computers |
| 516 | Domain Controllers |
| 519 | Enterprise Admins |

## Red-team relevance
- **RID 500** (built-in Administrator): Exempt from `LocalAccountTokenFilterPolicy` filtering — [[Pass-the-Hash]] always works for this account even when LATFP is not set.
- **RID cycling**: Enumerate domain users by brute-forcing RIDs via `lookupsid` (Impacket) or `LookupAccountSid` — works even with restricted LDAP access.
- **SID filtering**: Cross-forest trusts filter SIDs with RID < 1000 ([[Forest Trust Abuse]]) — well-known RIDs from a foreign forest are stripped.

```bash
# RID cycling with Impacket
lookupsid.py <DOMAIN>/<USER>:<PASS>@<DC> 10000
```

## Related
- Part of: [[SID]].
- Built-in values: [[Well-known SIDs]].
- Allocation: [[FSMO Roles]] (RID Master).
