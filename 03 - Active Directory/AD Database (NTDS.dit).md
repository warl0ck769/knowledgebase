---
title: AD Database (NTDS.dit)
aliases: ["NTDS.dit", "ntds.dit", "AD database", "Domain database"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain Controller (DC)]]", "[[DCSync]]", "[[NTDS.dit Extraction]]", "[[krbtgt account]]", "[[LM and NT Hashes]]", "[[AD Database]]", "[[LDAP]]"]
created: 2026-06-06
updated: 2026-06-06
---

# AD Database (NTDS.dit)

> [!summary] One-liner
> The single file on each DC that stores every domain object and every user's credential secrets.

## What it is
`ntds.dit` (at **`C:\Windows\NTDS\ntds.dit`**) is the **domain database** held by each [[Domain Controller (DC)]]. It contains:
- All AD objects (users, computers, groups, OUs, etc.) and their attributes.
- **Credential secrets** for every account — [[LM and NT Hashes]] and [[Kerberos Keys]], including **[[krbtgt account|krbtgt]]**.

It is an ESE (Extensible Storage Engine) database; secrets inside are encrypted with the **SYSTEM** hive's BootKey (so dumping needs the SYSTEM hive too).

## Why a red teamer cares
This file **is** the domain's secrets. Two ways to get the contents:
- **[[DCSync]]** — pull specific accounts' secrets remotely via replication (no file access needed).
- **[[NTDS.dit Extraction]]** — copy the file (via `ntdsutil`/`vssadmin`) + SYSTEM hive and parse offline.

Either yields krbtgt (→ [[Golden Ticket]]) and every other hash (→ [[Pass-the-Hash]] everywhere).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domain Controllers → Domain database dumping"
