---
title: Domain Controller Discovery
aliases: ["DC discovery", "Find domain controllers"]
type: technique
domain: [red-team]
attack_tactic: [recon]
tags: [type/technique, domain/red-team, attack/recon]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain Controller (DC)]]", "[[AD DNS]]", "[[Windows Host Enumeration]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Domain Controller Discovery

> [!summary] One-liner
> Locate the domain's DCs via DNS SRV records, the nltest utility, or port scanning.

## Concept abused
DCs advertise themselves via **DNS SRV records** and respond on a fixed [[Domain Controller (DC)|port set]].

## Prerequisites
- Network access. The DNS-SRV method needs **no credentials**; `nltest /dclist` needs a domain user.

## Commands & tools

### DNS SRV (unauthenticated)
```bash
# LDAP SRV record for DCs = the domain controllers
nslookup -q=srv _ldap._tcp.dc._msdcs.contoso.local
```

### nltest (needs domain creds)
```cmd
:: DC list with site info
nltest /dclist:contoso.local
```

### Port scan
Look for the DC signature ports: **88 (Kerberos)**, **389 (LDAP)**, 445, 135, 53, 636, 3268/3269, 5985, 9389.
```bash
nmap -p 53,88,135,139,389,445,464,636,3268,3269,5985,9389 192.168.100.0/24
```

## Detection / artifacts
DNS SRV queries are normal client behavior (low signal); broad port scans are noisy/IDS-visible.

## Mitigation
Limited — DC discovery is part of normal AD operation. Monitor for anomalous scanning.

## Related
- [[Domain Controller (DC)]] · [[AD DNS]] · follow with [[Windows Host Enumeration]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domain Controllers discovery"
