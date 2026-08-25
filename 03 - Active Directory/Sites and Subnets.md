---
title: Sites and Subnets
aliases: ["AD site", "AD subnet", "site link", "inter-site replication"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/creating-a-site-design"
verified: true
related: ["[[Domain Controller (DC)]]", "[[AD Replication]]", "[[Global Catalog]]", "[[Domain]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Sites and Subnets

> [!summary] One-liner
> AD Sites represent physical network locations (mapped to IP subnets) — they control which DC a client authenticates to and how replication traffic flows between locations.

## Core concepts

| Object | Purpose |
|---|---|
| **Site** | A set of well-connected IP subnets (typically a physical location — office, datacenter) |
| **Subnet** | An IP range (e.g., `10.1.0.0/16`) mapped to a site |
| **Site Link** | Defines replication paths between sites + cost/schedule |
| **ISTG (Inter-Site Topology Generator)** | Automatically builds replication topology between sites |
| **Bridgehead server** | DC chosen per site to handle inter-site replication traffic |

## How clients use sites
1. Client boots → gets IP from DHCP.
2. Client queries DNS for `_ldap._tcp.dc._msdcs.<domain>` → gets list of DCs.
3. Client contacts a DC → DC checks client IP against subnet-to-site mapping → tells client which **site** it's in.
4. Client then contacts a DC **in its own site** (fastest link).

## Replication

| Type | Within a site | Between sites |
|---|---|---|
| **Trigger** | Change-notification (near-instant, ~15 sec) | Scheduled (default: every 180 min per site link) |
| **Compression** | No | Yes (saves bandwidth) |
| **Protocol** | RPC over IP | RPC over IP (or SMTP for schema/config — rare) |

## Red-team relevance
- **DC affinity**: Knowing site topology tells you which DC a target machine authenticates to — useful for targeted attacks (NTLM relay, DC compromise).
- **Replication delay**: Inter-site replication lag (up to 3 hours by default) means password changes, group membership changes, and GPO updates don't propagate instantly — an attacker can exploit this window.
- **Site enumeration**: Reveals network topology / physical office locations.

```powershell
# Enumerate sites
Get-ADReplicationSite -Filter *
nltest /dsgetsite

# Enumerate subnets and their site assignments
Get-ADReplicationSubnet -Filter * | Select Name, Site

# Site links
Get-ADReplicationSiteLink -Filter *
```

## Related
- DC placement: [[Domain Controller (DC)]].
- Replication: [[AD Replication]].
- Client DC selection: [[Global Catalog]] (GC per site).
