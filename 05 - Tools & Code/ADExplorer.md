---
title: ADExplorer
aliases: ["AD Explorer", "ADExplorer.exe", "Sysinternals AD Explorer"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Microsoft Sysinternals"
source_url: "https://learn.microsoft.com/en-us/sysinternals/downloads/adexplorer"
verified: true
related: ["[[LDAP]]", "[[AD Database (NTDS.dit)]]", "[[LDAP Enumeration]]", "[[BloodHound]]"]
created: 2026-06-07
updated: 2026-06-07
---

# ADExplorer

> [!summary] One-liner
> A Sysinternals GUI tool for browsing and snapshotting the Active Directory database via LDAP — view all objects and attributes, take offline snapshots for later analysis, and compare snapshots to detect changes.

## Key features
- **Browse**: Navigate the AD tree, view all attributes on any object (including security descriptors, SPNs, group memberships).
- **Snapshot**: Save a complete offline copy of the AD database (`.dat` file) — can be analyzed without network access.
- **Compare**: Diff two snapshots to detect changes (new accounts, modified ACLs, group membership changes).
- **Search**: LDAP query builder with GUI.

## Red-team usage
```
# Take a snapshot (useful for offline enum / exfiltration)
ADExplorer.exe → Connect to <DC> → File → Create Snapshot

# Convert snapshot to BloodHound-compatible format
ADExplorerSnapshot.py <snapshot.dat> -o bloodhound_data/
```

## OPSEC notes
- Signed Microsoft binary (Sysinternals) — won't trigger most AV.
- Snapshot captures the entire AD database contents — treat as sensitive.
- The snapshot file can be converted for [[BloodHound]] ingestion using third-party tools.

## Related
- LDAP browsing: [[LDAP]], [[LDAP Enumeration]].
- Graph analysis: [[BloodHound]], [[SharpHound]].
