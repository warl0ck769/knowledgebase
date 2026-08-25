---
title: PsExec
aliases: ["PsExec", "PsExec.exe", "psexec.py"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Microsoft Sysinternals / Impacket"
source_url: "https://learn.microsoft.com/en-us/sysinternals/downloads/psexec"
verified: true
related: ["[[Remote Execution & Lateral Movement (Windows)]]", "[[SMB]]", "[[Pass-the-Hash]]", "[[Impacket]]"]
created: 2026-06-07
updated: 2026-06-07
---

# PsExec

> [!summary] One-liner
> Remote command execution via SMB — uploads a service binary to `ADMIN$`, creates and starts a Windows service, and pipes I/O over named pipes. Available as Sysinternals EXE and Impacket Python script.

## How it works
1. Connects to `ADMIN$` share (SMB, port 445) on the target.
2. Uploads a service executable (`PSEXESVC.exe` or equivalent).
3. Creates a Windows service via the Service Control Manager (SCM).
4. Starts the service → executes the command as SYSTEM.
5. Pipes stdin/stdout/stderr over named pipes for interactive use.

## Usage
```bash
# Sysinternals (from Windows)
PsExec.exe \\<TARGET> -u <DOMAIN>\<USER> -p <PASS> cmd.exe
PsExec.exe \\<TARGET> -s cmd.exe       # run as SYSTEM

# Impacket (from Linux)
psexec.py <DOMAIN>/<USER>:<PASS>@<TARGET>
psexec.py -hashes :<NT_HASH> <DOMAIN>/<USER>@<TARGET>    # pass-the-hash

# CrackMapExec
nxc smb <TARGET> -u <USER> -p <PASS> --exec-method smbexec -x "whoami"
```

## OPSEC notes
- **Very noisy**: creates a service, writes a binary to disk, generates Event 7045 (service creation).
- Service name is randomized by Impacket but still detectable by behavioral rules.
- Consider stealthier alternatives: `wmiexec.py` (WMI), `atexec.py` (scheduled task), `dcomexec.py` (DCOM).

## Detection
- **Event 7045**: New service installed.
- **Event 4697**: Service installation (security log).
- PSEXESVC binary on disk at `C:\Windows\`.
- Named pipe connections to `\pipe\PSEXESVC`.

## Related
- Lateral movement overview: [[Remote Execution & Lateral Movement (Windows)]].
- Protocol: [[SMB]] (`ADMIN$` share).
- Alternatives: [[wmiexec]], [[evil-winrm]], [[Impacket]].
