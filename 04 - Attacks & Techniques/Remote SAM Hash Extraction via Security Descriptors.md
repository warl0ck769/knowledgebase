---
title: Remote SAM Hash Extraction via Security Descriptors
aliases: ["DAMP", "Discretionary ACL Modification Project", "Registry Security Descriptor Backdoor", "Remote Hash Retrieval"]
type: technique
domain: [active-directory, windows-internals, red-team]
attack_tactic: [persistence, credential-access, defense-evasion]
tags: [type/technique, domain/red-team, domain/windows-internals, attack/persistence, attack/credential-access, attack/defense-evasion, proto/winreg]
source: "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/activedirectory/remote-hash-extraction-on-demand-via-host-security-descriptor-modification/"
verified: true
related: ["[[SAM Database]]", "[[LSA Secrets]]", "[[SAM and LSA Secrets Dump]]", "[[Silver Ticket]]", "[[Computer Accounts]]", "[[Pass-the-Hash]]", "[[ACL, ACE, DACL, SACL]]", "[[ACL Abuse]]", "[[GPO Abuse]]", "[[Credential Storage in Windows]]", "[[NTDS.dit Extraction]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Remote SAM Hash Extraction via Security Descriptors

> [!summary] One-liner
> On a host you already own as admin, modify the DACLs on the SAM/SECURITY/SYSTEM registry keys so a chosen trustee can remotely read the encrypted hive material over the remote-registry (winreg) RPC pipe — turning credential extraction into an on-demand, malware-free persistence backdoor.

## Concept abused
Windows stores secret material under registry keys whose **security descriptors** ([[ACL, ACE, DACL, SACL|DACLs]]) normally restrict read access to SYSTEM/Administrators. Local hashes live in **SAM**, LSA secrets and cached domain creds live in **SECURITY**, and the **SysKey/bootkey** is derived from **SYSTEM** keys ([[SAM Database]], [[LSA Secrets]], [[Credential Storage in Windows]]). Because access is gated purely by DACLs, an attacker who is already admin can **add an explicit allow ACE for an arbitrary trustee** to those keys. That trustee can then read the (still-encrypted) blobs **remotely via the Remote Registry / winreg RPC service** and decrypt them offline — never needing to re-enter the Administrators group, drop a binary, or touch LSASS again. This is the **DAMP (Discretionary ACL Modification Project)** technique.

> [!note] Not a vulnerability
> harmj0y explicitly frames this as **post-exploitation persistence**, not an exploit: *"This is not a vulnerability! Rather it's a useful post-exploitation approach to ensure continued access on a machine that you've already completely compromised."* It abuses the legitimate Windows access-control model.

## Prerequisites
- **Existing local administrative access** on the target (one-time, to set the ACEs).
- **WMI connectivity** to the target to push the security-descriptor changes (DAMP uses WMI's `StdRegProv`).
- The **RemoteRegistry** service reachable/startable on the target (default-on for Windows Server; on by default-or-startable to read the keys later).
- Knowledge of the **trustee** to grant: a `DOMAIN\user`, a well-known name (`Everyone`), or a SID string (`S-1-1-0`).

## How the attack works
1. **Backdoor (one-time, as admin):** Add an allow ACE granting your chosen trustee full access to the sensitive keys. DAMP writes ACEs with:
   - `AccessMask = 983103` (KEY_ALL_ACCESS)
   - `AceFlags = 0x2` (CONTAINER_INHERIT_ACE)
   - `AceType = 0x0` (ACCESS_ALLOWED)
2. **Key set that gets re-permissioned:**
   - **SysKey / bootkey (SYSTEM hive):** `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\JD`, `...\Skew1`, `...\Data`, `...\GBG`
   - **Remote-registry gate:** `HKLM\SYSTEM\CurrentControlSet\Control\SecurePipeServers\winreg` (so the trustee may use the remote-registry pipe at all)
   - **LSA key + machine account (SECURITY hive):** `HKLM\SECURITY\Policy\PolEKList`, `HKLM\SECURITY\Policy\Secrets\$MACHINE.ACC\CurrVal`
   - **Local accounts (SAM hive):** `HKLM\SAM\SAM\Domains\Account\Users\<RID>`
   - **Domain cached creds (SECURITY hive):** `HKLM\SECURITY\Policy\Secrets\NL$KM\CurrVal` and `HKLM\SECURITY\Cache\NL$<1-10>`
3. **On-demand retrieval (later, as the trustee — no local admin needed):** Connect to the target's remote registry, read the encrypted blobs, derive the bootkey from the four `Lsa` subkeys, decrypt the LSA key from `PolEKList`, then decrypt:
   - **Local SAM hashes** (RC4/DES over the per-user SAM `<RID>` data keyed by bootkey + the SAM `F` value),
   - the **machine account hash** from `$MACHINE.ACC\CurrVal`, and/or
   - **MsCacheV2** cached domain hashes from `NL$KM` + `NL$<1-10>`.
4. **Use the loot:** The machine account hash forges [[Silver Ticket|Silver Tickets]] for that host's services indefinitely; local-admin and cached hashes enable [[Pass-the-Hash]] / lateral movement. Because retrieval is just registry reads by an authorized principal, it can be repeated **on demand, indefinitely**.

## Commands & tools
DAMP (PowerShell, `HarmJ0y/DAMP`):
```powershell
# 1) Plant the backdoor: grant a trustee remote read on all the hash-relevant keys
Add-RemoteRegBackdoor -ComputerName <target> -Trustee "DOMAIN\<user>"
# Trustee can also be a well-known name or SID, e.g. -Trustee "Everyone" / -Trustee "S-1-1-0"

# 2) Later, as that trustee (no Administrators membership required), retrieve hashes:
Get-RemoteMachineAccountHash -ComputerName <target>   # -> machine ($MACHINE.ACC) NTLM hash for Silver Tickets
Get-RemoteLocalAccountHash   -ComputerName <target>   # -> local SAM account NTLM hashes
Get-RemoteCachedCredential   -ComputerName <target>   # -> MsCacheV2 cached domain credentials
```

Scaling via [[GPO Abuse|Group Policy]] (re-permission every affected host on policy refresh):
```text
# Computer Configuration > Policies > Windows Settings > Security Settings > Registry
# (writes ACL entries into the GPO template:)
<GPOPATH>\Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf
```

## Detection / artifacts
- **Anomalous ACEs** on the SAM, SECURITY, and SYSTEM\...\Lsa keys, and on `...\SecurePipeServers\winreg`, granting non-standard principals (e.g. a normal user, `Everyone`, `S-1-1-0`).
- **RemoteRegistry** service enabled/started where it normally isn't.
- **Remote-registry reads** of `SAM`/`SECURITY`/`Lsa` keys from unexpected principals (winreg RPC + registry-access auditing / SACLs on these keys).
- **GPO modifications** touching `GptTmpl.inf` registry security settings.

## Mitigation
- **Baseline and audit the DACLs** of SAM/SECURITY/SYSTEM Lsa keys and the `winreg` key; alert on additions of non-standard trustees.
- **Disable / restrict the RemoteRegistry service**; lock down who may connect to the `SecurePipeServers\winreg` pipe.
- **Rotate machine and local account passwords** after suspected compromise (invalidates harvested hashes and forged [[Silver Ticket|Silver Tickets]]).
- Add **SACLs / registry auditing** on these keys to surface remote reads.
- Tier separation and reducing local-admin sprawl limit who can plant the backdoor in the first place.

## Related
- Decrypts the same material covered by [[SAM and LSA Secrets Dump]] and [[SAM Database]] / [[LSA Secrets]], but **remotely and persistently** instead of one-shot local dumping.
- Machine-account hash output feeds [[Silver Ticket]]; local/cached hashes feed [[Pass-the-Hash]].
- DACL-based persistence is conceptually adjacent to [[ACL Abuse]]; mass-deployment uses [[GPO Abuse]].
- For full directory-database extraction instead of host-local secrets, see [[NTDS.dit Extraction]] / [[DCSync]].

## Sources
- [[Source - harmj0y blog]] — "Remote Hash Extraction On Demand Via Host Security Descriptor Modification" — https://blog.harmj0y.net/activedirectory/remote-hash-extraction-on-demand-via-host-security-descriptor-modification/
