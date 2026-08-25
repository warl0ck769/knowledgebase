---
title: Communication Protocols
aliases: ["Communication Protocols", "SMB", "Named Pipes", "RPC", "WinRM", "PowerShell Remoting", "Trusted Hosts", "RDP", "SSH", "SSH tunneling"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/smb]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Remote Execution & Lateral Movement (Windows)]]", "[[NetBIOS]]", "[[DCSync]]", "[[Linux in AD]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Communication Protocols

> [!summary] One-liner
> The network protocols AD relies on — and the same ones attackers use for recon, lateral movement, and data access.

## Protocol & port reference
| Protocol | Port(s) | Notes / tools |
|---|---|---|
| **SMB** | **445** | File shares + named pipes; `smbclient`, `nxc smb`, Impacket |
| **RPC** | **135** (EPM) + dynamic **49152–65535** | RPC over SMB (named pipes) or over TCP; `psexec.py`/`wmiexec.py`/`atexec.py` |
| **WinRM** | **5985** (HTTP) / **5986** (HTTPS) | PowerShell Remoting transport; `evil-winrm` |
| **RDP** | **3389** | GUI; transmits creds (cached) — Restricted Admin enables PtH; `mstsc`/`xfreerdp` |
| **SSH** | **22** | Linux access + **tunneling/port-forwarding** ([[Linux in AD]]) |
| **HTTP** | 80/443 | Web admin interfaces, ADWS, AD CS enrollment |

## SMB shares
**Default machine shares:** `C$` (system drive), `ADMIN$` (Windows dir), `IPC$` (named pipes).
**Default domain shares:** `SYSVOL` (GPOs + logon scripts), `NETLOGON` (logon scripts/policies).

## Named pipes
Named pipes carry **RPC over SMB** — the transport behind PsExec, many lateral-movement and enumeration RPC calls, and **DRSUAPI** ([[DCSync]]).

## PowerShell Remoting — Trusted Hosts
`TrustedHosts` lets a client authenticate to hosts **without Kerberos validation** (e.g. by IP) — handy cross-domain/workgroup, and an OPSEC/abuse consideration.

## Why a red teamer cares
Knowing which port/protocol is open dictates the lateral-movement tool ([[Remote Execution & Lateral Movement (Windows)]]); `IPC$`/named pipes underpin most remote RPC attacks.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Communication Protocols" (SMB, HTTP, RPC, WinRM, PS Remoting, SSH, RDP)
