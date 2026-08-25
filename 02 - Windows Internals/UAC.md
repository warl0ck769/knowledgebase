---
title: UAC
aliases: ["User Account Control", "UAC consent prompt", "admin approval mode"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/"
verified: true
related: ["[[Access Token]]", "[[Integrity Levels]]", "[[UAC Bypass]]", "[[Privileges and Rights]]"]
created: 2026-06-07
updated: 2026-06-07
---

# UAC (User Account Control)

> [!summary] One-liner
> A Windows security feature that forces admin-group members to run with a filtered (medium-integrity) token by default, requiring explicit consent to use their full elevated (high-integrity) token.

## How it works
1. At logon, Windows creates **two tokens** for any local admin: a **filtered token** (medium integrity, admin SIDs disabled) and a **full token** (high integrity, all groups/privileges).
2. Explorer.exe and all child processes get the **filtered token** — no admin powers by default.
3. When a process requests elevation (via manifest `requireAdministrator`, or right-click → "Run as administrator"), the **consent prompt** appears.
4. If approved, the process receives the **full token** and runs at high integrity.

## Settings (slider levels)

| Level | Behavior |
|---|---|
| **Always notify** (max) | Prompt for both app elevation AND Windows settings changes — defeats most [[UAC Bypass]] methods |
| **Notify only for apps** (default) | Prompt for third-party elevation; auto-elevate signed Windows binaries |
| **Notify (dim desktop off)** | Same but without the secure desktop |
| **Never notify** (off) | UAC effectively disabled; no split token, everything runs elevated |

Registry: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
- `EnableLUA` = 1 (UAC on) / 0 (UAC off)
- `ConsentPromptBehaviorAdmin` = 0–5 (controls prompt behavior)

## Auto-elevation
Certain signed Microsoft executables have `<autoElevate>true</autoElevate>` in their manifest. At the default UAC level, these silently elevate without a prompt. This is the root cause of most [[UAC Bypass]] techniques — attackers hijack input to auto-elevating binaries (registry keys, DLLs) to piggyback on the silent elevation.

## Red-team relevance
- UAC is **not** a security boundary (Microsoft's stated position) — it prevents accidental damage, not determined attackers.
- [[UAC Bypass]] goes from medium → high integrity within the same admin account.
- `LocalAccountTokenFilterPolicy` (LATFP): when set to 1, remote logons for local admins get the **full** token (no filtering) — relevant for [[Pass-the-Hash]] over the network.

## Related
- Tokens it splits: [[Access Token]] (primary vs. filtered).
- Integrity model it relies on: [[Integrity Levels]].
- Bypass techniques: [[UAC Bypass]].
- Remote filtering: [[Pass-the-Hash]] (LATFP).
