---
title: AD Replication Metadata (Detection)
aliases: ["Replication Metadata Hunting", "msDS-ReplAttributeMetaData", "msDS-ReplValueMetaData", "repadmin showobjmeta", "Hunting with AD Replication Metadata"]
type: technique
domain: [active-directory, windows-internals, red-team]
attack_tactic: [recon, defense-evasion]
tags: [type/technique, domain/active-directory, domain/red-team, attack/recon, attack/defense-evasion, proto/ldap, defense/detection]
source:
  - "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/defense/hunting-with-active-directory-replication-metadata/"
verified: true
related: ["[[DCSync]]", "[[ACL Abuse]]", "[[ACL, ACE, DACL, SACL]]", "[[SID History Abuse]]", "[[Golden Ticket]]", "[[Kerberoasting]]", "[[Service Principal Name (SPN)]]", "[[GPO Abuse]]", "[[Group Policy Object (GPO)]]", "[[AD Groups]]", "[[Privileged AD Groups]]", "[[LDAP Enumeration]]", "[[PowerView]]", "[[Domain Controller Discovery]]"]
created: 2026-06-07
updated: 2026-06-07
---

# AD Replication Metadata (Detection)

> [!summary] One-liner
> Every replicated AD attribute carries hidden, DC-maintained metadata (version count, last-change timestamp, originating DC); parsing it lets a defender (or an attacker doing recon) reconstruct *when* and *roughly how often* objects were modified — even after event logs are cleared — and lets a hunter spot suspicious "set-then-unset" changes to groups, SPNs, ACLs, and GPOs.

## Concept abused
Active Directory tracks per-attribute replication state so DCs can reconcile changes. Two **constructed** (computed-on-read) attributes expose this:

- **`msDS-ReplAttributeMetaData`** — one entry per modified *replicated* attribute on the object (non-linked attributes, e.g. `servicePrincipalName`, `userAccountControl`, `ntSecurityDescriptor`, `pwdLastSet`, `versionNumber`).
- **`msDS-ReplValueMetaData`** — one entry per value of a *linked* (forward-link) attribute, e.g. `member` / `manager`. Because linked values are tracked individually, this metadata preserves *which specific values* were added or removed and *when*, including values that no longer exist on the object.

Each metadata entry includes:
- **`dwVersion` (Version)** — number of originating modifications. Heuristic: an attribute set once = 1; set then cleared/re-set produces higher counts. For linked values, an **even** Version typically means the value was added then removed (deleted), an **odd** Version means it is currently present. [UNVERIFIED — exact even/odd semantics are environment-dependent; see Limitations]
- **`ftimeLastOriginatingChange` (LastOriginatingChange)** — UTC timestamp of the last originating change.
- **`uuidDsaOriginatingDSA` (LastOriginatingDsaDN)** — the NTDS-DSA (the DC) that originated the change.
- **`usnOriginatingChange` / `usnLocalChange`** — USNs for the originating and local change.
- **`ftimeCreated` / `ftimeDeleted`** — for linked values, when a value (e.g. a group member) was added and (if applicable) removed.

Why this matters for hunting: this metadata is maintained by the DCs as part of replication and **persists independently of the Security event log**. If an attacker clears logs or never had auditing enabled, the metadata still reveals that *something changed and when*, narrowing the forensic window.

## Prerequisites
- **For defenders/hunters:** read access to the object's metadata. The constructed attributes are returned via LDAP (e.g. PowerView / .NET `DirectoryEntry`) or via `repadmin /showobjmeta`. Reading metadata generally requires appropriate read rights on the object.
- **For attackers (recon angle):** the same read access lets an attacker profile when objects were changed (e.g. find the last time a krbtgt or admin password was set, or spot dormant SPNs).
- Understanding of **replicated vs. non-replicated** attributes — non-replicated attributes (e.g. `lastLogon`, `badPwdCount`) carry **no** replication metadata and are invisible to this technique.

## How the attack works
1. **Establish the change picture for an object.** Read `msDS-ReplAttributeMetaData` to enumerate every modified replicated attribute, its Version count, last-change time, and originating DC.
2. **Reconstruct linked-value history.** Read `msDS-ReplValueMetaData` to see individual group members / managers including *removed* ones, with create/delete timestamps. This is the only part of replication metadata that exposes *previous values* (and only for linked attributes).
3. **Spot anomalies via Version + timestamps.** A high Version on `servicePrincipalName`, `ntSecurityDescriptor`, `sIDHistory`, `member`, or `primaryGroupID`, or a "set-then-unset" pattern (e.g. an SPN that was added then removed — classic targeted Kerberoasting cleanup), is a hunting lead.
4. **Resolve the originating DC.** Map `LastOriginatingDsaDN` (an NTDS-DSA DN) to the physical DC to attribute the change to a source DC (see DSA resolution below).
5. **Correlate with event logs to recover the missing pieces (who / old value).** Metadata gives *when* and *which DC* but not *who* or *what the old value was* (except linked values). Pull matching directory-service / account-management events around the metadata timestamp.

### What the metadata can and cannot tell you
- Tells you: **when** an attribute was last changed, **how many times** (approx) it changed, the **originating DC**, and for linked attributes the **specific added/removed values** with times.
- Does NOT tell you: **who** made the change (requires event logs), or the **previous value** of a non-linked attribute (only linked-value history is preserved).

## Commands & tools

### PowerView — attribute and linked-attribute history
```powershell
# Parse msDS-ReplAttributeMetaData into objects (per-attribute version/time/originating DC)
Get-DomainObjectAttributeHistory -Identity <USER_OR_GROUP_OR_GPO>

# Focus on a single attribute (e.g. SPN tampering, ACL changes)
Get-DomainObjectAttributeHistory -Identity <TARGET> |
  Where-Object { $_.AttributeName -in 'servicePrincipalName','ntSecurityDescriptor','userAccountControl','sIDHistory' }

# Parse msDS-ReplValueMetaData — linked-attribute (member/manager) history, incl. removed values
Get-DomainObjectLinkedAttributeHistory -Identity <GROUP_OR_USER>

# Convenience: list members that were ADDED then REMOVED from a group (even Version + real TimeDeleted)
Get-DomainGroupMemberDeleted -Identity '<GROUP_NAME>'
```

### PowerView — distinguish replicated vs. non-replicated attributes (schema)
```powershell
# List NON-replicated attributes (FLAG_ATTR_NOT_REPLICATED, bit 0x1 of systemFlags)
# -> these carry NO replication metadata
Get-DomainObject -SearchBase 'ldap://CN=Schema,CN=Configuration,DC=<DOMAIN>,DC=<TLD>' `
  -LDAPFilter '(&(objectClass=attributeSchema)(systemFlags:1.2.840.113556.1.4.803:=1))' |
  Select-Object -ExpandProperty ldapdisplayname
# Negate the bitwise match (... :!=1 / NOT ...) to enumerate REPLICATED attributes instead.
```

### PowerView — GPO edit detection (correlate with SYSVOL)
```powershell
# When a GPO's versionNumber changes, get the change time, then diff against SYSVOL file times
Get-DomainObjectAttributeHistory -Identity '<GPO_DN>' |
  Where-Object { $_.AttributeName -eq 'versionNumber' }
# Then inspect file modification times under \\<DOMAIN>\SYSVOL\<DOMAIN>\Policies\{GUID}\
```

### PowerView — resolve LastOriginatingDsaDN to a physical DC
```powershell
# 1) The NTDS-DSA object referenced by the metadata
Get-DomainObject -SearchBase 'ldap://CN=Configuration,DC=<DOMAIN>,DC=<TLD>' `
  -Identity '<LastOriginatingDsaDN>'
# 2) Follow serverReferenceBL -> server object; then via ms-DFSR-Member / msDFSR-ComputerReference
#    chain to the actual DC computer object.
```

### Native — repadmin (no PowerView required)
```cmd
:: Show replication metadata for a single object directly from a DC
repadmin /showobjmeta <DC_NAME> "<OBJECT_DN>"
:: e.g.
repadmin /showobjmeta dc01 "CN=Domain Admins,CN=Users,DC=corp,DC=local"
```

### .NET / ADSI equivalent (read the constructed attributes)
```powershell
# msDS-ReplAttributeMetaData / msDS-ReplValueMetaData are returned as XML blobs you parse
([adsi]"LDAP://<OBJECT_DN>").get_Item('msDS-ReplAttributeMetaData')
([adsi]"LDAP://<OBJECT_DN>").get_Item('msDS-ReplValueMetaData')
```

## Detection / artifacts
This note *is* a detection technique; the artifacts below are the corroborating event-log entries you correlate with metadata timestamps (enable the matching audit policy first):

- **Group membership changes** — enable **Audit Security Group Management**: event IDs **4735** (security-enabled local group changed), **4737** (global group changed), **4755** (universal group changed); member add/remove events reveal the **principal** that made the change. Correlate with `member` `msDS-ReplValueMetaData` times.
- **SPN / user-object changes** — event **4738** (user account management: a user account was changed) identifies who set/cleared `servicePrincipalName`. A Version that incremented twice (set + clear) flags targeted Kerberoasting cleanup. Requires **Audit User Account Management**.
- **DACL / owner changes** — `ntSecurityDescriptor` metadata shows the change time but **cannot distinguish a DACL edit from an owner change**, and only the *directly modified* object's descriptor carries metadata (not inheritors). Correlate with 4738 / directory-service change events.
- **Password resets** — `pwdLastSet` metadata exists but is too noisy to hunt on alone; rely on **4723** (user changed own password) and **4724** (admin reset another's password).
- **GPO edits** — enable **Audit Directory Service Changes**: event **5136** shows which principal modified the GPO object; correlate the `versionNumber` metadata time with SYSVOL file change times.

> Forensic value note: replication metadata persists even when an attacker disables auditing or wipes the Security log, so it is most useful precisely when event logs are unavailable. The Version field is the highest-signal field for spotting set-then-unset tampering.

## Mitigation
This is primarily a *defensive/detection* capability, so "mitigation" here is operational hygiene to maximize its value and limit attacker recon:
- **Enable and forward** the audit policies above (Security Group Management, User Account Management, Directory Service Changes) so the *who* is captured before logs can be cleared.
- **Centralize logs** (SIEM/WEF) so log clearing on a DC does not destroy the *who/old-value* context that metadata lacks.
- **Baseline high-value objects** (Domain Admins, krbtgt, AdminSDHolder, sensitive GPOs) and periodically diff their replication metadata for unexpected Version jumps.
- Treat the metadata read itself as low-risk to expose but remember attackers can use it for recon (dormant SPNs, last krbtgt reset, ACL backdoor timing).

## Limitations
- **No "who".** Metadata never identifies the principal that made a change — only the originating DC. You must correlate with event logs.
- **No previous value for non-linked attributes.** Only linked attributes (`member`, `manager`) preserve prior/removed values via `msDS-ReplValueMetaData`.
- **Non-replicated attributes are invisible** (no metadata at all).
- **Approximate, not authoritative.** Version counts and even/odd linked-value semantics vary by **domain functional level** and Windows version; lab results may not match production. Use as a lead-generator, not proof. [UNVERIFIED — version-semantics edge cases]
- **DACL vs. owner ambiguity** on `ntSecurityDescriptor`; **password-reset noise** on `pwdLastSet`.

## Related
- [[DCSync]] — replication abuse this metadata can help spot/contextualize
- [[ACL Abuse]] / [[ACL, ACE, DACL, SACL]] — `ntSecurityDescriptor` change hunting
- [[Kerberoasting]] / [[Service Principal Name (SPN)]] — set-then-unset SPN detection
- [[SID History Abuse]] — `sIDHistory` change hunting
- [[Golden Ticket]] — correlate krbtgt `pwdLastSet`/changes
- [[GPO Abuse]] / [[Group Policy Object (GPO)]] — `versionNumber` + SYSVOL correlation
- [[AD Groups]] / [[Privileged AD Groups]] — `member` linked-value history
- [[PowerView]] — tooling; [[LDAP Enumeration]]; [[Domain Controller Discovery]] — DSA resolution

## Sources
- harmj0y — "Hunting with Active Directory Replication Metadata" — https://blog.harmj0y.net/defense/hunting-with-active-directory-replication-metadata/
