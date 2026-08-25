---
title: MOC - Tools & Code
type: moc
domain: [red-team]
tags: [type/moc, domain/red-team]
created: 2026-06-06
updated: 2026-06-07
---

# MOC — Tools & Code

> [!abstract] Scope
> Offensive tooling referenced by technique notes. ✅ = note written.

## harmj0y / GhostPack ecosystem
- [[PowerView]] ✅ (AD recon/abuse) · [[PowerUp]] ✅ (local privesc) · [[Rubeus]] ✅ (Kerberos) · [[GhostPack]] ✅ (Seatbelt/SharpUp/SharpDPAPI/Certify…)
- [[Empire]] ✅ (PowerShell/Python C2 + EmPyre) · [[PowerShell Offensive Tradecraft]] ✅ · [[PowerSCCM]] ✅ · [[Pwnstaller]] ✅

## Credential extraction
- [[Mimikatz]] ✅ (Swiss-army knife: LSASS, DCSync, tickets, DPAPI)
- [[Impacket]] ✅ (Python: secretsdump, GetUserSPNs, lateral movement scripts)

## Enumeration
- [[BloodHound]] ✅ (graph-based attack path analysis) · [[SharpHound]] ✅ (BloodHound collector)
- [[ADExplorer]] ✅ (Sysinternals LDAP browser + snapshots) · [[ldapsearch]] ✅ (CLI LDAP queries)

## Kerberos / credential-specific
- [[Kekeo]] ✅ (dedicated Kerberos toolkit) · [[Certipy]] ✅ (AD CS enumeration + exploitation)

## Relay / poisoning
- [[Responder]] ✅ (LLMNR/NBT-NS/mDNS poisoner + hash capture)
- [[ntlmrelayx]] ✅ (NTLM relay server — Impacket) · [[Inveigh]] ✅ (Windows-native poisoner)
- [[mitm6]] ✅ (IPv6 DHCPv6 MITM + DNS hijack)

## Execution / lateral movement
- [[CrackMapExec|CrackMapExec / NetExec]] ✅ (mass credential testing + exec across SMB/LDAP/WinRM/MSSQL)
- [[PsExec]] ✅ (SMB service-based remote exec) · [[wmiexec]] ✅ (WMI-based remote exec) · [[evil-winrm]] ✅ (WinRM shell)

## Payload / evasion
- [[Veil]] ✅ (AV-evasion payload generation)
