---
title: DCSync
aliases: ["DCSync", "DRSUAPI replication attack"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Database (NTDS.dit)]]", "[[krbtgt account]]", "[[Golden Ticket]]", "[[NTDS.dit Extraction]]", "[[ACL Abuse]]", "[[DCShadow]]", "[[SID History Abuse]]", "[[Inter-realm TGT]]", "[[Domain and Forest Trusts]]"]
created: 2026-06-06
updated: 2026-06-07
---

# DCSync

> [!summary] One-liner
> Ask a DC to replicate an account's secrets to you using the directory-replication protocol — dump hashes (incl. krbtgt) remotely, no file access or code on the DC.

## Concept abused
Domain Controllers replicate directory data to each other via **DRSUAPI (`IDL_DRSGetNCChanges`)**. Any principal with the replication rights can *impersonate a DC* and request secrets from [[AD Database (NTDS.dit)|ntds.dit]].

## Prerequisites
- The **"Replicating Directory Changes"** + **"Replicating Directory Changes All"** extended rights — held by default by **Domain Admins / Enterprise Admins / DCs**, but can be **delegated to any user** (a common [[ACL Abuse]] privesc).

## How the attack works
1. Send a `GetNCChanges` request to a DC for the target account.
2. The DC returns that account's credential secrets ([[LM and NT Hashes]], [[Kerberos Keys]]).

## Commands & tools

### [[Mimikatz]]
```
:: pull the krbtgt secret (-> Golden Ticket)
lsadump::dcsync /domain:contoso.local /user:krbtgt
```

per harmj0y, the user format must be the **NT4 (short) domain name**, and you can target a specific DC with `/dc:`:
```
:: NT4_DOMAINNAME\username form; /dc: targets a specific DC
lsadump::dcsync /user:TESTLAB\krbtgt /domain:testlab.local /dc:Primary.testlab.local
```

### [[Impacket]] (secretsdump)
```bash
secretsdump.py 'contoso.local/Administrator@192.168.100.2' -just-dc-user krbtgt
```
> Dumping **all** accounts at once can exhaust DC memory — target specific users.

## Detection / artifacts
- DC event **4662** referencing the replication GUIDs (`1131f6aa-...` DS-Replication-Get-Changes) from a non-DC account.
- DRSUAPI replication traffic **sourced from a host that isn't a DC**.
- (harmj0y) Watch for unusual replication requests from non-DC accounts, multiple DCSync operations in a short window, and use of the "Replicating Directory Changes" permission.

## Mitigation
- Tightly restrict who holds replication rights; audit `4662`; monitor non-DC replication.

## More from harmj0y (DCSync + ExtraSIDs / forest-wide krbtgt)

From *"Mimikatz and DCSync and ExtraSIDs, Oh My"* — DCSync abuses **MS-DRSR** and the `GetNCChanges` function to impersonate a DC and pull credential material with **no code execution on the DC** (only replication rights + network reachability to a DC).

**Chaining DCSync into a cross-trust Golden Ticket (ExtraSIDs attack):** after DCSyncing a child domain's `krbtgt` hash, a [[Golden Ticket]] can be forged that injects **extra SIDs** into the PAC via `/sids:` ([[SID History Abuse]]) to elevate across the domain trust toward the forest root ([[Domain and Forest Trusts]], [[Inter-realm TGT]]):
```
kerberos::golden /user:SECONDARY$ /krbtgt:<KRBTGT_HASH> /domain:dev.testlab.local ^
  /sid:S-1-5-21-<CHILD-SID> /sids:S-1-5-21-<ROOT-SID>-516,S-1-5-9 ^
  /groups:516 /id:<RID> /ptt
```
SIDs useful for trust hopping:
- Domain Controllers group `<domain>-516`
- Enterprise Domain Controllers `S-1-5-9`
- Target/root domain Enterprise Admins `<rootdomain>-519`
- the child DC machine account SID

**OPSEC notes (harmj0y):**
- The initial DCSync still runs as your current Domain Admin context, which **generates log entries**.
- Forging the golden ticket as the **child DC machine account (`SECONDARY$`)** rather than a normal DA account reduces logging visibility.
- If the `/id:` (RID) does **not** match `/user:`, the resulting ticket is only valid for ~**20 minutes**.

**Forest-wide remediation (harmj0y):** it is **not** sufficient to reset passwords and roll the `krbtgt` hash of just the root (or compromised) domain — you must roll the **`krbtgt` hash for ALL domains in the forest**, because ExtraSIDs/golden tickets allow cross-domain reuse.

## More from harmj0y (Invoke-MassMimikatz — mass credential dumping)

From *"Dumping a Domain's Worth of Passwords with Mimikatz, pt. 2"* — this is **not DCSync** but a complementary mass-credential-harvesting technique. `Invoke-MassMimikatz` runs [[Mimikatz]] across many hosts at once and aggregates the loot centrally. Unlike DCSync (which needs replication rights and one DC), this needs **local admin on each target**.

How it works:
1. Spins up a **backgrounded PowerShell web server** (default port `8080`, set with `-LocalPort`).
2. Builds a one-liner `IEX` download cradle that fetches `Invoke-Mimikatz` (reflective DLL injection), runs it, Base64-encodes the output, and POSTs results back.
3. Deploys the command across hosts via **WMI** (no PSRemoting, no dropped binaries).
4. Raw outputs are saved per host as `HOSTNAME.txt` in the output folder (default `MimikatzOutput`, set with `-OutputFolder`); a parser aggregates them into credential objects.

Detection surface (harmj0y gives no explicit OPSEC): HTTP delivery traffic, WMI execution audit logs, and Base64-encoded process command lines are all observable.

## Related
- Yields [[krbtgt account]] → [[Golden Ticket]]; alternative to [[NTDS.dit Extraction]]; enabled by [[ACL Abuse]].
- Combine with [[SID History Abuse]] / ExtraSIDs to pivot across [[Domain and Forest Trusts]] via the [[Inter-realm TGT]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domain database dumping → DCSync"
- [[Source - harmj0y blog]] — "Mimikatz and DCSync and ExtraSIDs, Oh My" — https://blog.harmj0y.net/redteaming/mimikatz-and-dcsync-and-extrasids-oh-my/
- [[Source - harmj0y blog]] — "Dumping a Domain's Worth of Passwords with Mimikatz, pt. 2" — https://blog.harmj0y.net/powershell/dumping-a-domains-worth-of-passwords-with-mimikatz-pt-2/
