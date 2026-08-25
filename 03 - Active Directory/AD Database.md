---
title: AD Database
aliases: ["AD Database", "Classes", "Properties", "Partitions", "Naming Context", "Global Catalog", "Distinguished Name", "DN", "AD Schema", "GUID", "objectGUID"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/ldap]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LDAP]]", "[[SID]]", "[[Security Principal]]", "[[AD Database (NTDS.dit)]]", "[[AD User Object]]"]
created: 2026-06-06
updated: 2026-06-06
---

# AD Database

> [!summary] One-liner
> The directory data model: objects are instances of classes with attributes, named by Distinguished Names, split across partitions, and queried via LDAP/ADWS.

> The on-disk file is [[AD Database (NTDS.dit)|ntds.dit]]; this note is about the **logical model** and how to read it.

## Classes (object types)
Objects are instances of **classes** defining their structure:
- **User**, **Computer** (a subclass of User), **Group**, **organizationalUnit (OU)** … defined by the **Schema**.

## Properties (attributes)
Objects hold **attributes**: `sAMAccountName`, `distinguishedName`, `objectGUID`, `objectSID`, `operatingSystem`, `description`, `memberOf`, `servicePrincipalName`, … (see [[AD User Object]]).
- **objectGUID** = globally unique, immutable ID; **objectSID** = the [[SID]].

## Distinguished Name (DN)
LDAP path to an object, read right-to-left:
```
CN=Anakin,CN=Users,DC=contoso,DC=local
```
- **CN** = common name · **OU** = organizational unit · **DC** = domain component.

## Partitions (naming contexts)
The DB is split into partitions:
| Partition | Base DN | Holds |
|---|---|---|
| **Domain** | `DC=contoso,DC=local` | Users, computers, groups |
| **Configuration** | `CN=Configuration,DC=contoso,DC=local` | Forest-wide config (sites, services) |
| **Schema** | `CN=Schema,CN=Configuration,DC=contoso,DC=local` | Class/attribute definitions |
| **DomainDnsZones** | — | Domain DNS records (see [[ADIDNS]]) |
| **ForestDnsZones** | — | Forest-wide DNS |

## Global Catalog (GC)
A **partial, read-only replica of every object in the forest**, indexing commonly-searched attributes — enables **forest-wide** queries without contacting each DC. Ports **3268** (LDAP) / **3269** (LDAPS).

## Querying
Read/modify via **[[LDAP]]** (389/636), **ADWS** (9389, used by the PowerShell AD module), or RPC/**DRSUAPI** (replication → [[DCSync]]).

## Why a red teamer cares
The directory is the **map of the whole org** — classes/attributes/DNs/partitions tell you what to query and where; the **GC** gives forest-wide recon from one query. Tooling: [[LDAP Enumeration]], [[BloodHound enumeration]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Database" (classes, properties, principals, DN, partitions, Global Catalog)
