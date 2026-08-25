---
title: Exchange Privilege Abuse
aliases: ["Exchange Windows Permissions", "PrivExchange", "Exchange DCSync"]
type: technique
domain: [red-team]
attack_tactic: [privesc]
tags: [type/technique, domain/red-team, attack/privesc]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[DCSync]]", "[[ACL Abuse]]", "[[NTLM Relay]]", "[[Privileged AD Groups]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Exchange Privilege Abuse

> [!summary] One-liner
> On-prem Exchange installs grant the "Exchange Windows Permissions" group excessive rights on the domain — abusable to grant DCSync and reach Domain Admin.

## Concept abused
Installing Exchange grants the **`Exchange Windows Permissions`** group (and Exchange servers) high privileges in the domain — notably **WriteDACL on the domain object**. If you control a member (or an Exchange server), you can grant yourself **DCSync** rights.

## How the attack works
1. Compromise an account in `Exchange Windows Permissions` (or an Exchange server's machine account).
2. Use its **WriteDACL** to add **DS-Replication-Get-Changes** to yourself ([[ACL Abuse]]).
3. **[[DCSync]]** → krbtgt → domain compromise.

> Historically also **PrivExchange**: coerce the Exchange server to authenticate to you, then **[[NTLM Relay|relay]]** that high-priv auth to LDAP for the same WriteDACL→DCSync outcome.

## Commands & tools
```powershell
# grant DCSync via the Exchange group's WriteDACL
Add-DomainObjectAcl -TargetIdentity 'DC=contoso,DC=local' -PrincipalIdentity me -Rights DCSync
```
```bash
# PrivExchange-style relay
ntlmrelayx.py -t ldap://DC --escalate-user me     # after coercing Exchange auth
```

## Detection / artifacts
DACL changes on the domain object; replication by non-DC; Exchange server coercion traffic.

## Mitigation
Apply Microsoft's Exchange split-permissions / reduced-rights guidance, monitor domain-object ACL changes, LDAP signing+channel binding.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Microsoft extras → Exchange"
