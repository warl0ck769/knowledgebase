---
title: SMB
aliases: ["Server Message Block", "CIFS", "SMB1", "SMB2", "SMB3", "port 445"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/overview"
verified: true
related: ["[[NTLM Authentication]]", "[[NTLM Relay]]", "[[Pass-the-Hash]]", "[[Remote Execution & Lateral Movement (Windows)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# SMB (Server Message Block)

> [!summary] One-liner
> The primary Windows file-sharing and IPC protocol (port 445) — also the transport for remote administration (PsExec, sc.exe, schtasks), NTLM relay, and lateral movement.

## Versions

| Version | Introduced | Key features |
|---|---|---|
| **SMBv1** | Windows NT | Legacy; EternalBlue (MS17-010); **should be disabled** |
| **SMBv2** | Vista / 2008 | Improved performance, larger reads/writes |
| **SMBv2.1** | Win 7 / 2008R2 | Oplocks, large MTU |
| **SMBv3** | Win 8 / 2012 | Encryption, multichannel, RDMA |
| **SMBv3.1.1** | Win 10 / 2016 | Pre-authentication integrity, AES-128-GCM |

## Ports

| Port | Protocol | Notes |
|---|---|---|
| **445** | SMB over TCP | Primary port (modern) |
| **139** | SMB over NetBIOS | Legacy (NetBIOS session service) |
| **137-138** | NetBIOS name/datagram | [[NetBIOS]] name resolution |

## Authentication
SMB authenticates via [[NTLM Authentication]] or [[Kerberos]] (negotiated through [[SPNEGO]]). This makes it a prime target for:
- **[[NTLM Relay]]**: Relay a captured NTLM response to another SMB service.
- **[[Pass-the-Hash]]**: Authenticate to SMB with an NT hash directly.
- **[[Kerberoasting]]**: Service tickets for SMB services (if SPNs are set).

## SMB signing
- When **enabled and required**, it prevents NTLM relay to that host (the relay can't produce a valid signature).
- **Default**: Required on DCs; negotiated (not required) on member servers/workstations.
- Red-team impact: [[NTLM Relay]] only works against hosts that don't require signing.

## Key shares

| Share | Purpose |
|---|---|
| `ADMIN$` | `C:\Windows\` — used by PsExec for service deployment |
| `C$` | Root of C: drive — admin-only default share |
| `IPC$` | Inter-process communication — named pipes, RPC |
| `SYSVOL` / `NETLOGON` | GPO scripts, logon scripts on DCs |

## Red-team relevance
- **Lateral movement**: [[Remote Execution & Lateral Movement (Windows)]] — PsExec, smbexec, wmiexec all use SMB.
- **Enumeration**: `net view`, `smbclient`, CrackMapExec — list shares, check access.
- **File transfer**: Copy payloads / exfil data via SMB shares.
- **[[GPP Passwords]]**: `\\<domain>\SYSVOL\` may contain old Group Policy Preference XML with encrypted passwords.

```bash
# Enumerate shares
smbclient -L //<TARGET>/ -U <USER>%<PASS>
crackmapexec smb <TARGET> -u <USER> -p <PASS> --shares

# Check SMB signing
crackmapexec smb <TARGET> --gen-relay-list unsigned.txt
```

## Related
- Authentication: [[NTLM Authentication]], [[Kerberos]], [[SPNEGO]].
- Attacks: [[NTLM Relay]], [[Pass-the-Hash]], [[Remote Execution & Lateral Movement (Windows)]].
- Legacy: [[NetBIOS]].
