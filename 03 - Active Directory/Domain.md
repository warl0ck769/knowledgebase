---
title: Domain
aliases: ["AD Domain", "NetBIOS name", "Domain SID"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[What is Active Directory]]", "[[Domain Controller (DC)]]", "[[Forest]]", "[[Tree]]", "[[SID]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Domain

> [!summary] One-liner
> A set of connected computers that share one Active Directory database, managed by that domain's Domain Controllers.

## What it is
A **domain** is the foundational organizational unit of AD: a group of computers/users sharing a single AD database. The servers that manage it are the [[Domain Controller (DC)|Domain Controllers]].

## Domain identity (three names/IDs)
A domain is referred to in three ways — knowing all three matters during enumeration:

| Identifier | Example | Used for |
|---|---|---|
| **DNS name** | `contoso.com`, `contoso.local` | DNS resolution, FQDNs, modern auth |
| **NetBIOS name** | `CONTOSO` | Legacy logon format `NETBIOSNAME\username` |
| **Domain SID** | `S-1-5-21-1372086773-2238746523-2939299801` | Identity/security; prefix of every principal's [[SID]] in the domain |

> [!note] Recon
> PowerView/AD module output exposes these directly, e.g.:
> `DNSRoot: contoso.local | NetBIOSName: CONTOSO | DomainSID: S-1-5-21-1372086773-2238746523-2939299801`

## How it relates upward
- Domains can have **subdomains**, forming a [[Tree]].
- One or more trees rooted under a common forest root domain form a [[Forest]].

## Why a red teamer cares
- The **Domain SID** is the prefix for every user/computer/group [[SID]] in that domain — needed to forge tickets ([[Golden Ticket]]) and to reason about [[SID History Abuse]].
- The **NetBIOS name** appears throughout authentication (`DOMAIN\user`) and in [[NTLM Authentication|NTLM]] exchanges.
- A domain is **not** the top security boundary — the [[Forest]] is.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domains" / "Domain name"
