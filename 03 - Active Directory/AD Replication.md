---
title: AD Replication
aliases: ["AD replication", "DRS", "DRSUAPI", "Directory Replication Service"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/replication/active-directory-replication-concepts"
verified: true
related: ["[[Domain Controller (DC)]]", "[[AD Database (NTDS.dit)]]", "[[Sites and Subnets]]", "[[DCSync]]", "[[AD Replication Metadata (Detection)]]"]
created: 2026-06-07
updated: 2026-06-07
---

# AD Replication

> [!summary] One-liner
> The process by which Domain Controllers synchronize the AD database with each other — multi-master by default, using the DRSUAPI (MS-DRSR) protocol over RPC, with conflict resolution via USNs and version vectors.

## How it works
1. A change is made on DC-A (e.g., password reset) → DC-A increments its **USN** (Update Sequence Number) and stamps the attribute with a new `version` + `originating timestamp`.
2. DC-A **notifies** replication partners (intra-site: within ~15 seconds).
3. Partner DCs request changes since their last-known USN for DC-A → DC-A sends the delta.
4. If two DCs change the same attribute simultaneously, **conflict resolution**: higher version wins; if equal, later timestamp wins; if still tied, higher originating DC GUID wins.

## Replication protocol (DRSUAPI / MS-DRSR)
- **`DRSGetNCChanges`**: the RPC call that requests replication data — this is exactly what [[DCSync]] abuses.
- Requires **Replicating Directory Changes** (and optionally **Replicating Directory Changes All** for secrets) extended rights on the domain object.

## What replicates

| Partition | Scope | Contents |
|---|---|---|
| **Domain** | Per-domain (all DCs in that domain) | Users, groups, computers, GPOs, OUs |
| **Configuration** | Forest-wide | Sites, subnets, services, replication topology |
| **Schema** | Forest-wide | Class and attribute definitions |
| **Application** (e.g., DomainDnsZones) | Configurable | DNS zones, custom app data |

## Replication metadata
Every attribute has metadata: `version`, `originating DC`, `originating USN`, `originating time`. This metadata is used for conflict resolution and is also valuable for **detection** — see [[AD Replication Metadata (Detection)]].

```powershell
# View replication metadata for an object
Get-ADReplicationAttributeMetadata -Object "CN=krbtgt,CN=Users,DC=corp,DC=local" -Server <DC> -ShowAllLinkedValues

# Replication status
repadmin /showrepl
repadmin /replsummary
```

## Red-team relevance
- **[[DCSync]]**: Mimics a DC requesting replication (`DRSGetNCChanges`) to extract password hashes for any account — requires `Replicating Directory Changes [All]` rights.
- **Replication lag**: Inter-site replication delay ([[Sites and Subnets]]) creates a window where changes haven't propagated — useful for attackers (e.g., racing a password change).
- **[[AD Replication Metadata (Detection)]]**: Defenders can inspect replication metadata to detect unauthorized changes (e.g., SID History injection, krbtgt key changes).

## Related
- Protocol abused: [[DCSync]] (DRSGetNCChanges).
- Database replicated: [[AD Database (NTDS.dit)]].
- Topology: [[Sites and Subnets]], [[Domain Controller (DC)]].
- Detection: [[AD Replication Metadata (Detection)]].
