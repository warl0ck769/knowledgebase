---
title: Integrity Levels
aliases: ["integrity level", "mandatory integrity control", "MIC", "IL"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/secauthz/mandatory-integrity-control"
verified: true
related: ["[[Access Token]]", "[[UAC]]", "[[Privileges and Rights]]", "[[ACL, ACE, DACL, SACL]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Integrity Levels

> [!summary] One-liner
> A mandatory access control layer in Windows that labels every process and securable object with a trust level — a process cannot write to objects at a higher integrity, regardless of DACL permissions.

## The four levels

| Level | SID | RID | Typical subjects |
|---|---|---|---|
| **System** | `S-1-16-16384` | 16384 | `SYSTEM`, kernel-mode services |
| **High** | `S-1-16-12288` | 12288 | Elevated admin processes (after UAC consent) |
| **Medium** | `S-1-16-8192` | 8192 | Standard user processes, unelevated admin shell |
| **Low** | `S-1-16-4096` | 4096 | Sandboxed processes (browser tabs, AppContainers) |
| **Untrusted** | `S-1-16-0` | 0 | Rarely used; most restricted |

## How it works
1. Every [[Access Token]] carries an integrity level (set at logon, inherited by child processes).
2. Every [[Securable Objects|securable object]] has a **mandatory label** (in its SACL — see [[ACL, ACE, DACL, SACL]]).
3. **Before** the DACL is checked, Windows enforces the **no-write-up** policy: a process at Medium cannot write to a High-integrity object.

Default policy bits:
- **No-Write-Up** (default) — blocks writes from lower to higher.
- **No-Read-Up** — blocks reads (rarely set by default).
- **No-Execute-Up** — blocks execution.

## Why it matters for red team
- [[UAC]] splits admin tokens: the shell runs at **Medium**, and elevation gives **High**. [[UAC Bypass]] techniques cross this boundary without a prompt.
- Services running as `SYSTEM` are at **System** integrity — compromising one gives the highest local level.
- Sandboxed processes (Low integrity) cannot write to most user-profile paths, limiting post-exploitation options without an escape.

## Checking integrity level
```powershell
# Current process integrity
whoami /groups | findstr "Label"

# Via Process Explorer / Process Hacker: "Integrity" column
```

## Related
- Carried inside: [[Access Token]].
- Enforced alongside: [[ACL, ACE, DACL, SACL]] (mandatory label ACE in SACL).
- Elevation across levels: [[UAC]], [[UAC Bypass]].
