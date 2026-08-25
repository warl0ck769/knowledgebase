---
title: SSPI and SSPs
aliases: ["SSPI", "GSS-API", "Authentication Packages", "MSV1_0", "NTLMSSP", "Kerberos SSP", "Negotiate SSP", "Digest SSP", "Secure Channel SSP", "Schannel", "CredSSP"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SPNEGO]]", "[[NTLM Authentication]]", "[[Kerberos]]", "[[LSASS]]"]
created: 2026-06-06
updated: 2026-06-06
---

# SSPI and SSPs

> [!summary] One-liner
> SSPI (Windows' GSS-API) is the abstraction apps call for authentication; Security Support Providers (SSPs) are the concrete protocols behind it, all hosted in LSASS.

## What it is
- **GSS-API / SSPI** (Security Support Provider Interface) lets apps request auth services **without knowing the underlying protocol**.
- **SSPs** are the implementations, loaded by **[[LSASS]]** (which is why LSASS holds tickets/hashes).

## The Windows SSPs
| SSP | Role | Attacker note |
|---|---|---|
| **Kerberos SSP** | Kerberos auth; **holds tickets + Kerberos keys** for logged-on users | dump via [[LSASS Dumping]] |
| **NTLM SSP (MSV / MSV1_0)** | NTLM auth; **stores NT hashes** of current users | dump → [[Pass-the-Hash]] |
| **Negotiate SSP** | Picks **Kerberos first, NTLM fallback** (via [[SPNEGO]]) | downgrade to NTLM enables relay |
| **Digest SSP (WDigest)** | HTTP Digest; **caches cleartext passwords** to compute digests (off by default since 2008 R2, re-enableable via registry) | plaintext from LSASS if enabled |
| **Secure Channel (Schannel)** | SSL/TLS | — |
| **CredSSP** | Credential **delegation** (e.g. RDP) | sends creds to target |
| **Custom SSPs** | Org-specific | malicious SSP = persistence (e.g. mimikatz `memssp`) |

## Why a red teamer cares
SSPs are *where credentials live in memory*. Knowing which SSP holds what tells you what [[LSASS Dumping]] will yield, and the Negotiate→NTLM fallback is the root of relay/downgrade attacks.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "GSS-API/SSPI" / "Windows SSPs"
