---
title: Inveigh
aliases: ["Inveigh", "InveighZero", "Inveigh.exe"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Kevin Robertson"
source_url: "https://github.com/Kevin-Robertson/Inveigh"
verified: true
related: ["[[LLMNR-NBT-NS Poisoning]]", "[[WPAD and IPv6 (mitm6) Poisoning]]", "[[NetNTLM]]", "[[Responder]]", "[[NTLM Relay]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Inveigh

> [!summary] One-liner
> A Windows-native (C# / PowerShell) LLMNR/NBT-NS/mDNS poisoner and NTLM capture tool — the Windows equivalent of [[Responder]], runs without Python dependencies.

## Why use Inveigh over Responder
- Runs natively on Windows (C# exe or PowerShell) — no Python/Linux needed.
- Can run in-memory via `execute-assembly` in C2 frameworks.
- Useful when you have a Windows foothold but no Linux pivot.

## Usage
```powershell
# PowerShell version
Import-Module .\Inveigh.ps1
Invoke-Inveigh -NBNS Y -mDNS Y -LLMNR Y -ConsoleOutput Y

# C# version (InveighZero)
Inveigh.exe

# Stop
Stop-Inveigh
Get-Inveigh        # view captured hashes
```

## Captured output
- NetNTLMv1/v2 hashes in hashcat-compatible format.
- Cleartext credentials if basic auth is used.

## Related
- Linux equivalent: [[Responder]].
- Poisoning techniques: [[LLMNR-NBT-NS Poisoning]], [[WPAD and IPv6 (mitm6) Poisoning]].
- Relay: [[NTLM Relay]], [[ntlmrelayx]].
