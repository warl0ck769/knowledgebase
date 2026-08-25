---
title: Windows Services
aliases: ["service", "SCM", "Service Control Manager", "sc.exe"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/services/services"
verified: true
related: ["[[Privileges and Rights]]", "[[Access Token]]", "[[LSA Secrets]]", "[[PowerUp]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Windows Services

> [!summary] One-liner
> Long-running background processes managed by the Service Control Manager (SCM), often running as high-privilege accounts (SYSTEM, Network Service) — misconfigured services are a classic local privilege escalation vector.

## Service accounts

| Account | Token level | Network identity | Notes |
|---|---|---|---|
| **LocalSystem** (`NT AUTHORITY\SYSTEM`) | System integrity | Machine account (`DOMAIN\MACHINE$`) | Highest local privilege; most services default here |
| **LocalService** (`NT AUTHORITY\LOCAL SERVICE`) | Medium integrity | Anonymous | Reduced privileges |
| **NetworkService** (`NT AUTHORITY\NETWORK SERVICE`) | Medium integrity | Machine account | Like LocalService but authenticates on network |
| **Domain user / gMSA** | Depends on account | That user/gMSA | Service account credentials stored in [[LSA Secrets]] |

## Service configuration
Each service has a registry entry under `HKLM\SYSTEM\CurrentControlSet\Services\<name>`:

| Value | Meaning |
|---|---|
| `ImagePath` | Path to the service binary (or `svchost.exe -k` group) |
| `ObjectName` | Account the service runs as |
| `Start` | Start type (0=Boot, 2=Auto, 3=Manual, 4=Disabled) |
| `Type` | Service type (own process, shared, kernel driver) |

## Privilege escalation vectors
[[PowerUp]] automates finding these:

| Misconfiguration | Attack |
|---|---|
| **Writable service binary path** | Replace the EXE → service restarts as SYSTEM |
| **Unquoted service path** with spaces | Drop a binary in a parent directory that matches the unquoted tokenization |
| **Weak service DACL** | `sc config <svc> binpath= "cmd /c <payload>"` → restart service |
| **Writable service registry key** | Modify `ImagePath` directly |
| **DLL hijacking** | Service loads a DLL from a writable directory |

## Key commands
```cmd
# List services
sc query state= all
Get-Service

# Service details
sc qc <SERVICE_NAME>
Get-WmiObject Win32_Service | Select Name, StartName, PathName, State

# Modify service binary path (requires appropriate DACL)
sc config <SERVICE_NAME> binpath= "<NEW_PATH>"

# Service permissions (Sysinternals)
accesschk.exe -ucqv <SERVICE_NAME>
accesschk.exe -uwcqv "Authenticated Users" *
```

## Related
- Credential storage: service account passwords in [[LSA Secrets]].
- Escalation toolkit: [[PowerUp]] (service abuse modules).
- Privilege context: [[Access Token]], [[Privileges and Rights]].
