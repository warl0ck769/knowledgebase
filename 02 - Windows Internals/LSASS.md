---
title: LSASS
aliases: ["lsass.exe", "Local Security Authority Subsystem Service"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Credential Storage in Windows]]", "[[LSASS Dumping]]", "[[Authentication Packages]]", "[[Privileges and Rights]]", "[[LM and NT Hashes]]", "[[Kerberos Keys]]"]
created: 2026-06-06
updated: 2026-06-06
---

# LSASS

> [!summary] One-liner
> The Windows process that handles authentication and caches the credentials of logged-on users for SSO — the #1 in-memory credential theft target.

## What it is
**Local Security Authority Subsystem Service** (`lsass.exe`) enforces local security policy and performs authentication. To enable **single sign-on**, it **caches credentials** from interactive logons and RDP sessions in its memory.

## What's cached in memory
- **NT hashes** (via NTLMSSP / MSV1_0 authentication package)
- **Kerberos keys & tickets** (via the Kerberos SSP)
- **Plaintext passwords** — only on misconfigured systems (e.g. **WDigest** enabled: `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential = 1`; WDigest plaintext caching is **off by default since Windows 2008 R2**)

(Credentials live behind the **[[Authentication Packages|Security Support Providers / authentication packages]]** that LSASS loads.)

## Access requirements
Reading LSASS memory needs **`SeDebugPrivilege`** (normally admin-only) enabled in the calling process.

## Protections (defender side)
- **Credential Guard** — hypervisor-isolates secrets (VBS) so they aren't in normal LSASS memory.
- **PPL (Protected Process Light)** for `lsass.exe` — blocks non-PPL processes from opening its memory.

## Why a red teamer cares
Dumping LSASS yields ready-to-use hashes/keys/tickets for [[Pass-the-Hash]], [[Overpass-the-Hash]], [[Pass-the-Ticket]]. See the technique: [[LSASS Dumping]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Windows computers credentials → LSASS credentials"
