---
title: PowerView
aliases: [Veil-PowerView, PowerView 2.0, PowerView 3.0]
type: tool
domain: [active-directory, red-team, windows-internals]
tags: [type/tool, domain/red-team, domain/active-directory, tool/powerview]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/redteaming/powerview-2-0/
verified: true
related: ["[[PowerUp]]", "[[GhostPack]]", "[[Empire]]", "[[Rubeus]]", "[[LDAP Enumeration]]", "[[ACL Abuse]]", "[[ACL, ACE, DACL, SACL]]", "[[GPO Abuse]]", "[[User Hunting]]", "[[Kerberoasting]]", "[[Domain and Forest Trusts]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary] PowerView is a PowerShell tool (part of PowerSploit) for Active Directory reconnaissance and offensive enumeration. It reimplements the Windows `net *` commands plus deep LDAP/.NET, WMI, and Win32 API queries to map users, computers, groups, sessions, shares, ACLs, GPOs, sites, subnets, and trust relationships — and to hunt high-value users and abuse ACL/GPO/delegation primitives.

## What it does

- Pure-PowerShell domain/network situational awareness — no RSAT, no admin rights required for most functions. Originally written to bypass corporate restrictions on `net *` commands.
- Queries AD via three traffic types, signaled by a noun prefix in the v3 naming scheme:
  - **Domain** (e.g. `Get-DomainUser`) — LDAP/.NET `DirectorySearcher` queries.
  - **WMI** (e.g. `Get-WMIRegProxy`) — WMI enumeration.
  - **Net** (e.g. `Get-NetSession`) — Win32 API calls.
- Verb scheme mirrors official AD cmdlets: **Get-** (raw dataset), **Find-** (locate/threaded hunting), **Add-/New-** (create), **Set-** (modify), **Convert-** (format transforms).
- Supports alternate credentials everywhere: `Verb-Domain*` builds `DirectoryEntry` with `-Credential`; `Verb-Net*` uses token impersonation (`Invoke-UserImpersonation` → `LogonUser()` with `LOGON32_LOGON_NEW_CREDENTIALS` → `ImpersonateLoggedOnUser()`); `Convert-ADName` uses the NameTranslate COM `InitEx`.

### Version lineage
- **Veil-PowerView (v1):** original. `Get-NetComputers`, `Get-NetShare`, `Get-NetSessions`, `Invoke-ShareFinder`, `Invoke-UserHunter`, `Invoke-StealthUserHunter`, `Invoke-FindVulnSystems`, `Invoke-Netview`.
- **PowerView 2.0:** major refactor — singularized/standardized names (`Get-NetDomainControllers`→`Get-NetDomainController`, `Invoke-NetUserAdd`→`Add-NetUser`), folded functions into parameters (`Get-NetUserSPNs`→`Get-NetUser -SPN`; `Get-NetPrinters`→`Get-NetComputer -Printers`), moved threading to `-Threads X`, and added ACL/GPO/site/subnet functions plus Pester tests.
- **PowerView 3.0 ("Make PowerView Great Again"):** full rewrite. **Verb-PrefixNoun** naming, standardized parameters (`-Identity`, `-LDAPFilter`, `-Properties`, `-SearchBase`, `-Server`, `-SearchScope`, `-Credential`, `-FindOne`, `-Tombstone`, `-SecurityMasks`), full objects on the pipeline (dropped `-FullData`), PSScriptAnalyzer compliance, XML help + platyPS docs. Breaking change: `Get-NetLocalGroup` returns groups; members come from `Get-NetLocalGroupMember`.

## Key commands/functions

> Names below use the v3.0 `Get-Domain*` convention; the older `Get-Net*` aliases largely still work.

```powershell
# --- Core enumeration ---
Get-Domain                              # current domain object
Get-DomainController                    # DCs in the domain
Get-DomainUser -Identity <user>         # user objects (LDAP/.NET)
Get-DomainComputer                      # computer objects
Get-DomainGroup -UserName <user>        # groups; recursive membership, -AdminCount
Get-DomainGroupMember <group>
Get-DomainUser -SPN                     # accounts with SPNs (kerberoast targets)

# Optimize-to-the-left: request only needed attributes server-side
Get-DomainUser -Properties samaccountname
Get-DomainComputer -LDAPFilter '(name=WINDOWS1)' -Properties dnshostname

# Forest-wide search via the Global Catalog
Get-DomainComputer -SearchBase "GC://$ENV:USERDNSDOMAIN" -LDAPFilter '(name=<short>)' -Properties dnshostname
```

```powershell
# --- Sessions / shares / hunting (Net + Find) ---
Get-NetSession -ComputerName <host>
Get-NetLoggedon -ComputerName <host>
Find-DomainShare                        # threaded share finder (was Invoke-ShareFinder)
Invoke-UserHunter -Stealth -SearchForest   # where high-value users are logged in
Invoke-Kerberoast                       # request + extract roastable TGS hashes
```

```powershell
# --- ACL / object takeover discovery ---
Get-DomainObjectAcl -ResolveGUIDs
# GPO editors in a foreign domain (object-takeover primitive):
Get-DomainObjectAcl -Domain dev.testlab.local -LDAPFilter '(objectCategory=groupPolicyContainer)' -ResolveGUIDs |
  Where-Object { $_.SecurityIdentifier -match 'S-1-5-.*-[1-9]\d{3,}$' }
Convert-ADName -Identity <SID> -OutputType DN
Get-DomainSID -Domain dev.testlab.local   # filter same-domain principals for cross-domain ACE hunting
```

```powershell
# --- GPO / sites / subnets / trusts ---
Get-DomainGPO
Get-DomainGPOLocalGroup                  # GPOs setting local group membership (Restricted Groups / GPP)
Find-GPOLocation -Identity <user>        # where a user/group gets admin/RDP via GPO
Find-GPOComputerAdmin -ComputerName <h>
Get-DomainSite ; Get-DomainSubnet ; Get-DomainPolicy ; Get-DomainTrust
```

```powershell
# --- Modify / create / impersonate ---
New-DomainUser ; New-DomainGroup ; Add-DomainGroupMember
Set-DomainObject -Identity <obj> -Set @{...}
Add-DomainObjectAcl                      # grant ACEs (ACL abuse / DCSync rights)
Invoke-UserImpersonation -Credential <cred> ; Invoke-RevertToSelf
Add-RemoteConnection ; Remove-RemoteConnection
```

### PowerUsage one-liner patterns (harmj0y series)
- **#1 Interactive-logon mapping:** `Get-DomainComputer` (filter out `TRUSTED_FOR_DELEGATION` to limit credential exposure) piped to `Get-WmiObject Win32_UserProfile`, regex-filter domain SIDs (`S-1-5-21-...$`), export CSV.
- **#2 Shortname → FQDN across a forest:** loop shortnames through `Get-DomainComputer -SearchBase "GC://..." -Properties dnshostname`.
- **#3 Foreign-domain GPO editors:** `Get-DomainObjectAcl -Domain <foreign> -LDAPFilter '(objectCategory=groupPolicyContainer)' -ResolveGUIDs`, filter RID ≥ 1000 + takeover rights.
- **#4 Cross-domain ACE relationships:** `Get-DomainSID` of the target, then `Get-DomainObjectAcl` keeping ALLOW ACEs whose SecurityIdentifier is outside the target domain and grants GenericAll/owner rights (trust-hopping vectors).
- **#5 User→workstation mapping:** `Get-DomainUser -Properties samaccountname`, then `Get-DomainComputer -Identity *$samaccountname*`, export CSV.

## Used in techniques
- [[LDAP Enumeration]], [[User Hunting]], [[Credential Hunting]]
- [[ACL Abuse]], [[ACL, ACE, DACL, SACL]], [[DCSync]]
- [[GPO Abuse]], [[Group Policy Object (GPO)]]
- [[Kerberoasting]], [[Service Principal Name (SPN)]]
- [[Unconstrained Delegation]], [[Constrained Delegation]], [[Resource-Based Constrained Delegation (RBCD)]]
- [[Domain and Forest Trusts]], [[Trust Direction and Transitivity]], [[Forest]]
- Related tooling: [[PowerUp]], [[GhostPack]], [[Empire]], [[Rubeus]]

## Sources
- Veil-PowerView: A Usage Guide — https://blog.harmj0y.net/powershell/veil-powerview-a-usage-guide/
- PowerView 2.0 — https://blog.harmj0y.net/redteaming/powerview-2-0/
- Make PowerView Great Again — https://blog.harmj0y.net/powershell/make-powerview-great-again/
- The PowerView PowerUsage Series #1 — https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-1/
- The PowerView PowerUsage Series #2 — https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-2/
- The PowerView PowerUsage Series #3 — https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-3/
- The PowerView PowerUsage Series #4 — https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-4/
- The PowerView PowerUsage Series #5 — https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-5/
