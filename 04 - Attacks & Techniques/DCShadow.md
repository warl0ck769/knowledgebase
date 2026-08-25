---
title: DCShadow
aliases: ["DCShadow attack", "rogue DC"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [persistence, defense-evasion]
tags: [type/technique, domain/red-team, attack/persistence]
source: "Benjamin Delpy & Vincent Le Toux (Mimikatz)"
source_url: "https://www.dcshadow.com/"
verified: true
related: ["[[DCSync]]", "[[AD Replication]]", "[[Domain Controller (DC)]]", "[[SID History Abuse]]", "[[AdminSDHolder]]"]
created: 2026-06-07
updated: 2026-06-07
---

# DCShadow

> [!summary] One-liner
> Register a rogue Domain Controller in AD and push malicious changes (backdoor ACLs, SID History, password resets) via legitimate replication — changes bypass most security monitoring because they appear as normal DC-to-DC replication.

## Concept abused
AD replication ([[AD Replication]]) is designed for DC-to-DC synchronization. DCShadow temporarily registers the attacker's machine as a DC in the AD configuration partition (by creating `nTDSDSA` and `server` objects), pushes arbitrary attribute changes via the DRSUAPI replication protocol, then deregisters. Because the changes arrive via replication (not LDAP writes), they evade most SIEM rules that monitor LDAP modification events.

## Prerequisites
- **Domain Admin** or equivalent privileges (needed to register a DC in the Configuration partition and to replicate).
- Two Mimikatz instances: one running as SYSTEM (to handle RPC), one running as DA (to push changes and trigger replication).

## How the attack works
1. **Register** the attacker machine as a temporary DC in AD (creates objects in `CN=Sites,CN=Configuration`).
2. **Stage changes** — specify what to modify (e.g., add SID to `sIDHistory`, modify `servicePrincipalName`, change group membership, alter `adminCount`).
3. **Push** — trigger replication from the rogue DC to a real DC; the real DC accepts and applies the changes.
4. **Deregister** — remove the rogue DC objects from the Configuration partition.

## Commands & tools

### Mimikatz (two-terminal approach)
```
# Terminal 1: RPC server (run as SYSTEM)
mimikatz # !+
mimikatz # !processtoken
mimikatz # lsadump::dcshadow /object:"CN=TargetUser,CN=Users,DC=corp,DC=local" /attribute:sIDHistory /value:S-1-5-21-<other_domain>-500

# Terminal 2: Push replication (run as DA)
mimikatz # lsadump::dcshadow /push
```

### Common DCShadow payloads

| Payload | Effect |
|---|---|
| Add `sIDHistory` | [[SID History Abuse]] — grants cross-domain/forest access |
| Modify `primaryGroupID` | Silently add to Domain Admins |
| Set `servicePrincipalName` | Enable [[Kerberoasting]] of the target |
| Modify `AdminCount` + `ntSecurityDescriptor` | Backdoor via [[AdminSDHolder]] propagation |
| Modify `userAccountControl` | Enable reversible encryption, disable pre-auth, etc. |

## Detection / artifacts
- **New `nTDSDSA` objects** appearing in `CN=Sites,CN=Configuration` (rogue DC registration).
- **Event 4742** on DC: computer account changes for the rogue DC registration.
- **Replication metadata**: `Get-ADReplicationAttributeMetadata` shows the originating DC GUID doesn't match any known DC — see [[AD Replication Metadata (Detection)]].
- Monitor `CN=Configuration` for unauthorized changes.
- Short-lived DC objects (created and deleted within seconds/minutes).

## Mitigation
- Monitor the Configuration partition for new `nTDSDSA`/`server` objects.
- Use [[AD Replication Metadata (Detection)]] — check `msDS-ReplAttributeMetaData` for unknown originating DC GUIDs.
- Restrict DA-equivalent access (DCShadow requires DA — preventing DA compromise prevents DCShadow).
- SIEM correlation: alert on replication events from non-DC sources.

## Related
- Counterpart: [[DCSync]] (reads from replication; DCShadow writes via replication).
- Replication mechanics: [[AD Replication]].
- Detection: [[AD Replication Metadata (Detection)]].
- Common payloads: [[SID History Abuse]], [[AdminSDHolder]], [[Kerberoasting]].

## Sources
- Benjamin Delpy & Vincent Le Toux — DCShadow (presented at BlueHat IL 2018).
- https://www.dcshadow.com/
