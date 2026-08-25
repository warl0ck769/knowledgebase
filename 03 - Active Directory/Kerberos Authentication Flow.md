---
title: Kerberos Authentication Flow
aliases: ["AS-REQ and AS-REP", "TGS-REQ and TGS-REP", "AP-REQ", "AS-REQ", "AS-REP", "TGS-REQ", "TGS-REP", "Kerberos messages"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[KDC]]", "[[TGT vs TGS]]", "[[PAC]]", "[[AS-REP Roasting]]", "[[Kerberoasting]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Kerberos Authentication Flow

> [!summary] One-liner
> Six messages across three exchanges: AS (get TGT) → TGS (get ST) → AP (use ST).

## 1. AS exchange — get a TGT
- **AS-REQ:** client sends username + a **timestamp encrypted with the user's key** (this is **pre-authentication**).
- **AS-REP:** AS returns, **encrypted with the user's key**:
  - the **TGT** (itself encrypted with the **krbtgt key**), and
  - a **TGS session key**.

> [!note] Two attack hooks here
> - No pre-auth required ⇒ AS-REP is encrypted material crackable offline → **[[AS-REP Roasting]]**.
> - The encrypted timestamp can be brute-forced for password guessing (Kerberos pre-auth brute-force).

## 2. TGS exchange — get a Service Ticket
- **TGS-REQ:** client sends the **TGT** + an **authenticator** (timestamp encrypted with the TGS session key) + the target **SPN**.
- **TGS-REP:** TGS returns, **encrypted with the TGS session key**:
  - the **Service Ticket** (encrypted with the **target service's key**), and
  - a **service session key**.

> [!note] Attack hook
> Any user can ask for any SPN's ST; the ST is encrypted with the service account's key → **[[Kerberoasting]]** (crack offline).

## 3. AP exchange — use the Service Ticket
- **AP-REQ:** client presents the ST to the application server.
- **AP-REP:** (optional) service confirms mutual authentication.

## Why a red teamer cares
Each message exposes a specific weakness (AS-REP roast, pre-auth brute-force, Kerberoast). Knowing which key encrypts which part tells you exactly what a captured/forged ticket can do.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Ticket acquisition" (AS-REQ/REP, TGS-REQ/REP, AP-REQ/REP)
