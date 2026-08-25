---
title: Process and Thread
aliases: ["process", "thread", "Windows process model"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/procthread/about-processes-and-threads"
verified: true
related: ["[[Access Token]]", "[[Integrity Levels]]", "[[Privileges and Rights]]", "[[LSASS]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Process and Thread

> [!summary] One-liner
> A process is an isolated execution environment (virtual address space + handles + a primary token); threads are the units of execution within it — every security decision starts with the token on the process or calling thread.

## Process
A running instance of a program. Key components:

| Component | What it is |
|---|---|
| **Virtual address space** | Private memory isolated from other processes |
| **Primary [[Access Token]]** | Security context — user SID, groups, privileges, integrity level |
| **Handle table** | References to kernel objects (files, registry keys, other processes) |
| **PEB (Process Environment Block)** | User-mode structure with image base, command line, environment variables |
| **PID** | Unique numeric identifier |

## Thread
A unit of execution within a process. Shares the process's address space and handles.

| Component | What it is |
|---|---|
| **Stack** | Per-thread call stack |
| **TEB (Thread Environment Block)** | User-mode structure (TLS, exception chains) |
| **Impersonation token** | Optional — lets this thread act as a different user than the process's primary token |
| **TID** | Unique numeric identifier |

## Security model
- **Process token** = "who is this process running as?" Set at creation, inherited from parent.
- **Thread impersonation token** = "is this thread temporarily acting as someone else?" Used by services handling client requests.
- When a thread makes a security-checked call, Windows checks the **thread token first** (if impersonating), then falls back to the **process token**.

## Red-team relevance
- **Process injection**: Injecting code into a process with a better token (e.g., `SYSTEM`) gives access to that token's context — `CreateRemoteThread`, APC injection, process hollowing.
- **Token impersonation**: Duplicating a thread's impersonation token from one process and applying it to your own — the basis of `SeImpersonatePrivilege`-based attacks (potatoes).
- **LSASS**: [[LSASS]] (`lsass.exe`) is a high-value process because it holds credential material; dumping its memory yields hashes and tickets.
- **PPID spoofing**: Creating a process with a spoofed parent PID to inherit a different token or evade parent-child detection heuristics.

## Key commands
```powershell
# List processes with owner
tasklist /V
Get-Process | Select-Object Id, ProcessName, SessionId

# Process details (Sysinternals)
handle.exe -p <PID>            # open handles
listdlls.exe -p <PID>          # loaded DLLs
```

## Related
- Security context: [[Access Token]], [[Integrity Levels]], [[Privileges and Rights]].
- Key target process: [[LSASS]].
