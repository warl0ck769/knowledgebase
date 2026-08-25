---
title: GhostPack
aliases: [Seatbelt, SharpUp, SharpRoast, SharpDump, SafetyKatz, SharpWMI]
type: tool
domain: [red-team, windows-internals, active-directory]
tags: [type/tool, domain/red-team, domain/windows-internals, tool/csharp, defense/amsi-evasion]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/redteaming/ghostpack/
verified: true
related: ["[[PowerUp]]", "[[PowerView]]", "[[Rubeus]]", "[[Kerberoasting]]", "[[LSASS Dumping]]", "[[Pass-the-Hash]]", "[[Empire]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary] GhostPack is a SpecterOps collection of C#/.NET ports of popular PowerShell offensive tools, built to retain .NET library access while sidestepping PowerShell v5+ defenses (script block logging, AMSI, constrained language mode).

## What it does
GhostPack reimplements offensive tradecraft in C# rather than PowerShell. As defenders hardened PowerShell (v5+ protections, script block logging, AMSI), the offensive community pivoted to C# — keeping full access to existing .NET libraries while gaining new weaponization/execution vectors, at the cost of losing the PowerShell pipeline and one-liner in-memory stubs. harmj0y frames PowerShell as "a great 'gateway drug' to C#." Tools are distributed as source only (no binaries, to avoid author-identifying signatures) and compile in Visual Studio Community 2015.

The initial release bundled six tools:

- **Seatbelt** — host situational-awareness / safety-check clearinghouse (40+ checks): OS info, token privileges, UAC/PowerShell/audit/WEF settings, browser history (Firefox/Chrome/IE), saved RDP/Putty sessions, recent files, Kerberos tickets, logon events, Recycle Bin, network connections, AV detection. Influenced by Lee Christensen's `Get-HostProfile.ps1` and Andrew Chiles' `HostEnum.ps1`.
- **SharpUp** — C# port of [[PowerUp]] privilege-escalation *checks only* (no exploitation): modifiable services/service binaries, `AlwaysInstallElevated`, %PATH% hijacking, modifiable registry autoruns, special token-group privileges, unattended-install files, McAfee `SiteList.xml`.
- **SharpRoast** — C# [[Kerberoasting]]: requests [[Service Principal Name (SPN)]] tickets and outputs Hashcat-format hashes for offline cracking; supports trusted/cross-domain targets and explicit credentials. (Functionality later superseded by [[Rubeus]].)
- **SharpDump** — minidumps a process (LSASS by default) via the `MiniDumpWriteDump` Win32 API, GZip-compresses the dump to a `.bin`, and deletes the raw minidump. See [[LSASS Dumping]].
- **SafetyKatz** — combines SharpDump + a customized, embedded Mimikatz + subTee's .NET PE loader to extract creds in-memory: dumps LSASS to `C:\Windows\Temp\debug.bin`, loads Mimikatz via the PE loader, runs `sekurlsa::logonpasswords` + `sekurlsa::ekeys` against the dump, then deletes it — avoiding Mimikatz attaching to LSASS directly.
- **SharpWMI** — C# WMI wrapper for enumeration and remote execution: arbitrary WQL queries, remote process creation via `Win32_Process`, and remote VBS execution via timer-based WMI event subscriptions (`ActiveScriptEventConsumer`) with cleanup.

## Key commands/functions
```text
# Seatbelt — situational awareness
Seatbelt.exe system          # system checks
Seatbelt.exe user            # user checks
Seatbelt.exe all             # run all checks (use "full" flag to disable filtering)

# SharpUp — privesc enumeration (checks only)
SharpUp.exe

# SharpRoast — Kerberoasting (Hashcat output)
SharpRoast.exe all
SharpRoast.exe harmj0y
SharpRoast.exe "OU=TestingOU,DC=testlab,DC=local"

# SharpDump — minidump LSASS (or PID) -> compressed .bin
SharpDump.exe                # dumps LSASS by default
SharpDump.exe 8700           # dump process by PID
#  -> process .bin offline:  mimikatz "sekurlsa::minidump <file>"

# SafetyKatz — dump + embedded Mimikatz in-memory
SafetyKatz.exe

# SharpWMI — query / create / executevbs
SharpWMI.exe action=query computername=<HOST> query="<WQL>"
SharpWMI.exe action=create computername=<HOST> command="<CMD>"
SharpWMI.exe action=executevbs computername=primary.testlab.local
```

## Used in techniques
- [[Kerberoasting]] — SharpRoast (later [[Rubeus]])
- [[LSASS Dumping]] — SharpDump, SafetyKatz
- [[Pass-the-Hash]] / [[Overpass-the-Hash]] — creds harvested via SafetyKatz/SharpDump
- Privilege escalation enumeration — SharpUp (see [[PowerUp]])
- Host/credential recon — Seatbelt (see [[Credential Hunting]])
- Lateral movement / remote execution — SharpWMI

## Sources
- harmj0y, "GhostPack" — https://blog.harmj0y.net/redteaming/ghostpack/
