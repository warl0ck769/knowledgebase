---
title: Global Catalog
aliases: ["GC", "Global Catalog server", "GC port 3268"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/ad/global-catalog"
verified: true
related: ["[[Domain Controller (DC)]]", "[[Forest]]", "[[LDAP]]", "[[AD Database (NTDS.dit)]]", "[[AD Replication]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Global Catalog (GC)

> [!summary] One-liner
> A read-only **partial copy of all objects in the entire forest** hosted on designated DCs, enabling cross-domain searches and Universal Group membership resolution without contacting every domain.

## What it stores
- **Every object** from every domain in the [[Forest]] — but only a **subset of attributes** (the "partial attribute set" / PAS).
- The PAS is defined in the [[AD Database (NTDS.dit)|AD Schema]]: attributes with `isMemberOfPartialAttributeSet = TRUE`.
- Full attribute data is only available from the object's home domain DC.

## Ports

| Protocol | Port | Use |
|---|---|---|
| LDAP (GC) | **3268** | Unencrypted GC queries |
| LDAPS (GC) | **3269** | TLS-encrypted GC queries |

Standard LDAP (389/636) queries only the local domain; GC ports return forest-wide results.

## Why it exists
- **Universal Group membership**: At logon, the DC checks the GC to resolve Universal Group memberships from other domains — if the GC is unavailable, logon can fail (unless Universal Group Membership Caching is enabled).
- **Forest-wide search**: Applications and tools (e.g., Exchange, [[BloodHound]]) query the GC to find objects across domains.
- **[[Kerberos]] referrals**: When a client requests a ticket for a service in another domain, the DC uses the GC to locate the target.

## Red-team relevance
- **Cross-domain enumeration**: Query port 3268 to enumerate users, groups, SPNs, and trusts across the entire forest in one shot.
- **[[AD as C2 Channel]]**: Attributes in the GC partial attribute set (like `mSMQSignCertificates`) replicate forest-wide, making them usable as a dead-drop from any domain.
- **SPN scanning**: [[Kerberoasting]] across domains — query GC for all SPNs in the forest.

```powershell
# Forest-wide LDAP search via GC
Get-ADUser -Filter * -Server "<DC>:3268" -Properties servicePrincipalName

# ldapsearch against GC
ldapsearch -H ldap://<DC>:3268 -b "DC=forest,DC=root" "(servicePrincipalName=*)"
```

## Related
- Hosted on: [[Domain Controller (DC)]] (any DC can be promoted to GC).
- Scope: [[Forest]] (one GC per forest minimum, typically one per site).
- Replication: [[AD Replication]].
- Queries: [[LDAP]].
