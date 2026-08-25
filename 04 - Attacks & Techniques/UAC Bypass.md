---
title: UAC Bypass
aliases: ["UAC bypass", "Invoke-BypassUAC", "auto-elevate", "Write-HijackDll"]
type: technique
domain: [red-team]
attack_tactic: [privesc, defense-evasion]
tags: [type/technique, domain/red-team, attack/privesc]
source: "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/powershell/invoke-bypassuac/"
verified: true
related: ["[[Privileges and Rights]]", "[[PowerUp]]", "[[Empire]]"]
created: 2026-06-07
updated: 2026-06-07
---

# UAC Bypass

> [!summary] One-liner
> Go from a medium-integrity admin context to a high-integrity (elevated) one without a consent prompt, by abusing Windows' auto-elevation behavior or DLL hijacking in elevated COM objects.

## Concept abused
**UAC (User Account Control)** runs even members of the local Administrators group with a **medium-integrity** token by default; elevation to **high integrity** normally requires a consent prompt. Certain signed Microsoft binaries and COM objects **auto-elevate** silently, and several of them load attacker-influenced input (registry keys, DLLs, manifests). Hijacking that input lets an already-admin (but unelevated) process spawn a high-integrity child **without triggering the consent dialog**.

> [!note]
> UAC bypass is **not** a privilege escalation across users — it requires you to already be in the Administrators group. It elevates *within* that account (medium → high integrity). True low-priv → admin is a different problem (see [[PowerUp]]).

## harmj0y's techniques

### 1. DLL hijacking via self-elevating COM objects
harmj0y's **Invoke-BypassUAC** used DLL hijacking combined with batch script execution through **self-elevating COM objects**:

1. **`Write-HijackDll`** (from [[PowerUp]]) generates a DLL that, when loaded, executes a supplied `.bat` file.
2. The attacker places this DLL where a self-elevating COM object will load it.
3. When the COM object instantiates (auto-elevated, high integrity), it loads the hijack DLL → the DLL runs the batch file → the batch file executes the attacker's payload **at high integrity**.

### 2. WSH `wscript.exe` exploit (Windows 7 only)
harmj0y documented that on **Windows 7**, `wscript.exe` **lacked an embedded application manifest** — the Windows Script Host would auto-elevate when invoked through certain code paths. This allowed running a `.vbs`/`.js` script at high integrity without a prompt. **Fixed in Windows 8+** (Microsoft added a manifest to `wscript.exe`).

## Common technique families (broader landscape)

| Family | Idea |
|---|---|
| **Registry hijack** (fodhelper, eventvwr, sdclt, computerdefaults) | Plant a command in a per-user registry key the auto-elevating binary reads (e.g., `HKCU\Software\Classes\...\shell\open\command`) → it runs elevated |
| **DLL hijack / side-load** | Drop a DLL an auto-elevating binary or COM object loads from a writable path |
| **Token/IFileOperation (ICMLuaUtil etc.)** | Abuse elevated COM interfaces to copy files or run commands as high integrity |
| **Manifest-less binaries** | Target signed Microsoft binaries that lack an embedded manifest (OS-version-specific) |

**UACME** (by hfiref0x) catalogs dozens of methods by ID number and tracks which Windows versions each works on.

## Prerequisites
- Membership in local **Administrators** group.
- Running at **medium integrity** (the default for admin tokens under UAC).
- UAC is **not** set to "Always notify" (the maximum setting defeats most bypasses).

## Commands & tools

### Empire modules
```
# DLL hijack + COM object method (primary)
usemodule privesc/bypassuac
set Listener <LISTENER_NAME>
execute

# WSH wscript.exe method (Windows 7 only)
usemodule privesc/bypassuac_wscript
set Listener <LISTENER_NAME>
execute
```

### PowerUp (Write-HijackDll)
```powershell
# Generate a hijack DLL that executes a batch file when loaded
Write-HijackDll -DllPath <OUTPUT.dll> -Command "<BATCH_FILE_PATH>"
```

### Manual fodhelper bypass (modern, common)
```powershell
# Registry hijack: fodhelper.exe reads this key and executes the value elevated
New-Item -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "(Default)" -Value "<PAYLOAD.exe>" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "DelegateExecute" -Value "" -Force

# Trigger — fodhelper auto-elevates and reads the hijacked key
Start-Process "C:\Windows\System32\fodhelper.exe"

# Cleanup
Remove-Item -Path "HKCU:\Software\Classes\ms-settings" -Recurse -Force
```

### Check current integrity level
```powershell
whoami /groups | findstr "Label"
# Medium Mandatory Level = not elevated
# High Mandatory Level   = elevated
```

## Detection / artifacts
- New or unexpected values under `HKCU\Software\Classes\...\shell\open\command` (watch for `ms-settings`, `mscfile`, `Folder` keys).
- Auto-elevating binaries (`fodhelper.exe`, `eventvwr.exe`, `sdclt.exe`, `computerdefaults.exe`) spawning `cmd.exe`, `powershell.exe`, or other unexpected children.
- Integrity-level jump (medium → high) with no corresponding consent UI event.
- Sysmon Event 1 (process create) correlating parent/child where parent is an auto-elevating binary.
- DLL writes to known hijack paths (e.g., `C:\Windows\System32\` or side-load directories).

## Mitigation
- Set UAC to **"Always notify"** (maximum slider position) — defeats most bypasses by requiring a prompt even for auto-elevating binaries.
- Don't grant users local admin unless necessary.
- Monitor the known hijack registry keys (`HKCU\Software\Classes\ms-settings\`, `HKCU\Software\Classes\mscfile\`, etc.) for unexpected writes.
- Application control (WDAC / AppLocker) to restrict what auto-elevating binaries can spawn.
- Keep Windows updated — Microsoft patches individual bypass vectors as they're disclosed.

## Related
- Integrity levels and token mechanics: [[Privileges and Rights]].
- Local privilege escalation (non-admin → admin): [[PowerUp]].
- [[Empire]] implements multiple UAC bypass modules.

## Sources
- [[Source - harmj0y blog]] — "Invoke-BypassUAC" — https://blog.harmj0y.net/powershell/invoke-bypassuac/
