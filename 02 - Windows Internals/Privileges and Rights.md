---
title: Privileges and Rights
aliases: ["Privileges and Rights", "SeDebugPrivilege", "SeImpersonatePrivilege", "SeBackupPrivilege", "SeRestorePrivilege", "SeLoadDriverPrivilege", "SeEnableDelegationPrivilege", "Token privileges"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals, attack/privesc, attack/persistence, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Access Token]]", "[[LSASS Dumping]]", "[[NTDS.dit Extraction]]", "[[UserAccountControl]]", "[[Constrained Delegation]]", "[[Kerberos Delegation]]", "[[DCSync]]", "[[GPO Abuse]]", "[[Group Policy Object (GPO)]]", "[[ACL Abuse]]", "[[PowerView]]", "[[Rubeus]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Privileges and Rights

> [!summary] One-liner
> Named capabilities attached to a token; several are effectively "local SYSTEM" or "read any secret" if you hold them.

## High-value privileges (and the abuse)
| Privilege | Abuse |
|---|---|
| **SeDebugPrivilege** | Open any process → dump [[LSASS]] ([[LSASS Dumping]]) |
| **SeBackupPrivilege** | Read any file → copy `ntds.dit` / SAM hives ([[NTDS.dit Extraction]]) |
| **SeRestorePrivilege** | Write any file → with SeBackup = arbitrary file overwrite → SYSTEM |
| **SeImpersonatePrivilege** | Impersonate a token → "Potato" attacks (PrintSpoofer/JuicyPotato) → **SYSTEM** |
| **SeAssignPrimaryTokenPrivilege** | Assign primary token to a process → privesc |
| **SeLoadDriverPrivilege** | Load a kernel driver → kernel-level code exec |
| **SeTakeOwnershipPrivilege** | Take ownership of objects → then rewrite ACLs |
| **SeEnableDelegationPrivilege** | Configure Kerberos delegation flags on accounts ([[UserAccountControl]]) — see harmj0y section below |
| **SeTcbPrivilege** | "Act as part of the OS" — the highest, total control |

## Why a red teamer cares
After landing on a host, **`whoami /priv`** is a first check: a single enabled privilege (esp. SeImpersonate on a server, or SeBackup) is often an instant local-SYSTEM or credential-theft win.

## More from harmj0y (SeEnableDelegationPrivilege — "the most dangerous user right")

> [!summary] SeEnableDelegationPrivilege
> A user right that controls who is allowed to flip Kerberos delegation settings on accounts. Combined with write access (GenericAll/GenericWrite) over any account, it lets an attacker compromise the domain at will, indefinitely — yet it is almost never audited.

### What it gates
SeEnableDelegationPrivilege is the user right "Enable computer and user accounts to be trusted for delegation." It applies **only to domain controllers and stand-alone systems** — NOT to domain-joined workstations. Holding it is what actually permits an account to modify, on other AD objects:
- The `TRUSTED_FOR_DELEGATION` flag (unconstrained delegation) and `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` flag (protocol transition) in [[UserAccountControl]].
- The `msDS-AllowedToDelegateTo` property (constrained delegation target SPNs).

In other words, write access to the delegation-related attributes is not sufficient by itself — the account performing the change must also hold this right (enforced on DCs).

### Why it's dangerous
If an attacker controls this right AND has GenericAll/GenericWrite over **any** user object (see [[ACL Abuse]]), they gain a self-renewing path to domain compromise:
1. Modify a controlled user's `msDS-AllowedToDelegateTo` to target a sensitive SPN such as `ldap/<DOMAIN_CONTROLLER>` (constrained delegation to the DC).
2. Set the `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` flag on that user so it can perform protocol transition (S4U2Self → S4U2Proxy).
3. Run the Kerberos S4U flow ([[Constrained Delegation]]) to obtain a service ticket as an admin to `ldap/<DC>`, then run a [[DCSync]] to pull hashes.
4. If the attacker has GenericAll over a victim account but doesn't know its password, force a password reset (`Set-DomainUserPassword`) and impersonate that user instead.

Because the right plus an ACL edit can be re-established at any time, harmj0y frames it as a durable persistence primitive, not just a one-shot privesc.

### Default holders
By default only **BUILTIN\Administrators** on domain controllers (i.e. Domain Admins / Enterprise Admins) hold SeEnableDelegationPrivilege.

### Enumeration / checking who holds it
- **PowerView**: `Get-DomainPolicy -Source DC` parses which GPOs applied to the domain controllers OU modify user-right assignments — typically the "Default Domain Controllers Policy" (GUID `{6AC1786C-016F-11D2-945F-00C04FB984F9}`). Inspect the resulting policy for the `SeEnableDelegationPrivilege` line.
- The underlying setting lives in the GPO's security template (`GptTmpl.inf`).

### Persistence via GPO (granting yourself the right)
An attacker with edit access to the default Domain Controllers GPO ([[GPO Abuse]]) can add their account SID to the SeEnableDelegationPrivilege assignment in the GPO's `GptTmpl.inf`:
```
\\<DOMAIN>\sysvol\<domain.fqdn>\Policies\{6AC1786C-016F-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf
```
The new right takes effect on DC reboot or the next Group Policy refresh.

### Tools referenced
- [[PowerView]] — DACL analysis, privilege/policy enumeration (`Get-DomainPolicy -Source DC`), `Set-DomainUserPassword`.
- `asktgt.exe` / `s4u.exe` (Kekeo-style utilities; the modern equivalent is [[Rubeus]] `s4u`) — execute the S4U2Self/S4U2Proxy delegation flow.

### Detection / artifacts
- Enable authorization-policy-change auditing and watch for **Event ID 4704** (a user right was assigned) containing `SeEnableDelegationPrivilege`.
- Monitor modifications to the Default Domain Controllers Policy GPO and to `msDS-AllowedToDelegateTo` / delegation [[UserAccountControl]] flags.

### Mitigation
- Audit exactly which accounts hold SeEnableDelegationPrivilege on domain controllers; keep it limited to Administrators.
- Monitor GPO changes to the default DC policy and enable authorization-policy-change auditing.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Privileges"
- [[Source - harmj0y blog]] — "The Most Dangerous User Right You (Probably) Have Never Heard Of" — https://blog.harmj0y.net/activedirectory/the-most-dangerous-user-right-you-probably-have-never-heard-of/
