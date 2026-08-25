---
title: Authentication Packages
aliases: ["authentication package", "AP", "MSV1_0", "Kerberos AP", "Negotiate"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/secauthn/authentication-packages"
verified: true
related: ["[[SSPI and SSPs]]", "[[LSASS]]", "[[NTLM Authentication]]", "[[Kerberos]]", "[[Credential Storage in Windows]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Authentication Packages

> [!summary] One-liner
> DLLs loaded by [[LSASS]] that implement specific authentication protocols — they validate credentials, create logon sessions, and cache credential material in memory.

## How they fit in the stack
```
Application
  → [[SSPI and SSPs|SSPI]] (Security Support Provider Interface)
    → SSP / Authentication Package (loaded in LSASS)
      → validates credentials against local SAM / domain DC / etc.
      → creates Access Token + logon session
```

## Built-in authentication packages

| Package | DLL | Protocol | Notes |
|---|---|---|---|
| **MSV1_0** | `msv1_0.dll` | [[NTLM Authentication|NTLM]] | Validates against local [[SAM Database]] or passes to DC; caches NT hashes in LSASS |
| **Kerberos** | `kerberos.dll` | [[Kerberos]] | Domain authentication; caches TGTs/service tickets and keys in LSASS |
| **Negotiate** | `secur32.dll` | [[SPNEGO]] | Meta-package: chooses Kerberos (preferred) or NTLM fallback |
| **WDigest** | `wdigest.dll` | Digest/HTTP | **Caches plaintext passwords** in LSASS on older systems — see [[WDigest Downgrade]] |
| **CredSSP** | `credssp.dll` | TLS + SPNEGO | Used by RDP; can cache delegated credentials |
| **Schannel** | `schannel.dll` | TLS/SSL | Certificate-based authentication |
| **TSPKG** | `tspkg.dll` | CredSSP helper | Terminal Services SSP; can cache plaintext credentials |

## Red-team relevance
- **LSASS dumping** extracts whatever each loaded package has cached — NT hashes (MSV1_0), Kerberos tickets/keys, and sometimes plaintext passwords (WDigest, TSPKG).
- **WDigest abuse**: On Windows 8.1+/2012R2+, WDigest no longer caches plaintext by default — but setting `UseLogonCredential=1` in the registry re-enables it ([[WDigest Downgrade]]).
- **Custom SSPs**: An attacker can load a malicious SSP DLL (e.g., `mimilib.dll`) into LSASS to log all future authentications in plaintext — a persistence/credential-harvesting technique.
- **Mimikatz modules** map directly to packages: `sekurlsa::msv` (MSV1_0 hashes), `sekurlsa::kerberos` (Kerberos tickets), `sekurlsa::wdigest` (WDigest plaintext).

## Related
- Interface layer: [[SSPI and SSPs]].
- Host process: [[LSASS]].
- Credential caching overview: [[Credential Storage in Windows]].
- Specific protocols: [[NTLM Authentication]], [[Kerberos]], [[SPNEGO]].
