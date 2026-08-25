---
title: AD as C2 Channel
aliases: ["Command and Control using Active Directory", "AD dead-drop", "Malleable C2"]
type: technique
domain: [red-team]
attack_tactic: [defense-evasion, persistence]
tags: [type/technique, domain/red-team, attack/defense-evasion]
source: "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/powershell/command-and-control-using-active-directory/"
verified: true
related: ["[[AD Database]]", "[[LDAP]]", "[[ACL, ACE, DACL, SACL]]", "[[Empire]]"]
created: 2026-06-07
updated: 2026-06-07
---

# AD as C2 Channel

> [!summary] One-liner
> Use writable Active Directory object attributes as a covert "dead-drop" to pass tasking and results between an implant and its operator — no traditional network C2 infrastructure required.

## Concept abused
Every authenticated domain user can **read** vast swaths of the [[AD Database]] over [[LDAP]], and can **write** to certain attributes on objects they control (e.g., their own user object). harmj0y's *Command and Control Using Active Directory* demonstrated that an attacker can stage data **inside the directory itself**: the implant writes tasking results into an attribute, the operator reads and writes the other side — all looking like ordinary LDAP traffic to a DC.

### The `mSMQSignCertificates` attribute
harmj0y identified **`mSMQSignCertificates`** as an ideal dead-drop attribute because:
- It is part of the **Personal-Information property set**, so by default **`NT AUTHORITY\SELF`** has read/write access (any user can modify it on their own object).
- It stores **binary data** up to ~**1 MB** — large enough for compressed payloads.
- It is almost never legitimately populated (MSMQ is rarely deployed), so writes are anomalous only if monitored.
- It replicates to the **Global Catalog** (it is in the GC partial attribute set), making the dead-drop readable **forest-wide** from any GC server.

## How the dead-drop works

```
Operator                          AD (DC / GC)                     Implant
   │                                    │                              │
   ├──LDAP write (tasking blob)────────►│                              │
   │    mSMQSignCertificates on         │                              │
   │    target user object              │◄──LDAP read (poll for task)──┤
   │                                    │                              │
   │                                    │◄──LDAP write (result blob)───┤
   │◄──LDAP read (get results)─────────│                              │
   │                                    │                              │
   ├──LDAP delete (cleanup)────────────►│                              │
```

### Data encoding
Payloads are **DeflateStream-compressed → base64-encoded** before writing. This keeps them under the attribute size limit and avoids binary encoding issues with LDAP string operations.

## PowerShell implementation (harmj0y's functions)

```powershell
# Stage a payload (tasking) to a user's mSMQSignCertificates attribute
New-ADPayload -Domain <DOMAIN> -UserName <TARGET_USER> -Payload "<COMMAND_STRING>"

# Implant-side: poll for and retrieve pending tasking
Get-ADPayload -Domain <DOMAIN> -UserName <TARGET_USER>

# Implant-side: write execution results back to the attribute
Get-ADPayloadResult -Domain <DOMAIN> -UserName <TARGET_USER>

# Operator-side: clean up after retrieving results
Remove-ADPayload -Domain <DOMAIN> -UserName <TARGET_USER>
```

All functions use `[System.DirectoryServices.DirectoryEntry]` / `[System.DirectoryServices.DirectorySearcher]` — pure .NET, no PowerShell AD module dependency.

## OPSEC advantages
- **Blends with legitimate traffic:** LDAP reads/writes to a DC are ubiquitous in enterprise environments and rarely alerted on.
- **No external infrastructure:** works even when egress is fully locked down, as long as the host can talk to a DC (port 389/636).
- **Asynchronous:** operator and implant never connect directly; timing is decoupled.
- **Forest-wide reach:** GC replication means the dead-drop is readable from any domain in the forest.

## Malleable C2 (complementary concept)
harmj0y's *A Brave New World: Malleable C2* covers **shaping traditional C2 traffic** to mimic legitimate protocols — a complementary approach to hiding in AD (make the *network channel* look benign vs. avoiding a network channel altogether).

Key concepts from that post:
- **Cobalt Strike Malleable C2 profiles**: customize GET/POST URIs, HTTP headers, user-agent strings, jitter, and encoding to blend with legitimate site traffic (e.g., mimic Amazon, Google, jQuery CDN requests).
- **`c2lint`** tool: validate a Malleable profile before deployment.
- **Traffic blending**: match the target environment's expected traffic patterns — the profile should reflect what the blue team considers normal.

## Prerequisites
- A foothold running as any **domain user** — default ACLs grant `SELF` write access to `mSMQSignCertificates` on the user's own object.
- Read access to the directory (default for all authenticated users).

## Commands & tools

```powershell
# Manual check: does the target user have MSMQ attributes populated?
Get-ADUser -Identity <USER> -Properties mSMQSignCertificates

# Manual LDAP write (raw .NET — what the functions do under the hood)
$entry = [ADSI]"LDAP://CN=<USER>,CN=Users,DC=<DOMAIN>,DC=com"
$payload = [System.Text.Encoding]::UTF8.GetBytes("<TASKING>")
$compressed = <DeflateStream compress $payload>
$entry.Properties["mSMQSignCertificates"].Value = [System.Convert]::ToBase64String($compressed)
$entry.CommitChanges()

# Read back
$raw = $entry.Properties["mSMQSignCertificates"].Value
$decoded = [System.Convert]::FromBase64String($raw)
# DeflateStream decompress → UTF8 decode → tasking string
```

## Detection / artifacts
- **Event 5136** (Directory Service Changes audit): fires on attribute modifications — alert on writes to `mSMQSignCertificates` (or any rarely-used binary attribute) that shouldn't normally change.
- Anomalous LDAP write patterns: high-frequency attribute modifications from a workstation to the same object.
- Large or base64-encoded values in attributes that are normally empty.
- Baseline which attributes are legitimately written in your environment; flag deviations.

## Mitigation
- **Enable Directory Service Changes auditing** (Advanced Audit Policy → DS Access → Audit Directory Service Changes) and alert on Event 5136 for sensitive attributes.
- Restrict write ACLs: remove unnecessary `SELF` write permissions on attributes that aren't needed ([[ACL, ACE, DACL, SACL|tighten DACLs]]).
- Monitor LDAP traffic volumes per source; segment so workstations only reach DCs on required ports.
- If MSMQ is not used, consider explicitly blocking or alerting on any writes to MSMQ-related attributes.

## Related
- Built on [[AD Database]] / [[LDAP]] read-write semantics.
- ACL mechanics: [[ACL, ACE, DACL, SACL]] (Personal-Information property set, `SELF` permissions).
- Pairs with C2 frameworks like [[Empire]]; Malleable C2 profiles are a Cobalt Strike concept.

## Sources
- [[Source - harmj0y blog]] — "Command and Control Using Active Directory" — https://blog.harmj0y.net/powershell/command-and-control-using-active-directory/
- [[Source - harmj0y blog]] — "A Brave New World: Malleable C2" — https://blog.harmj0y.net/redteaming/a-brave-new-world-malleable-c2/
