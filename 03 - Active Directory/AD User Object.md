---
title: AD User Object
aliases: ["User object", "User properties", "sAMAccountName", "userPrincipalName", "User identifiers"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SID]]", "[[RID]]", "[[GUID]]", "[[LM and NT Hashes]]", "[[Kerberos Keys]]", "[[UserAccountControl]]", "[[AD Database (NTDS.dit)]]"]
created: 2026-06-06
updated: 2026-06-06
---

# AD User Object

> [!summary] One-liner
> A user is an object in the AD database, identified by several attributes and holding derived secrets used to authenticate to the domain.

## What it is
AD stores users as **objects** in the central database. Any point in the domain with sufficient rights can **query or modify** them (via [[LDAP]] etc.).

## User identifiers
| Attribute | Example | Notes |
|---|---|---|
| **sAMAccountName** | `jdoe` | The "username" (legacy logon name) |
| **objectSID** | `S-1-5-21-…-1103` | Domain [[SID]] + user [[RID]] (last number) |
| **distinguishedName (DN)** | `CN=John,OU=IT,DC=contoso,DC=local` | How the LDAP API identifies the object |
| **objectGUID** | (128-bit) | Globally unique, immutable → [[GUID]] |
| **userPrincipalName (UPN)** | `jdoe@contoso.local` | Modern logon name |

> SID = `S-1-5-21-<domainSID>-<RID>`. E.g. `S-1-5-21-1372086773-2238746523-2939299801-1103` → domain SID + RID `1103`.

## Secrets (not plaintext)
Passwords are stored as derived secrets that allow DC authentication:
- **[[LM and NT Hashes]]** — NTLM-side secrets (also in local [[SAM Database|SAM]]).
- **[[Kerberos Keys]]** — RC4/AES/DES keys for Kerberos auth.

## Security-relevant attributes
| Attribute | Why it matters (attacker) |
|---|---|
| **[[UserAccountControl]]** | Flags: pre-auth, delegation, disabled → enables [[AS-REP Roasting]], delegation abuse |
| **servicePrincipalName (SPN)** | Services tied to the user → target for [[Kerberoasting]] |
| **msDS-AllowedToDelegateTo** | Constrained-delegation targets → [[Constrained Delegation]] abuse (modify needs `SeEnableDelegationPrivilege`) |
| **Description** | May leak permissions or **cleartext passwords** |
| **adminCount** | `1` ⇒ object is/was protected by [[AdminSDHolder]] (unreliable if stale) |
| **memberOf** | Groups the user is in (logical; excludes primary group) |
| **primaryGroupID** | The user's **primary group** — does **not** appear in `memberOf` |

## Why a red teamer cares
Every identifier and attribute is an enumeration target; secrets enable [[Pass-the-Hash]], [[Overpass-the-Hash]], [[Pass-the-Key]]; SPN/UAC/delegation attributes directly enable named attacks.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Users → User properties"
