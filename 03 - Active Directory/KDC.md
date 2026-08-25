---
title: KDC
aliases: ["KDC", "Key Distribution Center", "Authentication Server", "AS", "Ticket Granting Service", "TGS service"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[TGT vs TGS]]", "[[krbtgt account]]", "[[Kerberos Authentication Flow]]", "[[Domain Controller (DC)]]"]
created: 2026-06-06
updated: 2026-06-06
---

# KDC

> [!summary] One-liner
> The Key Distribution Center — the trusted ticket-issuing service that runs on every DC, made of the AS and the TGS.

## What it is
The **KDC** runs on each [[Domain Controller (DC)]] and has two logical halves:
- **AS (Authentication Server)** — verifies the initial logon and issues the **TGT** (encrypted with the **[[krbtgt account|krbtgt]] key**).
- **TGS (Ticket-Granting Service)** — accepts a valid TGT and issues **service tickets (ST)** for requested SPNs (encrypted with the **service's key**).

## Why a red teamer cares
- The KDC trusts anything correctly encrypted with the krbtgt key → forging TGTs ([[Golden Ticket]]) bypasses the AS entirely.
- The AS step is where **pre-auth** lives — accounts without it are [[AS-REP Roasting|AS-REP roastable]].
- The TGS step is where any user can request **any** SPN's ticket → [[Kerberoasting]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos actors"
