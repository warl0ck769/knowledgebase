---
title: WPAD and IPv6 (mitm6) Poisoning
aliases: ["WPAD poisoning", "mitm6", "IPv6 DNS takeover"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LLMNR-NBT-NS Poisoning]]", "[[NTLM Relay]]", "[[NetNTLM]]", "[[AD DNS]]"]
created: 2026-06-06
updated: 2026-06-06
---

# WPAD and IPv6 (mitm6) Poisoning

> [!summary] One-liner
> Abuse auto-proxy discovery (WPAD) and default IPv6 to become the victim's DNS/proxy, forcing authenticated traffic you can capture or relay.

## Concept abused
- **WPAD** (Web Proxy Auto-Discovery): clients auto-find a proxy via **DHCP, DNS, or LLMNR**. Poison the WPAD answer → clients route HTTP through your proxy and authenticate (NTLM).
- **IPv6 is enabled and preferred by default** on Windows but usually unmanaged. An attacker can run a **rogue DHCPv6** server, hand out **the attacker as the IPv6 DNS server**, then answer DNS (incl. WPAD) → man-in-the-middle.

## Prerequisites
- LAN access. No credentials needed.

## Commands & tools

### mitm6 (IPv6 DNS takeover) + relay
```bash
mitm6 -d contoso.local                       # become victims' IPv6 DNS server
ntlmrelayx.py -6 -t ldaps://DC -wh attacker-wpad --delegate-access
# relays captured auth to LDAP/SMB -> see [[NTLM Relay]]
```

### [[Responder]] WPAD
```bash
responder -I eth0 -wv     # also serves a rogue WPAD/proxy + auth prompt
```

## Detection / artifacts
Unexpected DHCPv6 advertisements, rogue IPv6 DNS, WPAD lookups resolving to a workstation, bursts of NTLM auth.

## Mitigation
Disable WPAD; disable IPv6 if unused (or RA-Guard/DHCPv6 controls); SMB/LDAP signing + channel binding to stop relay; Protected Users for admins.

## Related
- Same payoff as [[LLMNR-NBT-NS Poisoning]] (capture [[NetNTLM]]) but via WPAD/IPv6; usually paired with [[NTLM Relay]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "WPAD"
