---
title: Computer Accounts
aliases: ["Computer account", "Machine account", "NAME$", "Computer object"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD User Object]]", "[[Silver Ticket]]", "[[Resource-Based Constrained Delegation (RBCD)]]", "[[Domain Controller (DC)]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Computer Accounts

> [!summary] One-liner
> Every domain-joined machine has its own account (a User subclass named `NAME$`) — and these accounts are abusable for Silver Tickets and RBCD.

## What it is
Each domain computer holds **its own account** used for domain operations (pulling Group Policy, verifying user credentials, etc.).

- **Class:** `Computer` — a **subclass of `User`** (so computer accounts *are* user objects with extra attributes).
- **Naming:** hostname + `$` → e.g. `DC01$`, `WS01-10$`.
- **Notable attributes:** `operatingSystem`, `operatingSystemVersion` (useful for fingerprinting targets).

## Why a red teamer cares
A computer account, even with **no local admin rights**, is leverage:
- **[[Silver Ticket]]** — its key signs service tickets for services running as that machine (e.g. CIFS, HOST).
- **[[Resource-Based Constrained Delegation (RBCD)]]** — if you control a computer object's `msDS-AllowedToActOnBehalfOfOtherIdentity`, you can impersonate users to it and gain admin access.
- Machine account passwords are often **long/random but recoverable** from LSASS/registry, and machine accounts can create other computer objects (default `MachineAccountQuota = 10`).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Computer accounts"
