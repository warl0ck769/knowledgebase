---
title: AD DNS
aliases: ["AD DNS", "ADIDNS", "DNS zones", "DNS dynamic updates", "DomainDnsZones"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/dns]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Database]]", "[[Domain Controller Discovery]]", "[[DNS Attacks]]", "[[ADIDNS Spoofing]]", "[[LLMNR-NBT-NS Poisoning]]"]
created: 2026-06-06
updated: 2026-06-06
---

# AD DNS

> [!summary] One-liner
> DNS (port 53) is the backbone of AD name resolution and service location; in AD it's often integrated into the directory itself (ADIDNS).

## What it is
DNS resolves hostnames → IPs and, crucially in AD, **locates services via SRV records** (DCs, Kerberos, LDAP). DCs are usually the DNS servers.

- **Zones** = authoritative sections of the namespace, each managed by a nameserver.
- DCs are found via SRV records, e.g. `_ldap._tcp.dc._msdcs.contoso.local` (see [[Domain Controller Discovery]]).

## ADIDNS (AD-Integrated DNS)
DNS records can be **stored inside Active Directory** (in the **DomainDnsZones** / **ForestDnsZones** partitions of the [[AD Database]]). Consequence:
> [!danger]
> With **domain credentials**, a user can often **create arbitrary DNS records** via **DNS dynamic updates** — enabling record spoofing, traffic redirection, and NTLM capture without touching the DNS server config. → [[ADIDNS Spoofing]].

## DNS dynamic updates
A protocol letting clients register their own records. If insecure/misconfigured, attackers add fraudulent records (wildcard `*`, WPAD, etc.).

## Why a red teamer cares
- Service location + SRV = recon goldmine.
- ADIDNS dynamic updates = an **authenticated** path to poison name resolution domain-wide.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "DNS" (basics, zones, ADIDNS, dynamic updates)
