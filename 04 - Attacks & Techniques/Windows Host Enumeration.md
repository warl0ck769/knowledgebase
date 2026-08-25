---
title: Windows Host Enumeration
aliases: ["Windows computers discovery", "Computer enumeration"]
type: technique
domain: [red-team]
attack_tactic: [recon]
tags: [type/technique, domain/red-team, attack/recon]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Computer Accounts]]", "[[LDAP]]", "[[Domain Controller Discovery]]", "[[NetBIOS]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Windows Host Enumeration

> [!summary] One-liner
> Find Windows machines in the domain via LDAP computer objects, NetBIOS, or SMB scans.

## Concept abused
Every machine is a [[Computer Accounts|computer object]] in the directory; live hosts also answer NetBIOS (137) and SMB (445) probes that leak name/OS/domain.

## Prerequisites
- LDAP method needs domain creds; NetBIOS/SMB scans are unauthenticated.

## Commands & tools

### LDAP (authenticated)
```bash
ldapsearch -H ldap://192.168.100.2 -x -LLL -W -D "anakin@contoso.local" \
  -b "dc=contoso,dc=local" "(objectclass=computer)" DNSHostName OperatingSystem
```

### NetBIOS scan (port 137)
```bash
nbtscan 192.168.100.0/24
```

### SMB scan (port 445) — leaks name, OS, domain via NTLM
```bash
ntlm-info smb 192.168.100.0/24
```

## Detection / artifacts
Broad LDAP queries and network sweeps are detectable; LDAP query is logged on the DC.

## Mitigation
Network segmentation; SMB signing/NTLM hardening reduces info leakage.

## Related
- [[Computer Accounts]] · [[Domain Controller Discovery]] · [[LDAP Enumeration]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Windows computers discovery"
