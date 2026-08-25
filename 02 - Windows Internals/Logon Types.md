---
title: Logon Types
aliases: ["Logon Types", "Interactive logon", "Network logon", "Service logon", "RemoteInteractive logon", "Type 3 logon"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LSASS]]", "[[Credential Storage in Windows]]", "[[Pass-the-Hash]]", "[[LSASS Dumping]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Logon Types

> [!summary] One-liner
> How a session authenticates determines whether reusable credentials get cached on that machine — which is exactly what an attacker hunts for.

## The types (and credential caching)
| # | Type | When | Creds cached in LSASS? |
|---|---|---|---|
| **2** | **Interactive** | Console / RDP credential entry | **Yes** — NT hash, Kerberos keys, tickets |
| **3** | **Network** | Access to shares/RPC over the net | **No** (no reusable creds left on the remote box) |
| **4** | **Batch** | Scheduled tasks | Depends on config |
| **5** | **Service** | Service running as an account | Service password in **[[LSA Secrets]]** |
| **8** | **NetworkCleartext** | Plaintext over network (HTTP Basic, Digest) | Password available to the service |
| **9** | **NewCredentials** | `runas /netonly` | New creds cached for **network** use |
| **10** | **RemoteInteractive** | RDP | **Yes** — cached on the RDP target |

> (Type 7 = unlock, Type 11 = CachedInteractive also exist.)

## Why a red teamer cares
- **Type 2 / 10** sessions leave **reusable credentials** in [[LSASS]] → prime [[LSASS Dumping]] targets. Compromising a server where a **Domain Admin** RDP'd in (type 10) yields their creds.
- **Type 3** (network) is "safe to land on" for the victim — no reusable creds cached — which is why pass-the-hash uses it but doesn't *leak* new creds there.
- This drives **target selection**: hunt machines where privileged users log on **interactively**.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Logon types"
