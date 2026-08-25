---
title: NetBIOS
aliases: ["NetBIOS", "NBT-NS", "NetBIOS Name Service", "NBNS"]
type: concept
domain: [active-directory, windows-internals]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LLMNR-NBT-NS Poisoning]]", "[[Windows Host Enumeration]]", "[[SMB]]"]
created: 2026-06-06
updated: 2026-06-06
---

# NetBIOS

> [!summary] One-liner
> A legacy Windows name/session service over three ports (137/138/139); its broadcast name service is what NBT-NS poisoning abuses.

## What it is
NetBIOS over TCP/IP (**NBT**) provides legacy networking via three services:
| Port | Service | Role |
|---|---|---|
| **137** | **Name Service (NBT-NS / NBNS)** | Resolves NetBIOS names → IPs via **broadcast** (no auth) |
| **138** | Datagram Service | Connectionless broadcast/multicast messages |
| **139** | Session Service | Connection-oriented; historically carried **SMB/RPC** |

## Why a red teamer cares
- **Port 137** answers name queries unauthenticated → fingerprint hosts (`nbtscan`) and, more importantly, **poison** broadcast name requests when DNS fails → [[LLMNR-NBT-NS Poisoning]].
- Port 139 is a legacy SMB transport (modern SMB uses 445).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "NetBIOS" (Name/Datagram/Session services)
