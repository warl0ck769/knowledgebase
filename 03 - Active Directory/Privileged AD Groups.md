---
title: Privileged AD Groups
aliases: ["Domain Admins", "Enterprise Admins", "Schema Admins", "Account Operators", "Backup Operators", "Server Operators", "DnsAdmins", "Group Policy Creator Owners"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Groups]]", "[[Important Users]]", "[[Protected Users Group]]", "[[GPO Abuse]]", "[[DCSync]]", "[[AdminSDHolder]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Privileged AD Groups

> [!summary] One-liner
> The built-in groups whose membership grants (or quickly leads to) domain/forest control — the targets of most AD privilege escalation.

## Top-tier (effectively domain/forest owners)
| Group | Power | Notes |
|---|---|---|
| **Domain Admins** | Admin over the whole **domain** | Added to every machine's local Administrators by default (DA = local admin everywhere) |
| **Enterprise Admins** | Admin over **all domains in the forest** | Exists only in **forest root**; added to Administrators in every domain |
| **Administrators** (built-in) | Full control in the domain | Domain Local scope |
| **Schema Admins** | Can modify the **AD schema** | Forest-wide structural change |

> Related crown-jewel **accounts** (not groups): built-in **Administrator** and **[[krbtgt account|krbtgt]]** → see [[Important Users]].

## "Tier-0 adjacent" groups (each is a privesc path to DC/domain)
| Group | Why it's dangerous |
|---|---|
| **DnsAdmins** | Can load an **arbitrary DLL** into the DNS service → **code execution as `SYSTEM` on a DC** |
| **Backup Operators** | Back up/restore files on DCs **and log in** → can read/modify DC files (e.g. exfil [[NTDS.dit Extraction|NTDS.dit]]) |
| **Account Operators** | Modify many group memberships (not the top admin groups) — **but can modify Server Operators** |
| **Server Operators** | Can **log into DCs** (and manage services) |
| **Print Operators** | Can **log into DCs** |
| **Remote Desktop Users** | Can log into DCs via **RDP** |
| **Group Policy Creator Owners** | Can **edit GPOs** → [[GPO Abuse]] |

## Why a red teamer cares
Membership in any of these is a direct or one-hop route to a DC / domain takeover. Enumerate them first (e.g. [[BloodHound enumeration]]) and chase the shortest path. Note [[AdminSDHolder]] re-stamps ACLs on members of protected admin groups.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Important groups (Administrative / Other)"
