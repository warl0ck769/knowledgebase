---
title: WDigest Downgrade
aliases:
  - Targeted Plaintext Downgrade
  - Reversible Encryption Downgrade
  - Plaintext Password Downgrade
  - ENCRYPTED_TEXT_PWD_ALLOWED Abuse
  - Invoke-DowngradeAccount
type: technique
domain:
  - active-directory
  - red-team
attack_tactic:
  - credential-access
  - persistence
tags:
  - type/technique
  - domain/active-directory
  - domain/red-team
  - attack/credential-access
  - attack/persistence
  - proto/ntlm
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/redteaming/targeted-plaintext-downgrades-with-powerview/
verified: true
related:
  - "[[DCSync]]"
  - "[[UserAccountControl]]"
  - "[[PowerView]]"
  - "[[ACL Abuse]]"
  - "[[Credential Storage in Windows]]"
created: 2026-06-07
updated: 2026-06-07
---

> [!summary]
> A **targeted plaintext downgrade** weakens a *specific* AD user account so that the next time their password changes it is stored in a **reversible (effectively plaintext) form** in the directory. By flipping the `ENCRYPTED_TEXT_PWD_ALLOWED` bit in the target's `userAccountControl` and forcing a password reset (`pwdLastSet = 0`), an attacker with write access to the object can later **DCSync the cleartext password** instead of just an NT hash. PowerView's `Set-ADObject` / `Invoke-DowngradeAccount` automate the per-object change.

> [!warning] Title vs. technique
> This note is filed under the title **"WDigest Downgrade"** because reversible/Digest encryption is the legacy mechanism behind WDigest/Digest authentication. The harmj0y post this note is sourced from describes the **AD-side "reversible encryption" downgrade** (per-user `ENCRYPTED_TEXT_PWD_ALLOWED` → DCSync cleartext), **not** the host-side WDigest `UseLogonCredential` registry trick. The host-side LSASS WDigest registry downgrade is a *different* technique — see [UNVERIFIED] section below; it is **not** covered by the source post. (`status/unverified` content is tagged inline.)

---

## Concept abused

**Reversible encryption** is a legacy Active Directory feature for user accounts. It exists to support authentication protocols — notably **CHAP and Digest (WDigest) authentication** — that require the server to know the cleartext password. When reversible encryption is enabled for an account, the directory stores the password in a form that can be decrypted back to plaintext (the key material is recoverable from the DC), rather than only as a one-way NT hash.

This is controlled per-account by the `ENCRYPTED_TEXT_PWD_ALLOWED` flag in the [[UserAccountControl]] (`userAccountControl`) attribute. The same `userAccountControl` bitmask is the central object for many AD abuses, so being able to XOR-toggle individual bits on a target object is a powerful primitive.

> [!important] The downgrade is **not retroactive**
> Per the source: *"if this policy is set for a particular user through whatever method, the current password is not magically turned into a reversible form, but only after the password changes."* Enabling reversible encryption does nothing to the **current** secret — the account must **change its password** before the reversible (recoverable plaintext) representation is written to the directory. This is why the attack pairs the flag flip with a forced password change.

Once the password has been changed *while the flag is set*, the cleartext is recoverable via [[DCSync]] (the same replication path used to pull NT hashes), giving the attacker the actual plaintext password — not just a hash to relay or crack.

---

## Prerequisites

- **Write access to the target object's `userAccountControl` attribute** (and ideally the ability to set `pwdLastSet`). This is typically obtained via [[ACL Abuse]] — e.g. `GenericWrite`/`GenericAll`/`WriteProperty` over the user, or membership in a group that holds such rights.
- **Replication rights** (`DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All`) on the domain to actually pull the cleartext later — i.e. the ability to [[DCSync]]. (Domain Admins, Enterprise Admins, or accounts explicitly granted the extended rights.)
- A way to cause the target's password to actually change after the flag is set (forced change-on-next-logon, or the user/service rotating it on their own).
- PowerView available in the session for the modification helpers.

---

## How the attack works

1. **Inspect the target's current UAC flags** so you know what bits are set and can confirm the change afterwards. Reversible encryption is *not* normally enabled, so the goal is to set the `ENCRYPTED_TEXT_PWD_ALLOWED` bit.
2. **Flip the `ENCRYPTED_TEXT_PWD_ALLOWED` bit** on the target's `userAccountControl` by XOR-ing with `128`. Using XOR (rather than writing an absolute value) toggles only the one bit and preserves the account's other flags. Optionally toggle `DONT_EXPIRE_PASSWORD` (`65536`) as well.
3. **Force the password to change.** Setting `pwdLastSet` to `0` marks the password as expired and forces the user to change their password on next logon. When that change happens, the new password is stored reversibly.
4. **Wait for the change**, then **DCSync** the account. Per the source: *"we can DCSync the password as soon as the `pwdlastset` date changes."* Because reversible encryption is in effect, DCSync now yields the **cleartext password**, not just the NT hash.

This is a **targeted / surgical** version of the old domain-wide "Store passwords using reversible encryption" GPO setting: instead of weakening every account, the attacker downgrades exactly one chosen account (e.g. a high-value service account or admin), making it stealthier and avoiding a broad policy change.

> [!note] Persistence angle
> Leaving `ENCRYPTED_TEXT_PWD_ALLOWED` set means that **every future password change** for that account also lands in reversible form — so an attacker who retains DCSync rights can keep recovering the cleartext indefinitely, making this a persistence/credential-access primitive, not a one-shot.

---

## Commands & tools

> Values, function names, and parameters below are taken from the source post. Treat `<SAM>` / `<USER>` placeholders as the target's `samAccountName`.

### PowerView — inspect UAC flags

```powershell
# Decode a userAccountControl integer into named flags
ConvertFrom-UACValue -Value <UAC_INT>

# Show ALL possible UAC flags, marking the ones currently set with "+"
ConvertFrom-UACValue -Value <UAC_INT> -ShowAll
```

### PowerView — toggle the reversible-encryption bit (the downgrade)

`Set-ADObject` takes a SID / Name / `SamAccountName` target, a `-PropertyName` to manipulate, and either a `-PropertyValue` (absolute) or `-PropertyXorValue` (bit toggle). XOR is used so only the intended bit changes:

```powershell
# Set ENCRYPTED_TEXT_PWD_ALLOWED (reversible encryption) by XOR-ing bit 128
Set-ADObject -SamAccountName <SAM> -PropertyName useraccountcontrol -PropertyXorValue 128

# (optional) Flip DONT_EXPIRE_PASSWORD by XOR-ing bit 65536
Set-ADObject -SamAccountName <SAM> -PropertyName useraccountcontrol -PropertyXorValue 65536
```

### PowerView — force the password change

```powershell
# Setting pwdLastSet to 0 forces the user to change their password at next logon
Set-ADObject -SamAccountName <SAM> -PropertyName pwdlastset -PropertyValue 0
```

### PowerView — one-shot wrapper

`Invoke-DowngradeAccount` wraps the full workflow (flip the reversible-encryption flag and force the password change) into a single call against a target account:

```powershell
Invoke-DowngradeAccount -SamAccountName <SAM>
```

### Recover the cleartext — DCSync

Once `pwdLastSet` has changed (i.e. the user actually reset their password while the flag was set), pull the now-reversible secret via [[DCSync]]:

```text
# mimikatz — replicate the target and dump the (now reversible/cleartext) secret
lsadump::dcsync /domain:<DOMAIN.FQDN> /user:<SAM>
```
> [!caution] [UNVERIFIED]
> The exact `lsadump::dcsync` syntax above is the standard mimikatz invocation and is *not quoted verbatim* in the source post (the post says only that the password can be DCSynced once `pwdlastset` changes). Tagged `status/unverified`.

---

## [UNVERIFIED] Host-side WDigest LSASS registry downgrade (different technique)

> [!warning] Not in the source post — `status/unverified`
> This is the technique most people mean by "WDigest downgrade," but it is **not** described in the harmj0y post sourced here. Included for disambiguation only; verify against a primary source before relying on it.

On the local host, enabling WDigest credential caching causes LSASS to retain **cleartext** logon credentials in memory (recoverable with mimikatz `sekurlsa::wdigest`). The toggle is the registry value:

```text
HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest
    UseLogonCredential  (DWORD)  = 1
```

[UNVERIFIED] Setting `UseLogonCredential = 1` re-enables plaintext WDigest caching that KB2871997 disabled by default on modern Windows; the attacker then waits for an interactive re-logon and dumps cleartext from LSASS. This requires **local admin** on the host and does **not** involve AD object modification or DCSync. See [[LSASS Dumping]] and [[Credential Storage in Windows]]. Tagged `status/unverified`.

---

## Detection / artifacts

> [!note] The source post contains **no** detection guidance — the items below are general, defender-side correlations and are marked `status/unverified` where not anchored to the post.

- **`userAccountControl` change events** [UNVERIFIED]: a user object's UAC being modified to include `ENCRYPTED_TEXT_PWD_ALLOWED` (Directory Service / 4738 "user account changed" events) is highly anomalous for normal operations — alert on any account gaining reversible encryption.
- **`pwdLastSet` reset to 0 / forced password change** [UNVERIFIED] on a privileged or service account, especially correlated with a preceding UAC modification.
- **DCSync / replication from a non-DC source** [UNVERIFIED]: directory replication requests (DRSR `GetNCChanges`) originating from a host that is not a Domain Controller — same signal used to catch [[DCSync]].
- **ACL modifications** that granted the write primitive in the first place (see [[ACL Abuse]]).

---

## Mitigation

> [!note] The source post does not discuss mitigations; the following are standard defensive measures and are tagged `status/unverified` where not in the post.

- [UNVERIFIED] **Audit for accounts with reversible encryption enabled** and remediate them; treat any account with `ENCRYPTED_TEXT_PWD_ALLOWED` set as a finding. Enforce via the "Store passwords using reversible encryption" policy being **Disabled** and monitor for per-object overrides.
- [UNVERIFIED] **Lock down DACLs** on privileged user objects so non-admins cannot write `userAccountControl` / `pwdLastSet` ([[ACL Abuse]] hygiene).
- [UNVERIFIED] **Restrict and monitor replication rights** (`DS-Replication-Get-Changes-All`) to prevent the DCSync recovery step ([[DCSync]] mitigations).
- [UNVERIFIED] For the separate host-side WDigest variant: keep `UseLogonCredential` unset/`0`, deploy **Credential Guard**, and treat write access to the WDigest registry key as privileged ([[LSASS Dumping]]).

---

## Related

- [[DCSync]] — the replication path used to extract the now-reversible cleartext password.
- [[UserAccountControl]] — the attribute and `ENCRYPTED_TEXT_PWD_ALLOWED` (128) / `DONT_EXPIRE_PASSWORD` (65536) bits being toggled.
- [[PowerView]] — provides `Set-ADObject`, `ConvertFrom-UACValue`, and `Invoke-DowngradeAccount`.
- [[ACL Abuse]] — how the write primitive over the target object is usually obtained.
- [[Credential Storage in Windows]] / [[LSASS Dumping]] — context for the host-side WDigest plaintext-caching variant.

---

## Sources

- harmj0y — *Targeted Plaintext Downgrades with PowerView* — https://blog.harmj0y.net/redteaming/targeted-plaintext-downgrades-with-powerview/
