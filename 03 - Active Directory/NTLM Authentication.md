---
title: NTLM Authentication
aliases: ["NTLM", "NetNTLM", "NetNTLMv1", "NetNTLMv2", "NTLM challenge-response", "MIC"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LM and NT Hashes]]", "[[SSPI and SSPs]]", "[[Pass-the-Hash]]", "[[NTLM Relay]]", "[[NTLM Cracking]]", "[[Kerberos]]"]
created: 2026-06-06
updated: 2026-06-06
---

# NTLM Authentication

> [!summary] One-liner
> A challenge-response protocol (pre-Kerberos, still used as fallback): server sends a challenge, client proves it knows the NT hash without sending it.

## The three-message flow
1. **NEGOTIATE** — client announces capabilities.
2. **CHALLENGE** — server sends a random **8-byte challenge**.
3. **AUTHENTICATE** — client returns a response computed from the **[[LM and NT Hashes|NT hash]]** + challenge (+ client data).

> The response sent over the wire is the **NetNTLM** (v1/v2) — *distinct from* the stored NT hash. You crack NetNTLM, but you pass-the-**NT-hash**.

## NetNTLMv1 vs NetNTLMv2
| | NetNTLMv1 | NetNTLMv2 |
|---|---|---|
| Crypto | **DES** (challenge split into 3×7-byte keys encrypting `KGS!+#$%`) | **HMAC-MD5** |
| Inputs | server challenge only | NT hash + server challenge + **client nonce + timestamp + target info** |
| Strength | weak (parallel-crackable; downgradeable) | much stronger; unique each time |

## MIC (Message Integrity Code)
Optional HMAC-MD5 over **all three messages** (using the session key) appended to AUTHENTICATE — detects tampering. Some **relay protections require MIC validation**.

## NTLM in Active Directory
A server validating NTLM contacts a **DC over the Netlogon secure channel**; the DC fetches the user's NT hash from [[AD Database (NTDS.dit)|ntds.dit]], recomputes the expected response, and confirms.

## Why a red teamer cares
NTLM's design enables three classic attacks: **[[Pass-the-Hash]]** (hash = credential), **[[NTLM Relay]]** (forward the auth), and **[[NTLM Cracking]]** (crack captured NetNTLM). Forcing NTLM (vs Kerberos) is half the battle — see [[SPNEGO]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "NTLM Basics" / "NTLM in Active Directory"
