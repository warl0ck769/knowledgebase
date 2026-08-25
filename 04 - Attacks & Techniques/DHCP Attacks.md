---
title: DHCP Attacks
aliases: ["Rogue DHCP", "DHCP starvation", "DHCP Dynamic DNS"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, collection]
tags: [type/technique, domain/red-team, attack/credential-access]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[WPAD and IPv6 (mitm6) Poisoning]]", "[[LLMNR-NBT-NS Poisoning]]", "[[AD DNS]]"]
created: 2026-06-06
updated: 2026-06-06
---

# DHCP Attacks

> [!summary] One-liner
> Abuse DHCP (UDP 67/68) to feed clients attacker-controlled config (DNS/gateway/WPAD), or starve the pool to force fallback to poisonable name resolution.

## Concept abused
DHCP hands clients their IP, gateway, **DNS server**, and options (incl. WPAD) — and clients trust the first/any offer.

## Variants
| Attack | What it does |
|---|---|
| **Rogue DHCP server** | Offer config that sets attacker as gateway/DNS → MITM, credential capture |
| **DHCP starvation** | Flood `DHCPDISCOVER` to exhaust the pool → clients fall back to **LLMNR/mDNS** ([[LLMNR-NBT-NS Poisoning]]) |
| **DHCP discovery** | Locate DHCP servers (`DHCPDISCOVER`/`DHCPOFFER`) |
| **DHCP Dynamic DNS** | DHCP auto-registers client names in DNS → tie-in to [[AD DNS]] record manipulation |

## Commands & tools
```bash
# DHCP starvation + rogue server
dhcpstarv -i eth0
yersinia -G                     # DHCP attack module (rogue/starvation)
# IPv6 equivalent (preferred on modern Windows): mitm6 -> see linked note
```

## Detection / artifacts
Multiple DHCP servers on a segment; rapid lease exhaustion; unexpected DNS/gateway in client config.

## Mitigation
**DHCP snooping** on switches, authorized DHCP servers only, monitor lease usage.

## Related
- Most impactful modern variant is IPv6: [[WPAD and IPv6 (mitm6) Poisoning]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "DHCP" (rogue / starvation / discovery / dynamic DNS)
