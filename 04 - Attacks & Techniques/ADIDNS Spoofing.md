---
title: ADIDNS Spoofing
aliases: ["ADIDNS", "ADIDNS spoofing", "DNS dynamic update attack", "wildcard ADIDNS"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, collection]
tags: [type/technique, domain/red-team, attack/credential-access, proto/dns]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD DNS]]", "[[LLMNR-NBT-NS Poisoning]]", "[[NTLM Relay]]", "[[NetNTLM]]"]
created: 2026-06-06
updated: 2026-06-06
---

# ADIDNS Spoofing

> [!summary] One-liner
> As an authenticated user, create/modify DNS records in AD-integrated DNS (via dynamic updates) to redirect traffic and capture authentication — works even where LLMNR is disabled.

## Concept abused
[[AD DNS|AD-integrated DNS]] lets clients register records via **DNS dynamic updates**, and **any authenticated user can usually add records**. So you can add a record (or a **wildcard `*`**) pointing victims to you — a stealthier, network-wide alternative to broadcast poisoning.

## Prerequisites
- Valid domain credentials. Default ADIDNS ACLs typically allow authenticated users to **create** records.

## How the attack works
1. Add a DNS record (e.g. `wpad` or a missing hostname, or a `*` wildcard) → your IP.
2. Victims resolving that name connect to you and authenticate → capture **[[NetNTLM]]**, then crack or [[NTLM Relay|relay]].

## Commands & tools
```powershell
# Powermad / Invoke-DNSUpdate (PowerShell)
Import-Module .\Powermad.ps1
Invoke-DNSUpdate -DNSType A -DNSName wpad -DNSData 10.10.10.10
```
```bash
# Linux: dnstool / krbrelayx project
python dnstool.py -u 'contoso\user' -p pass --record wpad --action add --data 10.10.10.10 DC.contoso.local
```
Then capture/relay with [[Responder]] / `ntlmrelayx.py`.

## Detection / artifacts
New/unexpected DNS records (esp. `wpad`, wildcards) created by user accounts; DNS dynamic-update events.

## Mitigation
Pre-create a `wpad` record (block-listed), restrict ADIDNS record creation/ACLs, disable insecure dynamic updates, SMB/LDAP signing.

## Related
- Same payoff as [[LLMNR-NBT-NS Poisoning]] but authenticated + domain-wide; pairs with [[NTLM Relay]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ADIDNS" / "DNS dynamic updates"
