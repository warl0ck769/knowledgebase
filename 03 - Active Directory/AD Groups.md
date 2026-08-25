---
title: AD Groups
aliases: ["Groups", "Group Scope", "Universal group", "Global group", "Domain Local group"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Privileged AD Groups]]", "[[AD User Object]]", "[[SID]]", "[[Forest]]", "[[Domain]]", "[[Domain and Forest Trusts]]", "[[PowerView]]"]
created: 2026-06-06
updated: 2026-06-07
---

# AD Groups

> [!summary] One-liner
> Groups bundle permissions so admins manage access at the group level; each group is a DB object (with SamAccountName + SID), and its **scope** controls where members and permissions can come from.

## What it is
Instead of granting access per-user, admins add users to **groups** and manage permissions on the group. Groups are stored in the domain database and identified by **`sAMAccountName`** or **[[SID]]**, just like [[AD User Object|users]].

```powershell
# Enumerate group names
Get-ADGroup -Filter * | select SamAccountName
# -> Administrators, Domain Admins, Enterprise Admins, Schema Admins,
#    Protected Users, DnsAdmins, Cert Publishers, Key Admins, ...
```

## Group scope (where members & permissions can come from)
| Scope | Members can come from | Grants permissions in | Example |
|---|---|---|---|
| **Universal** | Same **forest** | Same forest or **trusted forests** | Enterprise Admins |
| **Global** | **Own domain** only | Same-forest domains, or trusting domains/forests | Domain Admins |
| **Domain Local** | Own domain or **any trusted domain** | **Own domain only** | Administrators |

> Domain groups/users can also be members of a **computer's local groups** — e.g. **Domain Admins** is added by default to every machine's local **Administrators** group (a key reason DA = local admin everywhere).

## Why a red teamer cares
- Group membership = the fastest map of "who can do what." See [[Privileged AD Groups]] for the high-value targets.
- Scope rules explain **cross-domain** reach of a compromised group ([[Domain and Forest Trusts]]).

## More from harmj0y (group scoping & enumeration blind spots)

harmj0y's "A Pentester's Guide to Group Scoping" digs into how the three scopes affect **nesting**, **global-catalog replication**, and therefore what an attacker (or defender) can actually see across a [[Forest]].

### Precise nesting / membership / permission rules
| Scope | Can be nested in | Can contain | Grants permissions in | Memberships replicated to Global Catalog? |
|---|---|---|---|---|
| **Domain Local** | Only other domain local groups (same domain) | Global groups, universal groups, **foreign trust members** | **Same domain only** | **No** |
| **Global** | Universal and domain local groups | Only global groups from the **same domain** | Any domain | **No** |
| **Universal** | Domain local and other universal groups | Global groups and other universal groups | Any domain or **forest** | **Yes** |

- **Domain local cannot nest into global/universal** — an intentional design to stop accidental privilege escalation.
- **Global cannot be added to groups in other domains** (its members must be same-domain).
- **Foreign trust users (external/forest trust) can only be added to domain local groups.** Adding them creates a **foreign security principal** object under `CN=ForeignSecurityPrincipals,DC=domain,DC=com`.

### The Global Catalog determines forest-wide visibility
- **Universal group memberships ARE replicated to the Global Catalog**, so you can enumerate the members of *any* universal group for *any* domain in the forest by querying a Global Catalog (a DC in your own domain) — no cross-domain traffic or referrals needed. This is also a **forest-wide blast radius**: a compromised universal group reaches the whole forest.
- **Domain local and global group memberships are NOT replicated** to the Global Catalog, so they are invisible forest-wide; you must bind to the specific domain to see them.

### `memberOf` is unreliable — known gaps
The `memberOf` back-link attribute does **not** tell the whole story:
- **Domain local** memberships won't populate when you query a *different* domain's Global Catalog.
- **Global** memberships may not appear consistently in forest-wide queries.
- **Foreign security principal** memberships are **never** reflected on the foreign principal's `memberOf` — when an external user is added to a domain local group, the group's `member` property updates but the foreign user's `memberOf` back-link cannot be calculated.
- Net effect: **results vary depending on which domain / Global Catalog you bind to**, creating enumeration blind spots for both offense and detection.

### Commands & tools (PowerView)
```powershell
# Filter groups by scope (Not- variants also exist: NotDomainLocal, NotGlobal, NotUniversal)
Get-DomainGroup -GroupScope DomainLocal
Get-DomainGroup -GroupScope Global
Get-DomainGroup -GroupScope Universal

# Filter by group property
Get-DomainGroup -GroupProperty Security        # security (vs distribution) groups
Get-DomainGroup -GroupProperty Distribution
Get-DomainGroup -GroupProperty CreatedBySystem

# Enumerate group members (recurse to resolve nested groups)
Get-DomainGroupMember -Identity "<GROUP>" -Recurse

# Tease out members that live in OTHER (trusted) domains / forests
Get-DomainForeignGroupMember -Domain <DOMAIN.FQDN>

# Locate Global Catalogs, then query the GC directly to enumerate forest-wide
Get-ForestGlobalCatalog
Get-DomainObject -SearchBase "GC://<DOMAIN.COM>"
```

```text
# LDAP groupType bit matching (rule OID 1.2.840.113556.1.4.803)
(groupType:1.2.840.113556.1.4.803:=4)            # Domain Local
(groupType:1.2.840.113556.1.4.803:=2)            # Global
(groupType:1.2.840.113556.1.4.803:=8)            # Universal
(groupType:1.2.840.113556.1.4.803:=2147483648)   # Security (vs Distribution)
```

### OPSEC / takeaways
- Prefer **Global Catalog** queries to enumerate **universal** groups forest-wide from your own domain — avoids cross-domain queries / referrals that may be noisier.
- Don't trust `memberOf` alone; for full effective membership resolve nested groups (`-Recurse`) and explicitly enumerate **foreign** members (`Get-DomainForeignGroupMember`) since domain local and foreign-principal memberships are missing from the Global Catalog and back-links.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Groups" / "Group Scope"
- [[Source - harmj0y blog]] — "A Pentester's Guide to Group Scoping" — https://blog.harmj0y.net/activedirectory/a-pentesters-guide-to-group-scoping/
