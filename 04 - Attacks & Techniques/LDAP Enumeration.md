---
title: LDAP Enumeration
aliases: ["LDAP enumeration", "AD recon via LDAP"]
type: technique
domain: [red-team]
attack_tactic: [recon]
tags: [type/technique, domain/red-team, attack/recon, proto/ldap]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LDAP]]", "[[AD Database]]", "[[SPN Scanning]]", "[[BloodHound enumeration]]", "[[UserAccountControl]]", "[[PowerView]]", "[[Credential Hunting]]", "[[User Hunting]]"]
created: 2026-06-06
updated: 2026-06-07
---

# LDAP Enumeration

> [!summary] One-liner
> Query the directory over LDAP (usually as any authenticated user) to map users, computers, groups, SPNs, and misconfigurations.

## Concept abused
The [[AD Database]] is readable over [[LDAP]] by **any authenticated domain user** — a goldmine of recon.

## Prerequisites
- One set of valid domain credentials (or anonymous bind on misconfigured DCs).

## Commands & tools

### ldapsearch (Linux)
```bash
# all users
ldapsearch -H ldap://DC -x -D "user@contoso.local" -W -b "dc=contoso,dc=local" "(objectClass=user)"

# accounts with SPNs (Kerberoast targets)
ldapsearch ... "(servicePrincipalName=*)" sAMAccountName servicePrincipalName

# AS-REP roastable (no pre-auth) via UAC bit
ldapsearch ... "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" sAMAccountName
```

### PowerView / AD module (Windows)
```powershell
Get-DomainUser | select samaccountname,serviceprincipalname
Get-DomainComputer -Properties dnshostname,operatingsystem
Get-DomainGroupMember "Domain Admins"
```

### NetExec / windapsearch / ldapdomaindump
```bash
nxc ldap DC -u user -p pass --users --groups --kerberoasting out.txt
ldapdomaindump ldap://DC -u 'contoso\user' -p pass
```

## Detection / artifacts
Directory service access on the DC; unusual/bulk LDAP queries; BloodHound's broad collection is noisy.

## Mitigation
Limit anonymous binds, monitor LDAP query volume, tier admin accounts.

## Related
- Feeds [[SPN Scanning]], [[AS-REP Roasting]], [[BloodHound enumeration]]; reads [[UserAccountControl]] flags.

## More from harmj0y (beyond-LDAP data mining: Windows Search Index)

> [!note] Scope
> harmj0y's "Mining a Domain's Worth of Data With PowerShell" is *not* about LDAP queries — it covers a complementary post-recon step: once LDAP enumeration tells you *which* machines matter, this technique mines the **content of files and emails** on those machines. Included here as a natural follow-on to directory recon. See also [[Credential Hunting]] and [[User Hunting]].

### Concept abused
Windows hosts run the built-in **Windows Search Index** service. By default the indexer indexes each user's **e-mail** and **Documents and Settings** folders, so the index already holds searchable copies of sensitive documents and mail. You can query this index **programmatically through PowerShell** instead of crawling the filesystem — much faster and stealthier than recursive `dir`/`Get-ChildItem` sweeps.

### Key technique
- The Windows Search Index exposes an **`AUTOSUMMARY`** field that returns *the section of a document that matches your query* — i.e. just the snippet you care about, without having to exfiltrate the whole file. [UNVERIFIED — exact field-query syntax not shown in the post]
- This lets an operator search a domain's worth of machines for terms like `password`, `confidential`, etc., and pull back only matching excerpts.

### Commands & tools
```powershell
# James O'Neill's original index-query function (adapted by harmj0y)
Get-IndexedItem            # query the local Windows Search Index for matching items

# harmj0y's weaponized, distributed version (Veil PowerTools / PewPewPew)
Invoke-MassSearch
#   - stands up a web server in the background
#   - triggers a download cradle on specified target machines to pull a
#     PowerShell script (Get-IndexItem) from the attacker
#   - base64-encodes / reports the search results back to the invoking system
```
> Exact parameter syntax is not given in the post (only a screenshot). Ships in the **Veil Framework's PowerTools** repo under the `PewPewPew` directory. [UNVERIFIED — repo layout per post text]

### OPSEC / scaling notes
- Querying the index is far cheaper than full-disk file searches, reducing IO and noise on each host.
- `Invoke-MassSearch` scales the search across many machines via web-server + download-cradle delivery rather than requiring PSRemoting on each box.

### Detection / artifacts
- Web server stood up on the attacker host plus **download-cradle** (HTTP fetch of a PowerShell script) network activity on targets. [UNVERIFIED — derived from described behavior, not an explicit detection section]
- Unusual programmatic access to the Windows Search Index / `Search.CollatorDSO` provider on hosts.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "How to query the database? → LDAP"
- [[Source - harmj0y blog]] — "Mining a Domain's Worth of Data With PowerShell" — https://blog.harmj0y.net/powershell/mining-a-domains-worth-of-data-with-powershell/
