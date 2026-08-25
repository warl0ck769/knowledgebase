---
title: MOC - Attacks & Techniques
type: moc
domain: [red-team]
tags: [type/moc, domain/red-team]
created: 2026-06-06
updated: 2026-06-07
---

# MOC — Attacks & Techniques

> [!abstract] Scope
> Every offensive technique by ATT&CK tactic. Each note: **Concept abused → Commands & tools → Detection → Mitigation**. ✅ = written.

## Reconnaissance / Discovery `#attack/recon`
- [[Domain Controller Discovery]] ✅ · [[Windows Host Enumeration]] ✅ · [[LDAP Enumeration]] ✅ · [[SPN Scanning]] ✅
- [[Kerberos Brute-Force]] ✅ (user enum + spray) · [[Password Spraying]] ✅ (lockout-aware credential testing) · [[User Hunting]] ✅ (session/admin location)

## Name-resolution poisoning & MITM
- [[LLMNR-NBT-NS Poisoning]] ✅ · [[WPAD and IPv6 (mitm6) Poisoning]] ✅ · [[ADIDNS Spoofing]] ✅
- [[ARP Spoofing]] ✅ · [[DHCP Attacks]] ✅ · [[DNS Attacks]] ✅

## Credential Access `#attack/credential-access`
- [[Kerberoasting]] ✅ · [[AS-REP Roasting]] ✅ · [[NTLM Cracking]] ✅
- [[DCSync]] ✅ · [[NTDS.dit Extraction]] ✅ · [[LSASS Dumping]] ✅ · [[SAM and LSA Secrets Dump]] ✅ · [[Credential Hunting]] ✅
- [[LAPS]] ✅ · [[DPAPI Abuse]] ✅ · [[Attacking KeePass]] ✅ · [[Remote SAM Hash Extraction via Security Descriptors]] ✅ · [[WDigest Downgrade]] ✅

## Privilege Escalation `#attack/privesc`
- [[ACL Abuse]] ✅ · [[GPO Abuse]] ✅ · [[AD Certificate Services Abuse (ESC1-ESC8)]] ✅ · [[GPP Passwords]] ✅ · [[UAC Bypass]] ✅
- [[Exchange Privilege Abuse]] ✅ · delegation: [[Unconstrained Delegation]] ✅ / [[Constrained Delegation]] ✅ / [[Resource-Based Constrained Delegation (RBCD)]] ✅
- **Named CVEs/chains:** [[Zerologon]] ✅ · [[noPac]] ✅ · [[PetitPotam]] ✅ · [[PrintNightmare]] ✅ · [[MS14-068]] ✅

## Lateral Movement `#attack/lateral-movement`
- [[Remote Execution & Lateral Movement (Windows)]] ✅ · [[MSSQL Abuse]] ✅
- [[Pass-the-Hash]] ✅ · [[Overpass-the-Hash]] ✅ (Pass-the-Key) · [[Pass-the-Ticket]] ✅
- [[NTLM Relay]] ✅ · cross-domain: [[Inter-realm TGT]] ✅ · [[Forest Trust Abuse]] ✅

## Defense Evasion / C2 `#attack/defense-evasion`
- [[AD as C2 Channel]] ✅ · [[harmj0y Misc Tradecraft]] ✅ · [[EDR Bypass (Kernel)]] ✅ (driver architecture, SSDT, kernel callbacks)

## Detection (blue-team crossover)
- [[AD Replication Metadata (Detection)]] ✅

## Persistence `#attack/persistence`
- [[Golden Ticket]] ✅ · [[Silver Ticket]] ✅ · [[SID History Abuse]] ✅ · [[AdminSDHolder]] ✅
- [[Skeleton Key]] ✅ · [[DCShadow]] ✅ · [[DSRM]] ✅

## Cheat sheets
- _(none yet — candidate: a Kerberos-attacks cheat sheet)_
