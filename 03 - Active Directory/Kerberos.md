---
title: Kerberos
aliases: ["Kerberos", "Kerberos principal", "Realm"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[KDC]]", "[[TGT vs TGS]]", "[[Kerberos Authentication Flow]]", "[[PAC]]", "[[Kerberos Keys]]", "[[krbtgt account]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Kerberos

> [!summary] One-liner
> AD's primary authentication protocol: a trusted third party (the KDC) issues encrypted tickets so clients can prove identity to services without sending passwords.

## Core ideas
- **Realm** — the administrative domain, the DNS name **in uppercase** (e.g. `CONTOSO.LOCAL`).
- **Principal** — an identity in the realm:
  - **User:** `user@REALM` (e.g. `anakin@CONTOSO.LOCAL`)
  - **Service:** `service/host@REALM` (e.g. `host/dc01.contoso.local@CONTOSO.LOCAL`, `krbtgt@CONTOSO.LOCAL`)

## Actors
- **Client** — requests access.
- **[[KDC]]** (on the DC) — two logical services: **AS** (issues TGTs) + **TGS** (issues service tickets).
- **Service / application server** — the resource being accessed.

## Ports
- **88** (TCP/UDP) — AS + TGS.
- **464** (TCP/UDP) — kpasswd (password changes).

## Key hierarchy (who can read what)
| Key | Derived from | Encrypts |
|---|---|---|
| **User key** | user password ([[Kerberos Keys]], AES/RC4) | the AS-REP to that user |
| **krbtgt key** | [[krbtgt account]] password | **every TGT** |
| **Service key** | service account password | that service's tickets (ST) + PAC signature |
| **Session keys** | generated per exchange | client↔TGS and client↔service comms |

## Flow at a glance
1. **AS exchange** → get a **TGT** (proof you authenticated).
2. **TGS exchange** → trade the TGT for a **service ticket (ST)** for a specific [[Service Principal Name (SPN)|SPN]].
3. **AP exchange** → present the ST to the service.
Full detail: [[Kerberos Authentication Flow]]; ticket types: [[TGT vs TGS]].

## Why a red teamer cares
Every ticket is "something encrypted with a key." Steal a key → forge tickets: krbtgt key → [[Golden Ticket]]; service key → [[Silver Ticket]]; user key → [[Overpass-the-Hash]]. Tickets in memory → [[Pass-the-Ticket]]. SPN tickets → [[Kerberoasting]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos Basics" (principals, actors, services, keys)
