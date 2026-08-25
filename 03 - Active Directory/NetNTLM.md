---
title: NetNTLM
aliases: ["NetNTLMv1", "NetNTLMv2", "NTLMv1 response", "NTLMv2 response", "Net-NTLM hash"]
type: concept
domain: [active-directory, windows-internals]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/security/kerberos/ntlm-overview"
verified: true
related: ["[[NTLM Authentication]]", "[[LM and NT Hashes]]", "[[NTLM Cracking]]", "[[NTLM Relay]]", "[[LLMNR-NBT-NS Poisoning]]"]
created: 2026-06-07
updated: 2026-06-07
---

# NetNTLM (Net-NTLM Hash)

> [!summary] One-liner
> The challenge-response value sent over the network during [[NTLM Authentication]] — it proves knowledge of the password without sending it, but can be cracked offline or relayed to another service.

## NetNTLM vs. NT hash — critical distinction

| Property | NT hash | NetNTLM (v1/v2) |
|---|---|---|
| **What it is** | `MD4(password)` — stored in SAM/NTDS.dit | `HMAC(NT_hash, challenge + blob)` — sent on the wire |
| **Can be used for Pass-the-Hash?** | **Yes** | **No** |
| **Can be relayed?** | N/A | **Yes** (to another service accepting NTLM) |
| **Can be cracked?** | Yes (hashcat -m 1000) | Yes (hashcat -m 5500 / 5600) |
| **Where you get it** | SAM dump, LSASS dump, DCSync, NTDS.dit | Responder, Inveigh, MITM, Wireshark |

## Versions

### NetNTLMv1
- Challenge-response: `DES(NT_hash, server_challenge)`.
- **Weak**: Can be downgraded and cracked rapidly or even converted directly to an NT hash via rainbow tables (crack.sh) if the challenge is controlled.
- hashcat mode: **5500**

### NetNTLMv2
- Challenge-response: `HMAC-MD5(NT_hash, server_challenge + client_challenge + timestamp + target_info)`.
- **Stronger**: Includes client challenge and timestamp → replay attacks are harder, cracking is slower.
- hashcat mode: **5600**
- Default on modern Windows (LMCompatibilityLevel ≥ 3).

## How they're captured
```
Attacker (Responder/Inveigh)          Victim
         ←── NTLM Negotiate ──────────
         ──── Challenge (controlled) ──→
         ←── Authenticate (NetNTLM) ───
         (now crack or relay)
```

Captured via: [[LLMNR-NBT-NS Poisoning]], [[WPAD and IPv6 (mitm6) Poisoning]], [[ADIDNS Spoofing]], or any MITM position that triggers NTLM auth.

## Cracking
```bash
# NetNTLMv1
hashcat -m 5500 netntlmv1.txt wordlist.txt

# NetNTLMv2
hashcat -m 5600 netntlmv2.txt wordlist.txt
```

## Relaying (instead of cracking)
If the captured NetNTLM response is relayed in real-time to another service (SMB, LDAP, HTTP), the attacker authenticates as the victim — see [[NTLM Relay]]. Relaying is more powerful than cracking because it works regardless of password complexity.

## Related
- Protocol: [[NTLM Authentication]] (the full auth flow).
- Underlying secret: [[LM and NT Hashes]].
- Cracking details: [[NTLM Cracking]].
- Relay: [[NTLM Relay]].
- Capture methods: [[LLMNR-NBT-NS Poisoning]], [[WPAD and IPv6 (mitm6) Poisoning]].
