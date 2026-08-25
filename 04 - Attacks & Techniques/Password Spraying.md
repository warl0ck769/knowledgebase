---
title: Password Spraying
aliases: ["password spray", "spray and pray", "password spraying lockout"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source: "[[Source - hackndo blog]]"
source_url: "https://en.hackndo.com/password-spraying-lockout/"
verified: true
related: ["[[Kerberos Brute-Force]]", "[[NTLM Authentication]]", "[[Kerberos]]", "[[LDAP]]", "[[CrackMapExec / NetExec]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Password Spraying

> [!summary] One-liner
> Test one common password against many domain accounts simultaneously — avoids lockouts by staying under the threshold, and only needs one weak password to succeed.

## Concept abused
Most organizations set an account lockout threshold (e.g., 5 bad attempts). Password spraying inverts the brute-force approach: instead of many passwords against one account, it tests **one password against all accounts**, then waits for the lockout counter to reset before trying the next password.

## Lockout policy mechanics

### Three critical parameters

| Parameter | Meaning | Default |
|---|---|---|
| **Account Lockout Threshold** | Bad attempts before lockout | 0 (no lockout) or 5 |
| **Account Lockout Duration** | How long the account stays locked | 30 min |
| **Reset Account Lockout Counter After** (observation window) | Time after which `badPwdCount` resets to 0 if no new failures | 30 min |

### How `badPwdCount` works
1. User fails authentication → DC checks if last failure is older than the observation window.
2. If yes → `badPwdCount` resets to 0, then increments to 1.
3. If no → `badPwdCount` increments.
4. If `badPwdCount` reaches the threshold → `lockoutTime` is set, account is locked.
5. Successful authentication → `badPwdCount` resets to 0.

### Safe spraying formula
```
Wait time between sprays = Observation Window + safety margin
Attempts per spray = Lockout Threshold - 1 (leave margin)
```
Example: Threshold=5, Window=30min → spray 1 password, wait 35 minutes, spray next.

## Fine-Grained Password Policies (PSO) complication
**Password Settings Objects (PSOs)** can override the domain's default lockout policy on a per-user or per-group basis:
- A PSO with a lower lockout threshold (e.g., 3 instead of 5) on privileged accounts will lock them faster than expected.
- PSOs have priority over domain policy (lowest `Precedence` value wins).
- Stored in `CN=Password Settings Container,CN=System,DC=...`.

> [!warning] PSOs can have stricter thresholds
> The `msDS-ResultantPSO` attribute on a user object reveals which PSO applies — and **is readable by any authenticated domain user**. Check it before spraying to avoid locking accounts with stricter policies.

```powershell
# Check if a user has a PSO applied
Get-ADUser <USER> -Properties msDS-ResultantPSO | Select msDS-ResultantPSO

# Get the PSO's lockout threshold
Get-ADFineGrainedPasswordPolicy -Identity "<PSO_DN>" | Select LockoutThreshold, LockoutObservationWindow
```

## Commands & tools

### CrackMapExec / NetExec
```bash
# Spray one password against all users
nxc smb <DC_IP> -u users.txt -p 'Winter2026!' -d <DOMAIN> --continue-on-success

# Spray with hash
nxc smb <DC_IP> -u users.txt -H <NT_HASH> -d <DOMAIN>
```

### Kerbrute (Kerberos-based — no lockout logging on older DCs)
```bash
kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> users.txt 'Winter2026!'
```

### DomainPasswordSpray (PowerShell)
```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password 'Winter2026!' -Domain <DOMAIN>
```

### Conpass (lockout-aware sprayer)
```bash
# Retrieves DC time, respects observation windows, excludes PSO-protected users
conpass spray -d <DOMAIN> -u users.txt -p passwords.txt --dc <DC_IP>
```

### Check lockout policy first
```powershell
# Domain policy
Get-ADDefaultDomainPasswordPolicy | Select LockoutThreshold, LockoutObservationWindow, LockoutDuration

# Or net accounts
net accounts /domain
```

## Common spray passwords
Build candidates from: `Season+Year` (Winter2026), `Company+123`, `Welcome1`, `Password1`, keyboard patterns (`Qwerty123!`), local sports teams, recently expired passwords.

## Detection / artifacts
- **Event 4625** (failed logon) from a single source IP against many accounts in a short window.
- **Event 4771** (Kerberos pre-auth failure) for Kerberos-based spraying.
- High `badPwdCount` across many accounts simultaneously.
- Correlation: same password attempted across all accounts (if logging captures it).

## Mitigation
- Set a sane lockout policy (threshold ≥ 5, observation window ≥ 30 min).
- **Fine-grained policies (PSOs)** for privileged accounts with stricter settings.
- Enforce strong passwords / passphrases — spray only works against weak passwords.
- MFA for remote access (VPN, OWA, RDWeb).
- Monitor for distributed failed logon patterns.
- **Azure AD Smart Lockout** / on-prem extranet lockout for hybrid environments.

## Related
- Kerberos-based user enumeration: [[Kerberos Brute-Force]].
- Captured credentials: [[NetNTLM]], [[NTLM Cracking]].
- Spray tool: [[CrackMapExec / NetExec]].

## Sources
- [[Source - hackndo blog]] — "Spray passwords, avoid lockouts" — https://en.hackndo.com/password-spraying-lockout/
