---
title: Domain Controller (DC)
aliases: ["Domain Controller", "DC", "AD DS"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Database (NTDS.dit)]]", "[[Domain]]", "[[Domain Controller Discovery]]", "[[DCSync]]", "[[NTDS.dit Extraction]]", "[[krbtgt account]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Domain Controller (DC)

> [!summary] One-liner
> The central server running Active Directory Domain Services (AD DS) and hosting the domain database — owning a DC = owning the domain.

## What it is
A DC runs **AD DS** and maintains the domain database at **`C:\Windows\NTDS\ntds.dit`** ([[AD Database (NTDS.dit)]]), which holds **all domain objects and user credentials**. It also acts as the [[KDC]] (Kerberos) and the LDAP/auth server for the domain.

## Listening ports (DC fingerprint)
| Port | Service | Port | Service |
|---|---|---|---|
| 53 | DNS | 464 | kpasswd |
| 88 | **Kerberos** | 593 | RPC/HTTP |
| 135 | RPC Endpoint Mapper | 636 | LDAPS |
| 139 | NetBIOS Session | 3268/3269 | LDAP **Global Catalog** |
| 389 | **LDAP** | 5985 | WinRM |
| 445 | SMB | 9389 | **ADWS** |

> The combination of **88 (Kerberos) + 389 (LDAP) + 445** is the classic "this is a DC" signature.

## Why a red teamer cares
- DCs hold every credential in the domain (incl. **[[krbtgt account|krbtgt]]**). Getting `ntds.dit` or DCSync rights = full domain compromise.
- See [[Domain Controller Discovery]] to find them, then [[DCSync]] / [[NTDS.dit Extraction]] to loot them.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domain Controllers"
