---
title: Inter-realm TGT
aliases: ["Inter-realm TGT", "Inter-Realm ticket", "Trust ticket"]
type: technique
domain: [red-team]
attack_tactic: [lateral-movement, privesc]
tags: [type/technique, domain/red-team, attack/lateral-movement, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain and Forest Trusts]]", "[[Trust Accounts]]", "[[TGT vs TGS]]", "[[SID History Abuse]]", "[[Golden Ticket]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Inter-realm TGT

> [!summary] One-liner
> Forge the referral (inter-realm) TGT using a stolen trust key to move from one domain/realm into a trusted one.

## Concept abused
When a user accesses a resource in a trusted domain, their KDC issues an **inter-realm TGT** (`krbtgt/TARGET_DOMAIN@SOURCE_DOMAIN`) **encrypted with the trust key** shared between the two domains ([[Trust Accounts]]). Steal the trust key → forge this referral ticket.

## Prerequisites
- The **trust key** (NT hash / Kerberos key of the `TARGETDOMAIN$` trust account), plus a [[Domain and Forest Trusts|trust]] between the domains.

## How the attack works
1. Obtain the trust key (DCSync the trust account, or dump it).
2. Forge an inter-realm TGT for `krbtgt/TARGET@SOURCE` with chosen PAC/SIDs.
3. Present it to the target domain's TGS to obtain STs there.

## Commands & tools
```powershell
# Rubeus - use the forged inter-realm ticket to ask for service in target domain
Rubeus.exe asktgs /ticket:<interrealm.kirbi> /service:cifs/dc.target.local /dc:dc.target.local /ptt
```
```bash
# Impacket - forge with trust key (often combined with -extra-sid for forest escalation)
ticketer.py -nthash <trustkey> -domain-sid <sourceSID> -domain source.local \
  -spn krbtgt/target.local -extra-sid <targetSID>-519 Administrator
```

## Detection / artifacts
Cross-realm TGS requests with anomalous PAC/SIDs; tickets encrypted with the trust key in unusual patterns.

## Mitigation
**SID filtering** on trusts, rotate/protect trust keys, monitor cross-realm referrals.

## Related
- Frequently combined with [[SID History Abuse]] (extra SIDs) to escalate child → forest root; conceptually a cross-trust [[Golden Ticket]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Inter-realm TGT"
