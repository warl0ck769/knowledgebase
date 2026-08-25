---
title: Kerberos Keys
aliases: ["Kerberos key", "AES256 key", "RC4-HMAC key", "etype", "encryption types"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LM and NT Hashes]]", "[[AD User Object]]", "[[Kerberos]]", "[[Pass-the-Key]]", "[[Overpass-the-Hash]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Kerberos Keys

> [!summary] One-liner
> Password-derived keys used by Kerberos; multiple key types exist for different encryption algorithms (etypes), and the RC4 key equals the NT hash.

## What they are
Keys derived from the user's password and stored for **[[Kerberos]]** authentication. Each corresponds to an **encryption type (etype)**:

| Key type (etype) | Algorithm | Notes |
|---|---|---|
| **AES256** | AES256-CTS-HMAC-SHA1-96 | Recommended; **OPSEC** — blends in, less likely to trigger alarms |
| **AES128** | AES128-CTS-HMAC-SHA1-96 | |
| **RC4** | RC4-HMAC | **Equal to the [[LM and NT Hashes|NT hash]]** (unsalted) |
| **DES** | DES-CBC-MD5 | Deprecated |

> AES keys are **salted** (derived using the principal name + realm), so the same password yields different AES keys in different accounts/domains; the RC4 key has no salt (it *is* the NT hash).

## Why a red teamer cares
- A stolen Kerberos key lets you request tickets **as** that user → **[[Pass-the-Key]]** (a.k.a. Over-Pass-the-Hash when using the RC4/NT key → [[Overpass-the-Hash]]).
- Choosing **AES** over RC4 in tools is better OPSEC: requesting/using **RC4** tickets in an AES-capable domain is a detection signal (e.g. "encryption downgrade").

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "User Secrets → Kerberos keys"
