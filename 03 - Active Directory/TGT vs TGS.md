---
title: TGT vs TGS
aliases: ["TGT", "TGS", "Service Ticket", "ST", "Ticket Granting Ticket"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[KDC]]", "[[Kerberos Authentication Flow]]", "[[krbtgt account]]", "[[Golden Ticket]]", "[[Silver Ticket]]"]
created: 2026-06-06
updated: 2026-06-06
---

# TGT vs TGS

> [!summary] One-liner
> A TGT proves you authenticated (encrypted with the krbtgt key); a Service Ticket grants access to one specific service (encrypted with that service's key).

## The two ticket types
| | **TGT** (Ticket-Granting Ticket) | **ST** (Service Ticket) |
|---|---|---|
| Issued by | **AS** | **TGS** |
| Encrypted with | **[[krbtgt account|krbtgt]] key** | the **service account's key** |
| Used to | request STs (no re-auth to AS) | access **one** service (via **AP-REQ**) |
| Lifetime | ~**10 hours**, renewable up to ~**7 days** | bound to the session |
| Forge it by stealing | krbtgt key → **[[Golden Ticket]]** | service key → **[[Silver Ticket]]** |

> Every ticket = an **encrypted blob + a session key**. You can only read/forge a ticket if you hold the key it's encrypted with.

## Why a red teamer cares
This table *is* the Kerberos attack map: which key you steal determines which ticket you can forge and how far it reaches (krbtgt = whole domain; service key = that one service).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Ticket types (ST, TGT)"
