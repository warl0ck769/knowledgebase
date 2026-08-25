---
title: UserAccountControl
aliases: ["UAC flags", "userAccountControl", "DONT_REQUIRE_PREAUTH", "TRUSTED_FOR_DELEGATION", "TRUSTED_TO_AUTH_FOR_DELEGATION"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD User Object]]", "[[AS-REP Roasting]]", "[[Unconstrained Delegation]]", "[[Constrained Delegation]]", "[[Privileges and Rights]]"]
created: 2026-06-06
updated: 2026-06-06
---

# UserAccountControl

> [!summary] One-liner
> A bit-flag attribute on a user/computer object that controls account behavior — several flags directly enable or block attacks.

## What it is
`userAccountControl` is a property holding **flags** that govern an account's security and domain behavior. The attack-relevant ones:

| Flag | Meaning | Offensive relevance |
|---|---|---|
| **ACCOUNTDISABLE** | Account disabled, unusable | Skip these in targeting |
| **DONT_REQUIRE_PREAUTH** | Kerberos **pre-auth not required** | Enables **[[AS-REP Roasting]]** — request AS-REP and crack offline |
| **NOT_DELEGATED** | Account **cannot be delegated** | A protection; blocks delegation of that identity |
| **TRUSTED_FOR_DELEGATION** | Enables **[[Unconstrained Delegation]]** | Huge: this host can capture/impersonate any user that authenticates to it. Modifying requires **`SeEnableDelegationPrivilege`** |
| **TRUSTED_TO_AUTH_FOR_DELEGATION** | Enables Kerberos **S4U2Self** (protocol transition) | Part of **[[Constrained Delegation]]** abuse. Modifying requires **`SeEnableDelegationPrivilege`** |

## Why a red teamer cares
- Enumerating UAC flags instantly surfaces juicy targets: AS-REP-roastable accounts, unconstrained-delegation hosts, constrained-delegation principals.
- Setting these flags is a known **privesc/persistence** path — but the delegation flags are gated by the powerful **`SeEnableDelegationPrivilege`** (see [[Privileges and Rights]]).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "UserAccountControl"
