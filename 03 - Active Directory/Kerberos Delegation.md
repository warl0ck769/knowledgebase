---
title: Kerberos Delegation
aliases: ["Kerberos Delegation", "Delegation Abuse"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Unconstrained Delegation]]", "[[Constrained Delegation]]", "[[Resource-Based Constrained Delegation (RBCD)]]", "[[Protected Users Group]]", "[[Rubeus]]", "[[PowerView]]", "[[UserAccountControl]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Kerberos Delegation

> [!summary] One-liner
> Lets a service act *on behalf of* a user toward other services — three flavors, each with its own abuse path to privilege escalation.

## Why delegation exists
A front-end service (e.g. a web app) often needs to reach a back-end (e.g. a database) **as the user**. Delegation is the mechanism for that impersonation.

## The three flavors
| Type | Configured by | Scope | Abuse note |
|---|---|---|---|
| **[[Unconstrained Delegation]]** | `TRUSTED_FOR_DELEGATION` | **Any** service (full TGT) | Capture TGTs (esp. coerced DC) → DA |
| **[[Constrained Delegation]]** | `msDS-AllowedToDelegateTo` + `TRUSTED_TO_AUTH_FOR_DELEGATION` | Listed SPNs only | S4U2self+S4U2proxy to impersonate anyone to allowed SPNs |
| **[[Resource-Based Constrained Delegation (RBCD)]]** | `msDS-AllowedToActOnBehalfOfOtherIdentity` **on the target** | Set by the resource | Write the attribute (GenericWrite/relay) → impersonate to that target |

## Anti-delegation measures
- **[[Protected Users Group]]** members can't be delegated (any type).
- **`NOT_DELEGATED`** flag protects a specific account.
- Constrained delegation is **S4U-restricted** to its allow-list (but see "alternate service name" trick in [[Constrained Delegation]]).

## Why a red teamer cares
Delegation misconfigurations are among the most common and impactful AD privesc paths. Always enumerate the three flags during recon.

## More from harmj0y (RBCD & the sname-substitution attack chain)

harmj0y's "Another Word on Delegation" maps how the three delegation variants evolved across Windows releases and details the resource-based constrained delegation (RBCD) abuse primitive popularized by Elad Shamir.

### Historical evolution of the three variants
- **Unconstrained delegation (Windows 2000):** the user's TGT is embedded in the service ticket presented to a `TRUSTED_FOR_DELEGATION` server. That server extracts and caches the TGT in **LSASS memory**, enabling unlimited impersonation of the user across the domain (see [[Unconstrained Delegation]], [[LSASS Dumping]]).
- **Traditional constrained delegation (Windows 2003+):** introduced the **S4U2self** and **S4U2proxy** protocol extensions. Accounts whose `msDS-AllowedToDelegateTo` lists SPNs can impersonate domain users to those specific services. Editing `msDS-AllowedToDelegateTo` requires **`SeEnableDelegationPrivilege`** on a domain controller — a high bar.
- **Resource-based constrained delegation (Windows Server 2012+):** configuration moves *off* the front-end account and *onto the target resource*, stored in the target's `msDS-AllowedToActOnBehalfOfOtherIdentity` security descriptor. This only requires **edit rights on the target computer object** (GenericAll / GenericWrite / WriteDacl) — no domain admin and no `SeEnableDelegationPrivilege`. That lowered bar is what makes RBCD an attractive ACL-abuse primitive (see [[ACL Abuse]], [[Computer Accounts]]).

### S4U flow recap
The controlled account runs **S4U2self** to request a *forwardable* service ticket to itself on behalf of a chosen target user, then runs **S4U2proxy** to delegate that ticket onward to the target service.

### Key UserAccountControl requirement
To execute S4U2self, the controlling account must have **`TRUSTED_TO_AUTH_FOR_DELEGATION`** set, i.e. bit **`0x1000000` (16777216)** in [[UserAccountControl]]. (Note: with the RBCD path, control of *any* account that can perform S4U2self — including a computer/SPN account you create — is enough.)

### The sname-substitution trick (Alberto Solino)
Alberto Solino discovered that in the resulting **KRB-CRED** file the **service name (`sname`) is not cryptographically protected — only the *server* name is**. Because the sname is unprotected, **any service name can be substituted** on the obtained ticket. This means a ticket obtained for one service (e.g. `time/`) can be rewritten to a far more useful service (e.g. `cifs/`, `ldap/`, `host/`) against the same server, expanding what the delegated ticket grants. This is the same "alternate service name" idea referenced under [[Constrained Delegation]].

### PA-PAC-OPTIONS requirement for RBCD
RBCD S4U requests require the **resource-based constrained delegation bit** to be set in the **PA-PAC-OPTIONS** preauthentication data structure. Support for emitting this was added to [[Rubeus]] via Elad Shamir's pull request.

### Attack primitive: ACL-based computer takeover via RBCD
Given **write access** to a target computer object's `msDS-AllowedToActOnBehalfOfOtherIdentity` and control of an account that can perform S4U2self:
1. Modify the target computer's delegation descriptor (`msDS-AllowedToActOnBehalfOfOtherIdentity`) to permit the controlled account.
2. Use [[Rubeus]] **`s4u`** to request service tickets for the target (e.g. `cifs`, `ldap`, `host`), impersonating a privileged user — combine with the sname-substitution trick as needed.
3. Use the resulting ticket(s) for administrative access to the target system (e.g. [[Pass-the-Ticket]]).
4. Reset / clear the descriptor afterward to remove evidence (OPSEC cleanup).

## Commands & tools

**[[Rubeus]] — S4U with RBCD support** (request tickets impersonating a privileged user to a target service)
```
Rubeus.exe s4u /user:<CONTROLLED_ACCOUNT$> /rc4:<NTLM_HASH> /impersonateuser:<DOMAIN_ADMIN> /msdsspn:<cifs/TARGET.domain.local> /ptt
```
- `/altservice:<cifs,ldap,host>` leverages the sname-substitution trick to obtain additional usable service names from a single S4U2proxy result.

**[[PowerView]] — write the RBCD attribute on the target computer**
```powershell
# Grant <CONTROLLED_ACCOUNT$> the right to act on behalf of others to <TARGET$>
$sid = (Get-DomainComputer <CONTROLLED_ACCOUNT> -Properties objectsid).objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$sid)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer <TARGET> | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```
- Cleanup: `Get-DomainComputer <TARGET> | Set-DomainObject -Clear 'msds-allowedtoactonbehalfofotheridentity'`

**Kekeo** — legacy tool referenced for comparison; modern S4U/RBCD workflows are handled by [[Rubeus]].

## Detection / artifacts
- Changes to `msDS-AllowedToActOnBehalfOfOtherIdentity` on computer objects (RBCD), and to `msDS-AllowedToDelegateTo` (constrained delegation) — audit directory writes to these attributes.
- S4U2self / S4U2proxy activity: Kerberos service ticket requests (Event ID **4769**) where a service requests tickets on behalf of users it has no legitimate reason to impersonate.
- Tickets whose `sname` does not match expected usage (sname-substitution artifact).
- TGTs cached in LSASS on unconstrained-delegation hosts (see [[LSASS Dumping]]).

## Mitigation
- Add sensitive/privileged accounts to **[[Protected Users Group]]** and set the **`NOT_DELEGATED`** (account is sensitive and cannot be delegated) flag.
- Tightly control **write/edit ACLs on computer objects** to prevent RBCD attribute tampering (the RBCD bar is just object-edit rights, not DA).
- Restrict who holds `SeEnableDelegationPrivilege` on DCs (gates constrained-delegation config).
- Audit and minimize accounts with `TRUSTED_TO_AUTH_FOR_DELEGATION` and unconstrained delegation.

## Related
- [[Unconstrained Delegation]], [[Constrained Delegation]], [[Resource-Based Constrained Delegation (RBCD)]]
- [[Protected Users Group]], [[UserAccountControl]], [[Computer Accounts]]
- [[ACL Abuse]], [[ACL, ACE, DACL, SACL]]
- [[Rubeus]], [[PowerView]], [[Pass-the-Ticket]], [[LSASS Dumping]]
- [[Kerberos]], [[Kerberos Authentication Flow]], [[Service Principal Name (SPN)]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos Delegation" — https://zer1t0.gitlab.io/posts/attacking_ad/
- [[Source - harmj0y blog]] — "Another Word on Delegation" — https://blog.harmj0y.net/redteaming/another-word-on-delegation/
