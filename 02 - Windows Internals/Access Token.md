---
title: Access Token
aliases: ["access token", "security token", "impersonation token", "primary token"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/secauthz/access-tokens"
verified: true
related: ["[[SID]]", "[[Integrity Levels]]", "[[Privileges and Rights]]", "[[Security Principal]]", "[[ACL, ACE, DACL, SACL]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Access Token

> [!summary] One-liner
> A kernel object assigned to every logon session that carries the user's identity (SIDs), group memberships, privileges, and integrity level — Windows checks it on every access decision.

## What it contains

| Field | Purpose |
|---|---|
| **User SID** | Identifies the logged-on principal ([[SID]]) |
| **Group SIDs** | All groups the user belongs to (local + domain) — including [[Privileged AD Groups]] if applicable |
| **Privilege list** | Enabled/disabled privileges (e.g., `SeDebugPrivilege`, `SeImpersonatePrivilege`) — see [[Privileges and Rights]] |
| **Integrity level** | Untrusted / Low / Medium / High / System — see [[Integrity Levels]] |
| **Logon SID** | Unique per logon session |
| **Default DACL** | Applied to objects the process creates if no explicit DACL is given |
| **Token type** | Primary or Impersonation |

## Primary vs. impersonation tokens

| Type | Assigned to | Purpose |
|---|---|---|
| **Primary** | A process | Represents the user who started the process; one per process |
| **Impersonation** | A thread | Lets a thread act as a different user temporarily (e.g., a service handling a client request) |

Impersonation tokens have a **level**: Anonymous, Identification, Impersonation, or Delegation — controlling how far the thread can use the borrowed identity.

## Filtered (split) tokens and UAC
When a member of the local Administrators group logs in with [[UAC]] enabled, Windows creates **two tokens**:
1. A **filtered token** (medium integrity, admin groups disabled) — assigned to the user's shell.
2. A **full (elevated) token** (high integrity, all groups/privileges present) — used only when elevation is approved.

This split is why [[UAC Bypass]] exists: the user *has* admin rights, but their running processes use the weaker filtered token by default.

## Red-team relevance
- **Token theft / impersonation**: Tools like Mimikatz (`token::elevate`, `token::impersonate`) and Cobalt Strike (`steal_token`) duplicate another user's token to act as them — the basis of [[Pass-the-Hash]] and lateral movement.
- **`SeImpersonatePrivilege`**: Service accounts (e.g., IIS, MSSQL) often hold this privilege, enabling potato-family attacks to impersonate `SYSTEM`.
- **Integrity level check**: `whoami /groups | findstr Label` reveals the current token's integrity — Medium means unelevated, High means admin.

## Key commands
```powershell
# View current token details
whoami /all

# Check integrity level
whoami /groups | findstr "Label"

# List token privileges
whoami /priv
```

## Related
- Identity carried: [[SID]], [[Security Principal]].
- Rights granted: [[Privileges and Rights]], [[Integrity Levels]].
- Access checks against: [[ACL, ACE, DACL, SACL]], [[Securable Objects]].
- Elevation: [[UAC]], [[UAC Bypass]].
