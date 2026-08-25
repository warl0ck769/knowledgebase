---
title: LDAP
aliases: ["LDAP", "LDAPS", "ADWS", "Lightweight Directory Access Protocol"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/ldap]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Database]]", "[[LDAP Enumeration]]", "[[Global Catalog]]", "[[BloodHound enumeration]]"]
created: 2026-06-06
updated: 2026-06-06
---

# LDAP

> [!summary] One-liner
> The primary protocol for querying/modifying the AD database — base DN + filters return objects and attributes.

## What it is
**LDAP (Lightweight Directory Access Protocol)** is the main interface to the [[AD Database]].

| Port | Use |
|---|---|
| **389** | LDAP (cleartext/StartTLS) |
| **636** | LDAPS (TLS) |
| **3268 / 3269** | [[Global Catalog]] (forest-wide), LDAP/LDAPS |
| **9389** | **ADWS** (Active Directory Web Services) — used by the PowerShell `ActiveDirectory` module |

## Query anatomy
- **Base DN** — where to start, e.g. `DC=contoso,DC=local`.
- **Filter** — criteria, e.g. `(objectClass=user)`, `(servicePrincipalName=*)`.
- **Attributes** — which fields to return.

```bash
ldapsearch -H ldap://192.168.100.2 -x -LLL -W -D "anakin@contoso.local" \
  -b "dc=contoso,dc=local" "(objectclass=computer)" DNSHostName OperatingSystem
```

## Other query protocols
- **ADWS** (9389) — SOAP/HTTP; what `Get-AD*` cmdlets use.
- **RPC / DRSUAPI** — replication API (basis of [[DCSync]]).

## Why a red teamer cares
LDAP is the backbone of AD recon — almost everything ([[BloodHound enumeration]], PowerView, [[SPN Scanning]]) is LDAP underneath. Often readable by **any authenticated user**. See [[LDAP Enumeration]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "How to query the database? (LDAP / ADWS / Other protocols)"
