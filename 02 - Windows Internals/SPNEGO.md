---
title: SPNEGO
aliases: ["SPNEGO", "Negotiate", "GSS-API negotiation"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals, proto/kerberos, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SSPI and SSPs]]", "[[NTLM Authentication]]", "[[Kerberos]]"]
created: 2026-06-06
updated: 2026-06-06
---

# SPNEGO

> [!summary] One-liner
> The negotiation protocol that lets client and server agree on an auth mechanism — normally "Kerberos first, NTLM as fallback."

## What it is
**SPNEGO** (Simple and Protected GSS-API Negotiation Mechanism) is what the **Negotiate SSP** uses to pick a mutually supported mechanism:
1. **Client** sends a token proposing mechanisms (typically **Kerberos**, then **NTLM** fallback).
2. **Server** selects one and replies with its token.
3. Both proceed with the chosen mechanism's flow.

It is designed to **prefer the stronger mechanism** and commit early to resist downgrade.

## Why a red teamer cares
- The **Kerberos→NTLM fallback** is the seam attackers exploit: force/await NTLM and then **relay** it ([[NTLM Relay]]).
- Using **hostname/FQDN** (not IP) keeps you on Kerberos; an IP target falls back to NTLM.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "SPNEGO"
