---
title: SPN Scanning
aliases: ["SPN scanning", "SPN enumeration", "service discovery via SPN"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [recon]
tags: [type/technique, domain/red-team, attack/recon]
source: "Tim Medin / Sean Metcalf / community"
source_url: "https://adsecurity.org/?p=230"
verified: true
related: ["[[Service Principal Name (SPN)]]", "[[Kerberoasting]]", "[[LDAP Enumeration]]", "[[LDAP]]", "[[PowerView]]"]
created: 2026-06-07
updated: 2026-06-07
---

# SPN Scanning

> [!summary] One-liner
> Enumerate services across the domain by querying AD for [[Service Principal Name (SPN)|SPNs]] — a stealthier alternative to port scanning that uses LDAP queries instead of network probes.

## Concept abused
Every service registered in AD has a [[Service Principal Name (SPN)]] (e.g., `MSSQLSvc/sql01.corp.local:1433`). Since SPNs are stored as LDAP attributes on user/computer objects, any authenticated domain user can query for them — revealing what services exist, where they run, and what accounts host them, all without sending a single packet to the target hosts.

## Why it matters
- **No network scanning needed**: Traditional port scanning (nmap) is noisy and often detected. SPN queries are normal LDAP traffic.
- **Maps services to accounts**: SPNs on **user accounts** (not computer accounts) are prime [[Kerberoasting]] targets — their service tickets can be cracked offline.
- **Discovers non-obvious services**: MSSQL, Exchange, HTTP-based services, SCCM, etc.

## Common SPN prefixes

| Prefix | Service |
|---|---|
| `MSSQLSvc/` | Microsoft SQL Server |
| `HTTP/` | Web services (IIS, ADFS, SCCM, etc.) |
| `exchangeMDB/` | Exchange |
| `TERMSRV/` | RDP / Terminal Services |
| `WSMAN/` | WinRM |
| `ldap/` | LDAP (DCs) |
| `cifs/` | SMB/CIFS file services |
| `FIMService/` | Forefront Identity Manager |

## Commands & tools

### PowerView
```powershell
# All SPNs in the domain
Get-DomainUser -SPN | Select SamAccountName, ServicePrincipalName

# MSSQL services specifically
Get-DomainUser -SPN | Where-Object {$_.ServicePrincipalName -match "MSSQL"}

# SPNs on computer accounts
Get-DomainComputer -SPN | Select DnsHostName, ServicePrincipalName
```

### Built-in PowerShell
```powershell
# LDAP query for user accounts with SPNs (Kerberoastable)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName | Select Name, ServicePrincipalName
```

### setspn.exe (built-in)
```cmd
# All SPNs in the domain
setspn.exe -T <DOMAIN> -Q */*

# Search for a specific service
setspn.exe -T <DOMAIN> -Q MSSQLSvc/*
```

### Impacket
```bash
# GetUserSPNs — lists user accounts with SPNs (Kerberoastable)
GetUserSPNs.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP>
```

## Detection / artifacts
- Large LDAP queries filtering on `servicePrincipalName` from a single source.
- Generally low-risk from a detection standpoint — SPN queries blend with normal LDAP traffic.
- Correlate with subsequent [[Kerberoasting]] (TGS-REQ for discovered SPNs).

## Mitigation
- SPN scanning itself is hard to prevent (any authenticated user can query LDAP).
- Focus on downstream defenses: strong passwords on service accounts ([[Kerberoasting]] mitigation), use gMSAs, reduce SPNs on user accounts.

## Related
- Concept: [[Service Principal Name (SPN)]].
- Follows into: [[Kerberoasting]] (cracking service tickets for user-account SPNs).
- Enumeration context: [[LDAP Enumeration]], [[PowerView]].
