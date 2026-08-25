---
title: Silver Ticket
aliases: ["Silver Ticket"]
type: technique
domain: [red-team]
attack_tactic: [persistence, lateral-movement]
tags: [type/technique, domain/red-team, attack/persistence, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Service Principal Name (SPN)]]", "[[Computer Accounts]]", "[[PAC]]", "[[Golden Ticket]]", "[[SAM and LSA Secrets Dump]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Silver Ticket

> [!summary] One-liner
> Forge a service ticket for one specific service using that service account's key — stealthy because it never touches the DC.

## Concept abused
An **ST** is encrypted with the **service account's key** ([[TGT vs TGS]]). The target service validates that encryption and usually trusts the embedded [[PAC]] **without re-checking the KDC signature**. So with the service key you forge a valid ST directly.

### PAC double-signing detail
The [[PAC]] inside a service ticket is signed **twice**:
1. **Service account's secret** — the service verifies this signature.
2. **krbtgt secret** — intended for the KDC to verify if the service requests PAC validation.

In practice, services typically **do NOT verify the KDC's (krbtgt) signature**. This is precisely why Silver Tickets work despite only having the service key: the attacker forges the PAC's first signature using the service key, and the second signature (krbtgt) goes **unchecked** by the target service.

## Prerequisites
- The **service account's key/NT hash** (e.g. a [[Computer Accounts|machine account]]'s `$MACHINE.ACC` from [[SAM and LSA Secrets Dump]]) + the domain SID + target [[Service Principal Name (SPN)|SPN]].

## Commands & tools
```bash
# Impacket - forge ST for cifs/host
ticketer.py -nthash <service_hash> -domain-sid S-1-5-21-... -domain contoso.local \
  -spn cifs/server.contoso.local Administrator
```
```powershell
# mimikatz
kerberos::golden /user:Administrator /domain:contoso.local /sid:S-1-5-21-... \
  /target:server.contoso.local /service:cifs /rc4:<service_hash> /ptt
```

## Properties
- **Scoped to one service** on one host, but **stealthy** — no AS/TGS traffic to the DC (no 4768/4769).
- Survives krbtgt resets (depends on the *service* key instead).
- **Persistence**: Even if the krbtgt password is changed, Silver Tickets will still work, as long as the service's password doesn't change.

## Detection / artifacts
Service access with no corresponding TGS request on the DC; PAC anomalies if the service validates KDC signature (PAC validation hardening).

## Mitigation
Rotate machine/service account passwords, enable PAC validation, gMSA, tier separation.

## Related
- Domain-wide equivalent: [[Golden Ticket]]; machine-account leverage also enables [[Resource-Based Constrained Delegation (RBCD)]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Golden/Silver ticket"
- [[Source - hackndo blog]]
