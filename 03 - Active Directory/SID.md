---
title: SID
aliases: ["Security Identifier", "RID", "Relative Identifier", "objectSID"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Security Principal]]", "[[Domain]]", "[[Well-known SIDs]]", "[[Golden Ticket]]", "[[SID History Abuse]]"]
created: 2026-06-06
updated: 2026-06-06
---

# SID

> [!summary] One-liner
> A binary value that uniquely identifies a security principal; it ends in a RID and is prefixed by the domain SID.

## What it is
The **Security Identifier (SID)** uniquely identifies a [[Security Principal]] (user, computer, group). Format:

```
S-1-5-21-X-Y-Z-RID
```
| Part | Meaning |
|---|---|
| `S-1-5` | Authority — **NT Authority** |
| `21` | Non-unique authority |
| `X-Y-Z` | **Domain identifier** (the [[Domain]] SID) |
| `RID` | **Relative Identifier** — unique within the domain |

Example: `S-1-5-21-1372086773-2238746523-2939299801-1103` → domain SID + RID **`1103`**.

## RID (Relative Identifier)
The last number. Well-known RIDs: **500** = built-in Administrator, **501** = Guest, **502** = krbtgt, **512** = Domain Admins, **513** = Domain Users, **519** = Enterprise Admins. (Full list: [[Well-known SIDs]].)

## Why a red teamer cares
- Forging tickets needs the **domain SID** ([[Golden Ticket]]).
- **[[SID History Abuse]]** injects a privileged SID (e.g. Enterprise Admins of the parent) into a token to escalate across domains.
- RIDs let you identify privileged accounts at a glance during enumeration.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Database → Principals → SID"
