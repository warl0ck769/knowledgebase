---
title: Credential Storage in Windows
aliases: ["Credential Storage in Windows", "DCC2", "MSCACHEv2"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LSASS]]", "[[SAM Database]]", "[[LSA Secrets]]", "[[LSASS Dumping]]", "[[SAM and LSA Secrets Dump]]", "[[Credential Hunting]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Credential Storage in Windows

> [!summary] One-liner
> A map of *where* Windows keeps secrets — memory vs. registry — so you know which dump technique recovers what.

## The four main stores
| Store | Where | Holds | Dump with |
|---|---|---|---|
| **[[LSASS]]** (memory) | `lsass.exe` process | NT hashes, Kerberos keys/tickets, sometimes plaintext | [[LSASS Dumping]] |
| **[[SAM Database|SAM]]** (registry) | `HKLM\SAM` | Local users' NT hashes | [[SAM and LSA Secrets Dump]] |
| **[[LSA Secrets]]** (registry) | `HKLM\SECURITY\Policy\Secrets` | `$MACHINE.ACC`, service pwds, autologon, DPAPI_SYSTEM | [[SAM and LSA Secrets Dump]] |
| **DCC2 / DPAPI** | registry / per-user | cached domain logons, DPAPI-protected data | both of the above |

> All registry stores are encrypted with the **BootKey/SysKey** from the **SYSTEM** hive — you always need SYSTEM to decrypt SAM/LSA.

## DCC2 (Domain Cached Credentials, MSCACHEv2)
- Cached results of the **last domain logons**, so a laptop can log in offline.
- Format: **`$DCC2$10240#username#hash`** — **crack with hashcat** (cannot be used for pass-the-hash; it's a one-way verifier).

## DPAPI (Data Protection API)
- Windows encryption for per-user/per-machine secrets (browser passwords, Credential Manager, etc.).
- **`DPAPI_SYSTEM`** master key (from [[LSA Secrets]]) and user master keys decrypt this data.

## Why a red teamer cares
Knowing the store tells you the **format and usability**: LSASS/SAM hashes → pass-the-hash now; DCC2 → crack only; LSA `$MACHINE.ACC`/service creds → immediate auth. For files on disk (browsers, config, history) see [[Credential Hunting]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Windows computers credentials"
