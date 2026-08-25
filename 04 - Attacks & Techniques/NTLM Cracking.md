---
title: NTLM Cracking
aliases: ["NTLM cracking", "NetNTLM cracking", "hashcat NTLM modes", "NTLM brute-force"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LM and NT Hashes]]", "[[NTLM Authentication]]", "[[LLMNR-NBT-NS Poisoning]]", "[[Kerberoasting]]"]
created: 2026-06-06
updated: 2026-06-06
---

# NTLM Cracking

> [!summary] One-liner
> Recover plaintext passwords offline from stolen NT hashes or captured NetNTLM challenge-responses.

## Concept abused
- **NT hashes** are **unsalted** → vulnerable to dictionaries and **rainbow tables**.
- **NetNTLM** responses captured via poisoning can be cracked offline (the only secret is the user's NT hash / password).

## hashcat modes (canonical — memorize these)
| Mode | Hash type |
|---|---|
| **1000** | **NTLM** (the stored NT hash) |
| **5500** | **NetNTLMv1** (challenge-response) |
| **5600** | **NetNTLMv2** (challenge-response) |
| 3000 | LM hash |

## Commands & tools
```bash
# NetNTLMv2 captured by Responder
hashcat -m 5600 hashes.txt rockyou.txt -r rules/best64.rule

# NT hashes dumped from SAM/NTDS
hashcat -m 1000 nthashes.txt rockyou.txt

# John the Ripper
john --format=netntlmv2 hashes.txt
```
> **NetNTLMv1** (mode 5500) can often be cracked to the **NT hash** itself (not just the password) via known DES/rainbow techniques — then pivot to [[Pass-the-Hash]].

## Detection / artifacts
Offline — no target-side artifacts (the *capture* step is what's detectable).

## Mitigation
Long/complex passwords, disable NTLMv1, disable LM, monitor for capture vectors.

## Related
- Capture via [[LLMNR-NBT-NS Poisoning]]; same cracking toolchain as [[Kerberoasting]] (mode 13100).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "NTLM brute-force" / "NTLM hashes cracking"
