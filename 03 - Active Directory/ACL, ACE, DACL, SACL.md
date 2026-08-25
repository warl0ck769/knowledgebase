---
title: ACL, ACE, DACL, SACL
aliases: ["ACL", "ACE", "DACL", "SACL", "Security Descriptor", "SDDL"]
type: concept
domain: [active-directory, windows-internals]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Security Principal]]", "[[ACL Abuse]]", "[[AdminSDHolder]]", "[[SID]]", "[[DCSync]]"]
created: 2026-06-06
updated: 2026-06-06
---

# ACL, ACE, DACL, SACL

> [!summary] One-liner
> Every securable object carries a security descriptor whose DACL is a list of ACEs granting rights to principals — and the right ACE on the right object is a privesc.

## Security descriptor
Each securable object has a **security descriptor** containing:
- **Owner** (a [[SID]] — the owner can always rewrite the DACL),
- **DACL** (Discretionary ACL) — **who can do what**,
- **SACL** (System ACL) — **auditing** rules.

(Text form = **SDDL**.)

## ACE
An **Access Control Entry** is one line in an ACL: it grants/denies a specific **right** to a specific **principal** ([[Security Principal]], by SID).

## Rights that matter (abuse vectors)
| Right | What it lets you do |
|---|---|
| **GenericAll** | Full control — change any property/permission |
| **GenericWrite** | Write any property (e.g. SPN, scripts) |
| **WriteDACL** | Rewrite the DACL → grant yourself anything |
| **WriteOwner** | Make yourself owner → then rewrite DACL |
| **WriteProperty** | Write a specific property |
| **AllExtendedRights** | Incl. reset password, etc. |
| **ForceChangePassword** | Reset the target's password |
| **Self** | Add yourself to a **group** |
| **DS-Replication-Get-Changes (+ All)** | Replication rights → **[[DCSync]]** |

## Why a red teamer cares
ACLs are the hidden privilege graph of AD. Misconfigured ACEs let a low-priv user escalate one object at a time — see [[ACL Abuse]]; [[AdminSDHolder]] is how AD re-stamps ACLs on admins.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ACLs" (security descriptor, ACEs, rights)
