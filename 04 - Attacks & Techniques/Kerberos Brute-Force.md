---
title: Kerberos Brute-Force
aliases: ["Kerbrute", "Kerberos password spray", "Kerberos user enumeration"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, recon]
tags: [type/technique, domain/red-team, attack/credential-access, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos Authentication Flow]]", "[[KDC]]", "[[AS-REP Roasting]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Kerberos Brute-Force

> [!summary] One-liner
> Use the AS exchange to enumerate valid usernames and spray passwords against the KDC, often quieter than NTLM brute-force.

## Concept abused
The **AS-REQ** ([[Kerberos Authentication Flow]]) reveals whether a username exists (distinct KDC error for unknown user vs. bad pre-auth) and whether a password is correct (pre-auth succeeds), all without a full logon.

## Prerequisites
- Network access to the [[KDC]] (port 88). No prior credentials for user enumeration.

## Commands & tools
```bash
# Username enumeration
kerbrute userenum -d contoso.local --dc 192.168.0.2 users.txt

# Password spray (one password across many users)
kerbrute passwordspray -d contoso.local --dc 192.168.0.2 users.txt 'Spring2026!'
```

## Detection / artifacts
DC **4768** failures spike; many AS-REQs from one source. (Spraying still increments badPwdCount → can cause lockouts — pace it.)

## Mitigation
Account lockout + smart lockout, MFA, monitor 4768 anomalies, strong/unique passwords.

## Related
- Often paired with [[AS-REP Roasting]] (both use the AS exchange).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos brute-force"
