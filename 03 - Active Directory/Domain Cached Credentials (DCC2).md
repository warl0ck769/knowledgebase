---
title: Domain Cached Credentials (DCC2)
aliases: ["DCC2", "MSCACHEv2", "cached domain logon", "mscash2"]
type: concept
domain: [active-directory, windows-internals]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/cached-domain-logon-information"
verified: true
related: ["[[Credential Storage in Windows]]", "[[LM and NT Hashes]]", "[[NTLM Authentication]]", "[[SAM Database]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Domain Cached Credentials (DCC2)

> [!summary] One-liner
> Hashed copies of domain users' credentials stored locally in the registry so they can log on even when no Domain Controller is reachable — slow to crack and cannot be used for pass-the-hash.

## How it works
1. User authenticates to a domain via a DC.
2. Windows caches a **derived hash** of the user's credential in the registry at `HKLM\SECURITY\Cache`.
3. Default: **10 cached logons** stored (configurable via `CachedLogonsCount` GPO setting; set to 0 to disable).
4. Next time the user logs on and no DC is reachable, Windows validates against the cached entry.

## Hash format
- **DCC2 / MSCACHEv2** (Vista+): `PBKDF2(HMAC-SHA1, NT_hash, username, 10240 iterations)` — intentionally slow to crack.
- **DCC1 / MSCACHEv1** (XP/2003): `MD4(NT_hash + lowercase_username)` — fast to crack, obsolete.
- **Not an NT hash**: you **cannot** use DCC2 hashes for [[Pass-the-Hash]] — they're a one-way derivation, not the original credential.

## Extraction
```
# Impacket — from registry hives (offline)
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL

# Mimikatz — live (requires SYSTEM)
lsadump::cache

# From live system (reg save + offline)
reg save HKLM\SECURITY security.save
reg save HKLM\SYSTEM system.save
```

## Cracking
```bash
# hashcat mode 2100 (DCC2 / MSCACHEv2)
hashcat -m 2100 dcc2_hashes.txt wordlist.txt

# Format: $DCC2$10240#username#hash
```
DCC2 is deliberately slow (~10240 PBKDF2 iterations) — cracking speed is orders of magnitude slower than NT hashes.

## Red-team relevance
- **Offline/laptop attacks**: Laptops that cache domain creds are vulnerable if the disk is accessed (boot from USB, BitLocker bypass).
- **Last resort**: When LSASS dumping isn't possible, cached creds may be the only available credential material — but cracking is slow.
- **Not useful for relay/PtH**: Unlike NT hashes, DCC2 hashes cannot be relayed or passed.

## Mitigation
- Set `CachedLogonsCount` to **0** on machines that always have DC connectivity (servers, desktops on corporate LAN).
- For laptops that need offline logon, reduce to **1–2** cached logons.
- **BitLocker** with TPM+PIN prevents offline disk access.
- Strong passwords resist the slow DCC2 cracking.

## Related
- Part of: [[Credential Storage in Windows]].
- Derived from: [[LM and NT Hashes]] (NT hash is the input to DCC2).
- Compare with: [[SAM Database]] (local accounts — NT hashes, not DCC2).
