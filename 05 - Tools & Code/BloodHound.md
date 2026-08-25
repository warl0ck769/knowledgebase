---
title: BloodHound
aliases: ["BloodHound", "BloodHound CE", "BloodHound Community Edition", "BloodHound enumeration"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "SpecterOps"
source_url: "https://github.com/SpecterOps/BloodHound"
verified: true
related: ["[[SharpHound]]", "[[ACL Abuse]]", "[[Kerberos Delegation]]", "[[AD Groups]]", "[[PowerView]]"]
created: 2026-06-07
updated: 2026-06-07
---

# BloodHound

> [!summary] One-liner
> A graph-based AD attack path analysis tool — ingests domain data via [[SharpHound]], builds a graph of relationships (group memberships, ACLs, sessions, trusts), and finds the shortest path from any compromised user to Domain Admin.

## How it works
1. **Collection**: [[SharpHound]] (or BloodHound.py) enumerates the domain — users, groups, computers, sessions, ACLs, trusts, GPOs.
2. **Ingestion**: Data is imported into a **Neo4j** graph database (legacy) or **PostgreSQL** (BloodHound CE).
3. **Analysis**: Cypher queries traverse the graph to find attack paths — shortest path to DA, ACL abuse chains, Kerberos delegation paths, etc.

## Key built-in queries

| Query | Finds |
|---|---|
| Shortest Path to Domain Admin | The fastest privilege escalation chain from any owned principal |
| Kerberoastable Users | Users with SPNs (→ [[Kerberoasting]]) |
| AS-REP Roastable Users | Users with "Do not require Kerberos pre-auth" |
| Unconstrained Delegation Computers | [[Unconstrained Delegation]] targets |
| Users with DCSync Rights | Principals with `Replicating Directory Changes [All]` |
| Shortest Paths from Owned Principals | Attack paths from principals you've already compromised |

## Key edge types

| Edge | Meaning |
|---|---|
| `MemberOf` | Group membership |
| `AdminTo` | Local admin on a computer |
| `HasSession` | User has a session on a computer (token-theft opportunity) |
| `GenericAll` / `GenericWrite` / `WriteDacl` / `WriteOwner` | [[ACL Abuse]] primitives |
| `AllowedToDelegate` / `AllowedToAct` | [[Kerberos Delegation]] paths |
| `CanRDP` / `CanPSRemote` | Remote access edges |
| `GPLink` | GPO linked to OU/domain |

## Usage
```bash
# Start BloodHound CE (Docker)
docker compose up -d

# Collect data with SharpHound
SharpHound.exe --CollectionMethods All --Domain <DOMAIN>

# Collect data with bloodhound-python (Linux)
bloodhound-python -d <DOMAIN> -u <USER> -p <PASS> -ns <DC_IP> -c all
```

## OPSEC notes
- SharpHound collection is noisy — it queries every computer for sessions and touches many LDAP attributes.
- `--Stealth` mode reduces noise (queries DCs/file servers only).
- Defenders use BloodHound too (BloodHound Enterprise / community edition) — if they have a graph, they know the same paths.

## Related
- Collector: [[SharpHound]].
- Complementary enumeration: [[PowerView]].
- Attack paths it maps: [[ACL Abuse]], [[Kerberos Delegation]], [[Kerberoasting]], [[DCSync]].
