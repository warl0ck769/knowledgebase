---
title: LM and NT Hashes
aliases: ["NT hash", "LM hash", "NTLM hash", "NThash"]
type: concept
domain: [active-directory, windows-internals]
tags: [type/concept, domain/active-directory, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD User Object]]", "[[SAM Database]]", "[[AD Database (NTDS.dit)]]", "[[Pass-the-Hash]]", "[[Overpass-the-Hash]]", "[[Kerberos Keys]]"]
created: 2026-06-06
updated: 2026-06-06
---

# LM and NT Hashes

> [!summary] One-liner
> The two 16-byte password-derived secrets Windows stores (local [[SAM Database|SAM]] and AD [[AD Database (NTDS.dit)|NTDS.dit]]); the NT hash is the one that still matters and powers hash-based attacks.

## What they are
Two **16-byte** values derived from the user's password, stored in the local **[[SAM Database|SAM]]** and in **[[AD Database (NTDS.dit)|NTDS.dit]]**.

### LM hash (legacy, weak — usually disabled)
Weaknesses that make it trivially crackable:
- Password is **upper-cased** (shrinks keyspace).
- Passwords **> 14 chars are truncated**.
- Split into **two 7-byte halves**, each DES-encrypted with the constant string `KGS!+#$%` → halves crack **independently**.
- When LM is unused it shows as `aad3b435b51404eeaad3b435b51404ee` (LM hash of the empty string).

### NT hash (the important one)
```
nt_hash = MD4( UTF-16LE(password) )
```
- **No salt** → vulnerable to precomputed **rainbow tables**.
- Equivalent to the Kerberos **RC4-HMAC** key (see [[Kerberos Keys]]).

## Format you'll see (secretsdump etc.)
```
<username>:<rid>:<LM>:<NT>:::
# e.g.  jdoe:1103:aad3b435b51404eeaad3b435b51404ee:<32-hex-NT>:::
```

## Why a red teamer cares
The NT hash alone is enough to authenticate without the password:
- **[[Pass-the-Hash]]** — reuse NT hash over NTLM to access remote machines.
- **[[Overpass-the-Hash]]** — use NT hash (as RC4 key) to request Kerberos tickets.
- Crack offline to recover the plaintext (LM halves especially).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "User Secrets → LM/NT hashes"
