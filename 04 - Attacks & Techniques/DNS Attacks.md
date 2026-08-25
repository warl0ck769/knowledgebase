---
title: DNS Attacks
aliases: ["Fake DNS server", "DNS zone transfer", "AXFR", "DNS exfiltration", "Dump DNS records"]
type: technique
domain: [red-team]
attack_tactic: [recon, exfiltration, collection]
tags: [type/technique, domain/red-team, attack/recon, proto/dns]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD DNS]]", "[[ADIDNS Spoofing]]", "[[ARP Spoofing]]"]
created: 2026-06-06
updated: 2026-06-06
---

# DNS Attacks

> [!summary] One-liner
> Use DNS (53) offensively: enumerate records, abuse zone transfers, run a fake DNS server to redirect victims, or tunnel data out via DNS.

## Variants & how they work
| Variant | Mechanism | Tool |
|---|---|---|
| **Dump DNS records** | Enumerate hosts/services via standard queries (no full transfer) | `dnsrecon`, `dig`, `nslookup` |
| **DNS Zone Transfer (AXFR)** | Misconfigured server replicates the **entire zone** to anyone | `dig axfr` |
| **Fake DNS server** | Answer queries with attacker IPs → redirect victims to malicious hosts | **dnschef** |
| **DNS exfiltration** | Encode data in DNS queries/responses → covert channel past monitoring | `iodine`, `dnscat2` |

## Commands & tools
```bash
# Zone transfer attempt
dig axfr contoso.local @192.168.100.2
# or via dnsrecon
dnsrecon -d contoso.local -t axfr

# Record enumeration
dnsrecon -d contoso.local -n 192.168.100.2

# Fake DNS responses
dnschef --fakeip 10.10.10.10 --interface 0.0.0.0
```

## Detection / artifacts
AXFR requests from non-secondary hosts; anomalous high-volume/long DNS queries (exfil); clients resolving to wrong IPs.

## Mitigation
Restrict zone transfers to secondaries, DNS query monitoring/anomaly detection, DNSSEC, egress filtering.

## Related
- AD-specific record creation: [[ADIDNS Spoofing]]; MITM prerequisite: [[ARP Spoofing]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "DNS" (exfiltration, fake server, zone transfer, dump records)
