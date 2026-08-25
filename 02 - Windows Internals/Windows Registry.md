---
title: Windows Registry
aliases: ["registry", "regedit", "reg.exe", "registry hive"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/sysinfo/registry"
verified: true
related: ["[[SAM Database]]", "[[LSA Secrets]]", "[[Credential Storage in Windows]]", "[[UAC]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Windows Registry

> [!summary] One-liner
> A hierarchical database that stores OS configuration, user settings, installed software info, and security data — many credential stores ([[SAM Database]], [[LSA Secrets]]) are registry hives on disk.

## Root keys (hives)

| Hive | Abbreviation | Scope | Key contents |
|---|---|---|---|
| `HKEY_LOCAL_MACHINE` | HKLM | Machine-wide | `SAM`, `SECURITY`, `SYSTEM`, `SOFTWARE` sub-hives |
| `HKEY_CURRENT_USER` | HKCU | Per-user (loaded profile) | User preferences, environment, `Software\` settings |
| `HKEY_USERS` | HKU | All loaded user profiles | Contains every loaded user's hive |
| `HKEY_CLASSES_ROOT` | HKCR | Merged view | COM classes, file associations (HKLM + HKCU merge) |
| `HKEY_CURRENT_CONFIG` | HKCC | Hardware profile | Current hardware config |

## Hive files on disk

| Hive | Path |
|---|---|
| SAM | `C:\Windows\System32\config\SAM` |
| SECURITY | `C:\Windows\System32\config\SECURITY` |
| SYSTEM | `C:\Windows\System32\config\SYSTEM` |
| SOFTWARE | `C:\Windows\System32\config\SOFTWARE` |
| User (NTUSER.DAT) | `C:\Users\<user>\NTUSER.DAT` |

These files are locked by the OS while running — offline access (boot from USB, volume shadow copy) or the `reg save` command can extract them.

## Security-critical registry paths

| Path | Significance |
|---|---|
| `HKLM\SAM` | Local user password hashes → [[SAM Database]] |
| `HKLM\SECURITY\Policy\Secrets` | [[LSA Secrets]] (service passwords, machine account, DPAPI_SYSTEM) |
| `HKLM\SYSTEM\CurrentControlSet\Control\LSA` | Security settings: `RunAsPPL`, `DisableRestrictedAdmin`, WDigest `UseLogonCredential` |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | [[UAC]] configuration (`EnableLUA`, `ConsentPromptBehaviorAdmin`) |
| `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest` | `UseLogonCredential` = 1 forces plaintext password caching → [[WDigest Downgrade]] |
| `HKCU\Software\Classes\ms-settings\shell\open\command` | [[UAC Bypass]] hijack key (fodhelper) |

## Red-team relevance
- **Credential extraction**: `reg save HKLM\SAM sam.save` + `SYSTEM` hive → offline hash extraction.
- **Persistence**: Run keys (`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`), services, scheduled tasks.
- **Configuration abuse**: Enabling WDigest, disabling LSA protection, modifying UAC settings.
- **GPP/SYSVOL**: Group Policy preferences stored passwords (now patched but still found) — [[GPP Passwords]].

## Key commands
```cmd
reg query HKLM\SYSTEM\CurrentControlSet\Control\LSA
reg save HKLM\SAM C:\temp\sam.save
reg save HKLM\SYSTEM C:\temp\system.save
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
```

## Related
- Credential hives: [[SAM Database]], [[LSA Secrets]], [[Credential Storage in Windows]].
- Configuration targets: [[UAC]], [[WDigest Downgrade]].
