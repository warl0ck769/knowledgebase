---
title: User Hunting
aliases: ["User Hunting", "Hunting Sysadmins", "Invoke-UserHunter", "Finding Local Admin Access", "Domain User Hunting"]
type: technique
domain: [active-directory, red-team]
attack_tactic: [recon, lateral-movement]
tags: [type/technique, domain/red-team, domain/active-directory, attack/recon, attack/lateral-movement]
source:
  - "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/penetesting/i-hunt-sysadmins/"
verified: true
related: ["[[PowerView]]", "[[Pass-the-Hash]]", "[[GPO Abuse]]", "[[Group Policy Object (GPO)]]", "[[AD Groups]]", "[[Privileged AD Groups]]", "[[Credential Hunting]]", "[[Remote Execution & Lateral Movement (Windows)]]", "[[Domain and Forest Trusts]]", "[[Service Principal Name (SPN)]]", "[[Empire]]"]
created: 2026-06-07
updated: 2026-06-07
---

# User Hunting

> [!summary] One-liner
> "User hunting" is locating *where* high-value users (admins/sysadmins) are logged in or have sessions across a domain — and *where you already have local admin* — so you can pivot to a box, steal their token/credentials, and escalate. Most of it works **as an unprivileged domain user** because Windows leaks session/logon and local-group data without authorization.

## Concept abused
Two Windows design facts make hunting possible:

1. **Session/logon enumeration is unauthenticated by default.** The Win32 APIs `NetSessionEnum` (active SMB sessions on a server) and `NetWkstaUserEnum` (interactively logged-on users) historically return data to **any** domain user. So you can ask every server "who is connected to you?" and "who is logged in?" without admin rights. (`NetWkstaUserEnum` requires local admin on the *target* on modern OSes; `NetSessionEnum` against file servers typically does not.)
2. **Local-group membership and GPO-defined local admins are readable remotely.** The ADSI `WinNT` provider and the `NetLocalGroupGetMembers` API expose local `Administrators` membership; GPO **Restricted Groups** (`GptTmpl.inf`) and **Group Policy Preferences** (`Groups.xml`) in SYSVOL declare who is a local admin — all queryable as an unprivileged user.

The goal: build a map of **(target user) -> (machine they touch)** and **(machine) -> (where I have local admin)**, then intersect them to plan the shortest path to Domain Admin.

## Prerequisites
- Any valid domain user (low-priv is enough for the recon).
- Network access to LDAP/the DC (for AD queries) and SMB/RPC to target hosts (for live session enumeration).
- `Get-NetSession`/`NetWkstaUserEnum`, process hunting, and event hunting have escalating privilege needs (see table below).
- GPO-based mapping needs **only** DC/SYSVOL reads — never touches target machines.

| Technique | API / Source | Privilege required |
|---|---|---|
| `Get-NetSession` (sessions on a server) | `NetSessionEnum` | None (unpriv domain user) |
| `Get-NetLoggedon` (logged-on users) | `NetWkstaUserEnum` | Local admin on target (modern OS) |
| `Get-NetLocalGroup` (local admins) | `WinNT` ADSI / `NetLocalGroupGetMembers` | None (unpriv) |
| `Find-LocalAdminAccess` (do *I* have admin?) | `OpenSCManagerW` (SC_MANAGER_ALL_ACCESS) | None to test |
| `Invoke-UserProcessHunter` | remote tasklist / `Get-NetProcess` | Local admin on target |
| `Invoke-UserEventHunter` | logon events (4624) on DCs | Domain Admin |
| `Find-GPOLocation` (GPO-defined admins) | SYSVOL `GptTmpl.inf` / `Groups.xml` | None (unpriv) |

## How the attack works
1. **Identify your prey.** Use AD queries to find high-value targets and groups: `Get-NetGroup *admin*`, recurse nested groups, correlate an admin's regular/desktop/server accounts via `displayname`, and pull `homeDirectory`/`profilePath`/SPNs to know which servers a user touches.
2. **Find where they are.** Run `Invoke-UserHunter` (queries every server's sessions + logged-on users and compares to your target set) or `Invoke-StealthUserHunter` (only hits the small set of "common" servers users connect to — file servers, DCs, DFS — for low traffic).
3. **Find where YOU have power.** Run `Find-LocalAdminAccess` (or `Invoke-EnumerateLocalAdmin`) to learn which machines you are already local admin on; map GPO-granted local admins with `Find-GPOLocation` without touching hosts.
4. **Intersect & pivot.** Where a target admin is logged in AND you have local admin -> move there ([[Remote Execution & Lateral Movement (Windows)]]), dump their token/creds, and escalate. Where they're logged in but you lack access, find an intermediate hop.

## Commands & tools

### Identify high-value targets (no host contact)
```powershell
# Find groups by wildcard (e.g. all *admin* groups), then recurse membership
Get-NetGroup '*admin*'
Get-NetGroupMember -GroupName 'Domain Admins' -Recurse

# Correlate an admin's alternate accounts via displayname
Get-NetGroupMember -GroupName 'Domain Admins' | ForEach-Object {
    Get-NetUser -Filter "(displayname=$($_.MemberName)*)"
}

# Targeted user search by LDAP filter / UPN domain
Get-NetUser -Filter "(userprincipalname=*@dev.testlab.local)"

# Servers a user touches: home dir / roaming profile / SPNs
Get-NetUser -UserName <USER> | Select samaccountname, homeDirectory, profilePath, serviceprincipalname
```

### Where is a target user logged in?
```powershell
# Raw building blocks
Get-NetSession  -ComputerName <FILESERVER>     # NetSessionEnum  (unpriv)
Get-NetLoggedon -ComputerName <HOST>           # NetWkstaUserEnum (admin on target)

# Full hunt: query all domain servers' sessions+logons, compare to target set
Invoke-UserHunter -GroupName 'Domain Admins'   # default targets DA group
Invoke-UserHunter -UserName  <TARGETUSER>
Invoke-UserHunter -GroupName 'Domain Admins' -CheckAccess   # also flag boxes you can admin

# Stealth: only hit common servers (from homeDirectories/DFS), far less traffic
Invoke-UserHunter -Stealth
Invoke-UserHunter -Stealth -StealthSource DFS   # DC | File | DFS | All

# Higher-privilege variants
Invoke-UserProcessHunter -UserName <USER>      # remote tasklist; admin on targets
Invoke-UserEventHunter   -UserName <USER>      # logon events 4624 on DCs; needs DA
```

### Where do I already have local admin?
```powershell
# Probe each host's SCM handle with SC_MANAGER_ALL_ACCESS (PsExec/WMI-capable)
Find-LocalAdminAccess
Invoke-CheckLocalAdminAccess -ComputerName <HOST>   # single-host helper

# Read remote local Administrators membership (unpriv) across the domain -> CSV
Get-NetLocalGroup -ComputerName <HOST>                       # default: Administrators
Get-NetLocalGroup -ComputerName <HOST> -ListGroups
Get-NetLocalGroup -ComputerName <HOST> -Recurse             # resolve domain group members
Get-NetLocalGroup -ComputerName <HOST> -API                 # NetLocalGroupGetMembers (faster, less detail)
Invoke-EnumerateLocalAdmin -Threads 20 -OutFile la.csv      # whole domain
Invoke-EnumerateLocalAdmin -TrustGroups                     # cross-trust admin relationships

# ADMIN$ reachability as a proxy for admin
Invoke-ShareFinder -CheckAdmin
Invoke-ShareFinder -CheckAccess
```

### Map local admins via GPO (zero host contact)
```powershell
# Where does <USER/GROUP> get local admin from GPO (Restricted Groups + GPP)?
Find-GPOLocation -UserName <USER>
Find-GPOLocation -GroupName 'Server Admins'
Find-GPOLocation -LocalGroup RDP            # target a non-Administrators local group

# Inverse: who is local admin on <COMPUTER> per GPO?
Find-GPOComputerAdmin -ComputerName <HOST>
Find-GPOComputerAdmin -OUName 'OU=Servers,DC=corp,DC=local' -Recurse

# Building blocks
Get-NetGPOGroup -ResolveMemberSIDs   # parse GptTmpl.inf + Groups.xml for local-group settings
Get-NetOU   -GUID <GPO-GUID>         # OUs linked via gPLink
Get-NetSite -GUID <GPO-GUID>
Get-NetComputer -ADSPath <OUPath>

# Export with flattened computer arrays
Find-GPOLocation | %{ $_.ComputerName = $_.ComputerName -join ', '; $_ } | Export-CSV -NoTypeInformation gpo_map.csv
```

### Non-PowerShell equivalents
```bash
netsess.exe \\<FILESERVER>            # NetSessionEnum
psloggedon.exe \\<HOST>              # HKU registry + NetSessionEnum
netview.exe -d -f hosts.txt --delay 30 --jitter 10   # sessions/shares/logons with delay+jitter
nmap --script smb-enum-sessions.nse -p445 <HOST>     # needs valid creds, no admin
PVEFindADUser.exe -current           # last-logged-in user (needs admin)
```

## Detection / artifacts
- **Mass `NetSessionEnum`/`NetWkstaUserEnum` fan-out**: one host enumerating sessions/logons across many servers in a short window is the classic `Invoke-UserHunter` signature. Monitor SMB/SRVSVC and Workstation service RPC volume.
- **SAMR/`WinNT` provider local-group queries** to many hosts -> local-admin enumeration.
- **`OpenSCManagerW` probes** (SCM connection attempts that don't start a service) across many hosts -> `Find-LocalAdminAccess`.
- **SYSVOL reads** of many `GptTmpl.inf`/`Groups.xml` files -> GPO-based mapping (no host telemetry, only DC/file-share access logs).
- Event 4624 logon hunting itself queries DC security logs (visible to DC log monitoring).

## Mitigation
- **NetCease / "Net Session Enumeration" hardening**: tighten the `SrvsvcSessionInfo` registry ACL so non-admins can't read `NetSessionEnum` (kills `Invoke-UserHunter`'s primary data source).
- Restrict remote SAMR enumeration ([KB] `RestrictRemoteSAM` / `Network access: Restrict clients allowed to make remote calls to SAM`) to blunt local-group enumeration.
- **Limit where privileged accounts log on** (tiered admin, dedicated PAWs, "Authentication Policies/Silos", `Protected Users`) so hunting yields no juicy sessions; clear cached creds / disconnected sessions promptly.
- Manage local admins centrally and minimally (avoid sprawling Restricted Groups / GPP grants), and prefer [[LAPS]] so a single stolen local-admin hash doesn't [[Pass-the-Hash|pass]] everywhere.

## Related
- [[PowerView]] — the toolkit implementing every function above.
- [[Pass-the-Hash]] / [[Remote Execution & Lateral Movement (Windows)]] — what you do once you find a box you admin.
- [[GPO Abuse]] / [[Group Policy Object (GPO)]] — the Restricted Groups / GPP mechanics behind `Find-GPOLocation`.
- [[Privileged AD Groups]] / [[AD Groups]] — targets you hunt and recurse.
- [[Domain and Forest Trusts]] — `-TrustGroups` / `-Domain` extend hunting cross-trust.
- [[Service Principal Name (SPN)]] — SPNs reveal where service accounts run.
- [[Credential Hunting]] — what you collect after landing on a target's box.

## Sources
- I Hunt Sysadmins — https://blog.harmj0y.net/penetesting/i-hunt-sysadmins/
- Identifying your Prey — https://blog.harmj0y.net/redteaming/identifying-your-prey/
- Local Group Enumeration — https://blog.harmj0y.net/redteaming/local-group-enumeration/
- Where My Admins At? (GPO Edition) — https://blog.harmj0y.net/redteaming/where-my-admins-at-gpo-edition/
- Finding Local Admin with the Veil-Framework — https://blog.harmj0y.net/penetesting/finding-local-admin-with-the-veil-framework/
