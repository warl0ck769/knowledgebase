---
title: SharpHound
aliases: ["SharpHound", "SharpHound.exe", "BloodHound collector"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "SpecterOps"
source_url: "https://github.com/BloodHoundAD/SharpHound"
verified: true
related: ["[[BloodHound]]", "[[LDAP Enumeration]]", "[[PowerView]]"]
created: 2026-06-07
updated: 2026-06-07
---

# SharpHound

> [!summary] One-liner
> The official data collector for [[BloodHound]] — a C# executable (or PowerShell wrapper) that enumerates AD objects, ACLs, group memberships, sessions, and trusts, outputting JSON files for BloodHound ingestion.

## Collection methods

| Method | What it collects |
|---|---|
| `Default` | Group memberships, local admins, sessions, trusts, ACLs |
| `All` | Everything including GPO, OU, container, DCOM, RDP, PSRemote |
| `Session` | Active sessions only (re-run periodically) |
| `LoggedOn` | Privileged session collection via registry |
| `ACL` | DACL/SACL on AD objects |
| `Stealth` | Queries only DCs, file servers, Exchange — less noise |

## Usage
```powershell
# Full collection
SharpHound.exe --CollectionMethods All --Domain <DOMAIN> --OutputDirectory C:\temp\

# Stealth (reduced OPSEC footprint)
SharpHound.exe --CollectionMethods All --Stealth

# Session loop (re-collect sessions every 10 minutes for 2 hours)
SharpHound.exe --CollectionMethods Session --Loop --LoopDuration 02:00:00 --LoopInterval 00:10:00

# Exclude DCs from session enum (OPSEC)
SharpHound.exe --CollectionMethods All --ExcludeDomainControllers

# PowerShell wrapper
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\temp\
```

## Output
- ZIP file containing JSON files (users, groups, computers, domains, GPOs, OUs, containers).
- Import into [[BloodHound]] UI or CE API.

## Linux alternative
```bash
# bloodhound-python (bloodhound.py)
bloodhound-python -d <DOMAIN> -u <USER> -p <PASS> -ns <DC_IP> -c all
```

## Related
- Analysis platform: [[BloodHound]].
- Alternative enumeration: [[PowerView]], [[LDAP Enumeration]].
