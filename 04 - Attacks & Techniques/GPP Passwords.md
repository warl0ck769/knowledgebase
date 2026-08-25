---
title: GPP Passwords
aliases: ["GPP cpassword", "Group Policy Preferences passwords", "Get-GPPPassword", "cpassword"]
type: technique
domain: [active-directory, red-team]
attack_tactic: [credential-access, lateral-movement, privesc]
tags: [type/technique, domain/red-team, domain/active-directory, attack/credential-access, attack/lateral-movement]
source: ["[[Source - harmj0y blog]]"]
source_url: "https://blog.harmj0y.net/powershell/gpp-and-powerview/"
verified: true
related: ["[[Group Policy Object (GPO)]]", "[[GPO Abuse]]", "[[PowerView]]", "[[Credential Hunting]]", "[[Pass-the-Hash]]", "[[LAPS]]", "[[Privileged AD Groups]]"]
created: 2026-06-07
updated: 2026-06-07
---

# GPP Passwords

> [!summary] One-liner
> Group Policy Preferences (GPP) can push credentials (local admin, service, scheduled-task, mapped-drive accounts) into XML files in **SYSVOL**, where the password is stored as a reversibly-encrypted **`cpassword`** field whose AES key Microsoft published publicly — so any authenticated domain user who can read SYSVOL can recover the plaintext.

## Concept abused
Group Policy Preferences is a feature of [[Group Policy Object (GPO)]] that lets admins configure things like local users/groups, services, scheduled tasks, mapped drives, data sources, and printers across the domain. When a preference includes a password, it is written into the GPO's policy files in **SYSVOL** as a **`cpassword`** attribute.

The catch: `cpassword` is **encrypted with a fixed AES-256 key that Microsoft published in MSDN documentation** [UNVERIFIED]. Because the key is public, the encryption is effectively reversible by anyone. SYSVOL is **world-readable by all authenticated domain users**, so any low-privileged account can read these files and decrypt the embedded credentials — frequently yielding a **local administrator password reused across many machines**, which makes this a fast path to [[Pass-the-Hash]] / [[Credential Hunting]]-style lateral movement.

## Prerequisites
- Any **authenticated domain user** (read access to `\\<domain>\SYSVOL`).
- A GPP preference that actually stored a `cpassword` (i.e. an admin used GPP to set a password). [UNVERIFIED — frequency depends on environment]

## How the attack works
1. **Locate GPP files in SYSVOL.** GPP policy XML lives under each GPO's folder in `\\<domain>\SYSVOL\<domain>\Policies\{<GPO-GUID>}\`. Files that can contain `cpassword` include [UNVERIFIED — specific file list not enumerated in the assigned post]:
   - `Groups.xml` (local users/groups — the classic local-admin password)
   - `Services.xml` (service account)
   - `ScheduledTasks.xml` (scheduled-task run-as account)
   - `DataSources.xml`
   - `Drives.xml` (mapped-drive credentials)
   - `Printers.xml`
2. **Extract the `cpassword`** value from the XML.
3. **Decrypt it** using the publicly known AES key to recover the plaintext password. [UNVERIFIED — key/algorithm specifics not in the assigned post]
4. **Use the credential** — often a reused local-admin account — to move laterally. Map the recovered GPP back to the machines it applies to (see PowerView workflow below) so you know exactly where the credential is valid.

> [!note] MS14-025 (the partial fix)
> Microsoft's **MS14-025** patch **stops admins from creating new GPP preferences that store passwords** (it removes the password fields from the GPP UI), but it **does NOT remove `cpassword` values that already exist in SYSVOL**. Legacy GPP files therefore remain exploitable on patched domains. [UNVERIFIED — MS14-025 details not in the assigned post]

## Commands & tools

### Recover the password ([[PowerView]] / PowerSploit)
**`Get-GPPPassword`** parses GPP XML files in SYSVOL, extracts `cpassword` values, decrypts them, and returns the recovered credentials. Its output also includes the **file path** of the source XML, which contains the **GPO GUID** — the pivot used below.
```powershell
# Search SYSVOL for GPP passwords and decrypt them
Get-GPPPassword
```

### Pivot from a recovered GPP to the machines it affects ([[PowerView]])
The assigned post's core content: once `Get-GPPPassword` gives you the GPO GUID from the returned file path (e.g. `{31B2F340-016D-11D2-945F-00C04FB984F9}`), use PowerView to find exactly which OUs the GPO is linked to and which computers fall under them — so you know where the recovered credential is usable.
```powershell
# Find all machines affected by a specific GPP, by its GUID
Get-NetOU -GUID "{31B2F340-016D-11D2-945F-00C04FB984F9}" | %{ Get-NetComputer -ADSPath $_ }
```
- **`Get-NetOU -GUID <GPP_GUID>`** — returns OU objects whose `gPLink` references that GPO GUID (i.e. the OUs the policy applies to).
- **`Get-NetComputer -ADSPath <OUPath>`** — returns all computers under a given OU/ADS path; piping the OU results in enumerates every machine the policy hits.
- **`-FullData`** — pass to either `Get-NetComputer` or `Get-NetOU` to return the full object data instead of just names/paths.
- **`Get-NetSite -GUID <GPP_GUID>`** — same GUID-filtering approach to find affected **sites** (GPOs can also be linked at the site level).

### Other tooling for cpassword (commonly referenced)
- **Metasploit** post module for GPP password harvesting. [UNVERIFIED — not in the assigned post]
- **`gpp-decrypt`** (Kali) — decrypts a `cpassword` string from the command line. [UNVERIFIED]
- **`gpprefdecrypt.py`** — Python decryptor for `cpassword`. [UNVERIFIED]

## Detection / artifacts
- **SYSVOL reads of GPP XML** (`Groups.xml`, etc.) by non-admin accounts — anomalous LDAP/SMB access to Policy folders. [UNVERIFIED]
- LDAP queries enumerating OUs/computers by GPO GUID (the PowerView pivot) — bulk `gPLink`/OU/computer enumeration from a single host.
- Presence of any `cpassword` value in SYSVOL is itself an indicator of exposure (audit for it proactively).

## Mitigation
- **Apply MS14-025** to block creation of new GPP passwords. [UNVERIFIED — MS14-025 specifics not in the assigned post]
- **Remove existing `cpassword` values from SYSVOL** — the patch does not clean up legacy preferences; grep SYSVOL for `cpassword` and delete the offending GPP XML. [UNVERIFIED]
- **Rotate** any password that was ever stored in a GPP preference (treat it as compromised), especially shared local-admin passwords.
- Replace GPP-pushed local-admin passwords with **[[LAPS]]** (randomized, per-machine, access-controlled).
- Restrict and monitor who can read SYSVOL Policy folders where feasible.

## Related
- [[Group Policy Object (GPO)]]
- [[GPO Abuse]]
- [[PowerView]]
- [[Credential Hunting]]
- [[Pass-the-Hash]]
- [[LAPS]]
- [[Privileged AD Groups]]

## Sources
- [[Source - harmj0y blog]] — "GPP and PowerView" — https://blog.harmj0y.net/powershell/gpp-and-powerview/

> [!note] Sourcing scope
> The assigned harmj0y post ("GPP and PowerView") specifically covers the **PowerView pivot** — using the GPO GUID returned by `Get-GPPPassword` with `Get-NetOU -GUID` and `Get-NetComputer -ADSPath` to map a GPP to its affected machines/sites. The broader GPP background (the public AES `cpassword` key, the specific list of vulnerable XML files, MS14-025, and the `gpp-decrypt`/`gpprefdecrypt.py`/Metasploit tooling) is **not** detailed in that post and is marked `[UNVERIFIED]` above; verify against MS14-025 and PowerSploit `Get-GPPPassword` documentation before relying on those specifics.
