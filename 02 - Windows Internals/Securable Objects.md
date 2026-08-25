---
title: Securable Objects
aliases: ["securable object", "security descriptor"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/secauthz/securable-objects"
verified: true
related: ["[[ACL, ACE, DACL, SACL]]", "[[Access Token]]", "[[SID]]", "[[Security Principal]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Securable Objects

> [!summary] One-liner
> Any Windows resource that has a **security descriptor** — an owner, a DACL (who can access), and a SACL (what gets audited) — controlling who can do what with it.

## What qualifies as a securable object

| Category | Examples |
|---|---|
| **File system** | Files, directories (NTFS) |
| **Registry** | Registry keys |
| **Kernel objects** | Processes, threads, mutexes, events, semaphores, timers |
| **Services** | Windows service control entries |
| **Active Directory** | AD objects (users, groups, OUs, GPOs) — see [[ACL, ACE, DACL, SACL]] |
| **Network shares** | SMB shares |
| **Printers** | Printer objects |
| **Window stations / desktops** | Interactive session objects |

## Security descriptor structure
Every securable object has a **security descriptor** containing:

```
Security Descriptor
├── Owner SID          ← who owns the object
├── Group SID          ← primary group (mostly for POSIX compat)
├── DACL               ← list of ACEs controlling access (Allow/Deny)
└── SACL               ← list of ACEs controlling auditing + mandatory label
```

See [[ACL, ACE, DACL, SACL]] for how DACLs and SACLs are evaluated.

## Access check flow
When a process (carrying an [[Access Token]]) tries to access a securable object:
1. **Mandatory integrity check** — is the token's [[Integrity Levels|integrity level]] high enough? (no-write-up policy)
2. **Owner check** — the owner always gets `READ_CONTROL` and `WRITE_DAC`.
3. **DACL evaluation** — ACEs are evaluated in order; first match (Deny before Allow) wins.

## Red-team relevance
- [[ACL Abuse]] targets weak DACLs on AD objects (e.g., `GenericAll` on a user → reset password, write SPN for [[Kerberoasting]]).
- Misconfigured service DACLs → service binary replacement → privilege escalation ([[PowerUp]]).
- File/directory DACLs on sensitive paths (e.g., `C:\Windows\NTDS\`, GPO SYSVOL folders) control whether an attacker can read/write critical data.

## Related
- ACL mechanics: [[ACL, ACE, DACL, SACL]].
- Identity checked: [[Access Token]], [[SID]], [[Security Principal]].
- Integrity enforcement: [[Integrity Levels]].
