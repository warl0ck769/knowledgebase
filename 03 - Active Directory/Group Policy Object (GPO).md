---
title: Group Policy Object (GPO)
aliases: ["GPO", "Group Policy", "GPO Processing", "Group Policy Container", "Group Policy Template", "SYSVOL", "GPC", "GPT"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[GPO Abuse]]", "[[Organizational Unit (OU)]]", "[[AD Database]]", "[[Privileged AD Groups]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Group Policy Object (GPO)

> [!summary] One-liner
> A policy bundle (settings, scripts, deployments) applied to OUs/domains/sites; it has an AD half (GPC) and a SYSVOL file half (GPT).

## Two halves
| Component | Where | Holds |
|---|---|---|
| **GPC (Group Policy Container)** | AD: `CN=Policies,CN=System,DC=domain,DC=local` | Metadata, version, links |
| **GPT (Group Policy Template)** | **SYSVOL**: `\\domain\SYSVOL\domain\Policies\{GUID}\` | The actual files: scripts, registry settings, software packages |

Both are keyed by the GPO's **GUID**.

## Scope / linking
A GPO is **linked** to a **Site**, **Domain**, or **[[Organizational Unit (OU)]]**; it applies to all users/computers under that scope. Inheritance and block/enforce control the effective set.

## Why a red teamer cares
- A GPO linked to an OU full of machines = **mass code execution surface**. Write access to that GPO → run code / add admins / scheduled tasks on **every linked host** → [[GPO Abuse]].
- **SYSVOL** is world-readable to domain users — historically a place to find secrets (e.g. GPP `cpassword`).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Group Policy" (scope, template, container)
