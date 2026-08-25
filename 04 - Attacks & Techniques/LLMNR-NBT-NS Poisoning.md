---
title: LLMNR-NBT-NS Poisoning
aliases: ["LLMNR poisoning", "NBT-NS poisoning", "mDNS poisoning", "Responder attack"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[NetBIOS]]", "[[NetNTLM]]", "[[NTLM Relay]]", "[[NTLM Authentication]]", "[[WPAD and IPv6 (mitm6) Poisoning]]"]
created: 2026-06-06
updated: 2026-06-06
---

# LLMNR-NBT-NS Poisoning

> [!summary] One-liner
> Answer broadcast name-resolution requests (LLMNR/NBT-NS/mDNS) as the victim's intended host, making the victim authenticate to you — capture NetNTLM hashes to crack or relay.

## Concept abused
When DNS resolution fails, Windows falls back to **broadcast/multicast** name resolution that is **unauthenticated**:
- **LLMNR** (UDP **5355**)
- **NBT-NS** (UDP **137**, [[NetBIOS]] name service)
- **mDNS** (UDP **5353**)

An attacker on the LAN simply **replies "that name is me."** The victim then connects and authenticates, sending its **[[NetNTLM]]** response.

## Prerequisites
- Same broadcast/L2 segment as victims. No credentials needed.

## How the attack works
1. Victim mistypes/looks up a name with no DNS record → broadcasts an LLMNR/NBT-NS query.
2. Attacker responds with attacker IP.
3. Victim authenticates (e.g. SMB/HTTP) → sends **NetNTLMv1/v2**.
4. Attacker **cracks** it offline, or **[[NTLM Relay|relays]]** it live to another host.

## Commands & tools

### [[Responder]] (capture)
```bash
responder -I eth0 -wv        # poison LLMNR/NBT-NS/mDNS + serve rogue services (HTTP/SMB/WPAD)
# captured NetNTLMv2 -> hashcat -m 5600
```

### [[Inveigh]] (Windows equivalent)
```powershell
Invoke-Inveigh -NBNS Y -mDNS Y -LLMNR Y -ConsoleOutput Y
```

### Relay instead of crack (turn off Responder's SMB/HTTP first)
```bash
ntlmrelayx.py -tf targets.txt -smb2support   # see [[NTLM Relay]]
```

## Detection / artifacts
Injected LLMNR/NBT-NS responses; honeypot name lookups; spikes of NetNTLM auths to one host.

## Mitigation
**Disable LLMNR + NBT-NS + mDNS**; enable **SMB signing** (blocks relay); segment networks.

## Related
- Captures [[NetNTLM]] → feed [[NTLM Relay]] or crack. IPv6/WPAD variant: [[WPAD and IPv6 (mitm6) Poisoning]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "LLMNR", "mDNS", "NetBIOS Name Service"
