---
title: Security Principal
aliases: ["Security Principal", "Principal"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[SID]]", "[[AD User Object]]", "[[Computer Accounts]]", "[[AD Groups]]", "[[ACL, ACE, DACL, SACL]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Security Principal

> [!summary] One-liner
> Any entity that can be assigned permissions — i.e. that has a SID: users, computers, and groups.

## What it is
A **security principal** is an object that can hold rights/permissions and be referenced in access control. In AD these are **users, computers, and groups**. Each has a [[SID]] used throughout the security model (tokens, [[ACL, ACE, DACL, SACL|ACLs]]).

## Why a red teamer cares
- ACLs grant rights **to principals (by SID)** — so abusing permissions ([[ACL Abuse]]) means controlling a principal that has them.
- Non-principals (e.g. OUs, GPOs) **organize/apply** policy but aren't assigned rights the same way.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Database → Principals"
