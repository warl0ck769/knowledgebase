---
title: DPAPI
aliases: ["Data Protection API", "CryptProtectData", "CryptUnprotectData", "masterkey"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/seccng/cng-dpapi"
verified: true
related: ["[[Credential Storage in Windows]]", "[[LSA Secrets]]", "[[DPAPI Abuse]]", "[[Attacking KeePass]]"]
created: 2026-06-07
updated: 2026-06-07
---

# DPAPI (Data Protection API)

> [!summary] One-liner
> A Windows API that lets any application encrypt data tied to the current user (or machine) context — the encryption key is derived from the user's password and managed transparently by the OS.

## Core API
Two functions in `Crypt32.dll` (exposed in .NET as `System.Security.Cryptography.ProtectedData`):
- **`CryptProtectData`** — encrypts a blob; optionally ties it to the current user (`CRYPTPROTECT_LOCAL_MACHINE` flag for machine scope).
- **`CryptUnprotectData`** — decrypts; only succeeds if called in the same user/machine context (or with the masterkey).

Applications call these without managing keys — Windows handles everything.

## Masterkeys
DPAPI uses a hierarchy of keys:

```
User's password (or NT hash)
  └── derives → User masterkey(s)
       └── encrypts → individual DPAPI blobs
            (browser passwords, credentials, vault entries...)
```

| Key type | Location | Protected by |
|---|---|---|
| **User masterkey** | `%APPDATA%\Microsoft\Protect\<SID>\` | User's password (PBKDF2) |
| **Machine masterkey** | `%SYSTEMROOT%\System32\Microsoft\Protect\` | `DPAPI_SYSTEM` [[LSA Secrets|LSA secret]] |
| **Domain backup key** | Stored on DCs | The domain's RSA backup keypair |

Masterkeys rotate (default: 90 days) but old ones are kept — old blobs still decrypt with old masterkeys.

## What uses DPAPI

| Consumer | What's encrypted |
|---|---|
| **Credential Manager / Vault** | Saved Windows credentials, RDP passwords |
| **Chrome / Edge** | Saved passwords, cookies, autofill |
| **Wi-Fi** | WPA/WPA2 PSK profiles |
| **EFS** | User certificate private keys |
| **Scheduled Tasks** | Stored credentials |
| **KeePass** | "Windows user account" composite key component |
| **Custom apps** | Anything using `ProtectedData.Protect()` |

## Domain backup key
On domain-joined machines, every user masterkey is also encrypted with the **domain DPAPI backup key** (an RSA keypair on the DCs). This allows password resets without losing access to encrypted data — but also means anyone who obtains the domain backup key can decrypt **every domain user's** DPAPI-protected secrets. See [[DPAPI Abuse]].

## Related
- Part of: [[Credential Storage in Windows]] (DPAPI as a storage mechanism).
- Machine keys stored in: [[LSA Secrets]] (`DPAPI_SYSTEM`).
- Offensive abuse: [[DPAPI Abuse]] (masterkey extraction, domain backup key theft).
- Downstream consumers: [[Attacking KeePass]] (KcpUserAccount component).
