---
title: SAM Database
aliases: ["SAM", "Security Account Manager", "SAM hive"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Credential Storage in Windows]]", "[[LSA Secrets]]", "[[SAM and LSA Secrets Dump]]", "[[LM and NT Hashes]]", "[[Pass-the-Hash]]"]
created: 2026-06-06
updated: 2026-06-06
---

# SAM Database

> [!summary] One-liner
> The local registry hive storing NT hashes of a machine's **local** accounts — often reused across machines, enabling lateral movement.

## What it is
The **SAM (Security Account Manager)** hive holds the **[[LM and NT Hashes|NT hashes]] of local computer users** (e.g. the local Administrator). It is encrypted with the **BootKey/SysKey** derived from the **SYSTEM** hive — so to decrypt SAM you also need SYSTEM.

## Why a red teamer cares
- The local Administrator hash is frequently **reused domain-wide** (same image/build) → one SAM dump can [[Pass-the-Hash]] into many machines (classic if LAPS isn't deployed).
- Dumped together with [[LSA Secrets]] — see [[SAM and LSA Secrets Dump]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Registry credentials → SAM"
