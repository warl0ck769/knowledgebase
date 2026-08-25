---
title: AS-REP Roasting
aliases: ["ASREProast", "AS-REP Roast"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/kerberos]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[UserAccountControl]]", "[[Kerberos Authentication Flow]]", "[[Kerberoasting]]", "[[LDAP Enumeration]]", "[[ACL Abuse]]", "[[PowerView]]", "[[Rubeus]]"]
created: 2026-06-06
updated: 2026-06-07
---

# AS-REP Roasting

> [!summary] One-liner
> For accounts that don't require Kerberos pre-auth, request an AS-REP and crack the user-key-encrypted part offline — no credentials needed.

## Concept abused
Normally **pre-authentication** (a timestamp encrypted with the user's key) protects the AS exchange. If an account has **`DONT_REQUIRE_PREAUTH`** set ([[UserAccountControl]]), the AS will return an **AS-REP encrypted with the user's key to anyone** — crackable offline.

More precisely (harmj0y): in normal Windows Kerberos a user requesting a TGT must supply a timestamp encrypted with their password (`PA-ENC-TIMESTAMP`) inside the AS-REQ. The KDC decrypts this to verify identity *before* returning an AS-REP. When `DONT_REQ_PREAUTH` (userAccountControl value **4194304**) is set, that check is skipped, so an attacker can send an AS-REQ for the user and receive crackable encrypted material back.

## Prerequisites
- An account with `DONT_REQUIRE_PREAUTH`. Discovery often needs **no credentials** (you only need a valid username list), or use [[LDAP Enumeration]] if authenticated.
- The setting must be **explicitly set** on the target account for it to be vulnerable; the attack still relies on **weak password complexity** to succeed (strong crypto is not bypassed). harmj0y notes the flag tends to linger "on older accounts, specifically Unix-related ones."

## Commands & tools
```bash
# Impacket - unauthenticated if you have usernames
GetNPUsers.py 'contoso.local/' -usersfile users.txt -dc-ip 192.168.0.2 -no-pass -format hashcat
```
```powershell
# Rubeus (authenticated, auto-finds vulnerable accounts)
Rubeus.exe asreproast /format:hashcat /outfile:hashes.txt
```
```powershell
# PowerView - enumerate accounts that don't require pre-auth
Get-DomainUser -PreauthNotRequired
```
```powershell
# ASREPRoast toolkit (harmj0y) - full attack from a domain-auth context
Invoke-ASREPRoast            # enumerates DONT_REQ_PREAUTH users + grabs hashes
Get-ASREPHash -UserName <USER> -Domain <DOMAIN>   # hash for a single user
```
```bash
hashcat -m 18200 hashes.txt rockyou.txt -r best64.rule
```

## More from harmj0y (ASREPRoast toolkit & internals)
harmj0y's original "Roasting AS-REPs" post introduced the **ASREPRoast** toolkit and these functions:

- **`New-ASReq`** — builds a properly ASN.1-encoded AS-REQ by hand. It lets the attacker request **ARCFOUR-HMAC-MD5 (RC4)** encryption instead of the default AES256-CTS-HMAC-SHA1-96, which significantly speeds up offline cracking.
- **`Get-ASREPHash`** — wraps `New-ASReq` to: generate the AS-REQ for a specific user/domain, locate the DC, send the request and read the response bytes, decode the response with Bouncy Castle, extract the `enc-part` (RC4-HMAC encrypted), and return a crackable hash.
- **`Invoke-ASREPRoast`** — orchestrates the full attack from a domain-authenticated (but unprivileged) context: enumerates all `DONT_REQ_PREAUTH` users via the LDAP filter `(userAccountControl:1.2.840.113556.1.4.803:=4194304)`, calls `Get-ASREPHash` for each, and returns the set of crackable hashes.

**Message-type nuance for cracking:** the AS-REP encrypted part uses the same algorithm as a TGS-REP but is **Kerberos message type 8** (vs **type 2** for TGS-REP / [[Kerberoasting]]). This is why a dedicated cracker is needed:
- **John the Ripper** required a modified plugin `krb5_asrep_fmt_plug.c` — derived from the TGS-REP cracker but changed to message type 8 with the TGS-specific ASN.1 optimizations removed. Cracking performance matches TGS-REP.
- **Hashcat** — at publication, harmj0y noted the existing TGS-REP format could "simply" be modified the same way but no implementation existed yet. (Today hashcat mode **18200** covers AS-REP.)

### Privesc: setting DONT_REQ_PREAUTH via ACL rights
> [!important] Set-then-reset abuse
> If you hold **`GenericWrite`/`GenericAll`** rights over a target user ([[ACL Abuse]]), you can maliciously flip their `userAccountControl` to add `DONT_REQ_PREAUTH`, run ASREPRoast against them, then **reset the value** back. This makes accounts that are *not* normally vulnerable roastable on demand.

## Detection / artifacts
DC event **4768** (AS-REQ) with **pre-auth not required**; AS-REQ for accounts flagged DONT_REQUIRE_PREAUTH.
Also (harmj0y): alert on **abnormal AS-REP requests** for accounts that carry the flag, and regularly **audit which accounts have `DONT_REQ_PREAUTH`** (via PowerView `Get-DomainUser -PreauthNotRequired` or LDAP).

## Mitigation
Don't set `DONT_REQUIRE_PREAUTH`; strong passwords; Protected Users; monitor 4768.
Additional (harmj0y): remove the setting unless explicitly required for legacy/Unix compatibility; enforce **long, complex passwords** on any account that must keep it; investigate **Kerberos FAST pre-authentication (RFC 6113)** and deploy **PKINIT** for stronger initial auth.

## Related
- Sibling roast: [[Kerberoasting]] (mode 13100 vs 18200); enumerate via [[UserAccountControl]] bit `4194304`.
- [[ACL Abuse]] — `GenericWrite`/`GenericAll` enables the set-then-reset DONT_REQ_PREAUTH technique.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ASREProast"
- [[Source - harmj0y blog]] — "Roasting AS-REPs" — https://blog.harmj0y.net/activedirectory/roasting-as-reps/
