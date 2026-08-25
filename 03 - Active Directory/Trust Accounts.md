---
title: Trust Accounts
aliases: ["Trust account", "Trusted Domain Object", "DOMAIN$ account"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain and Forest Trusts]]", "[[Computer Accounts]]", "[[Inter-realm TGT]]", "[[Kerberos Keys]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Trust Accounts

> [!summary] One-liner
> When a trust is set up, each domain creates a `OTHERDOMAIN$` account that stores the shared trust key — stealing it lets you forge inter-realm tickets.

## What it is
Establishing a [[Domain and Forest Trusts|trust]] creates an associated **user object in each domain** that stores the **trust key**.

- **Naming:** NetBIOS name of the *other* domain + `$`.
  - Example — for a trust between **FOO** and **BAR**: domain **FOO** stores `BAR$`, domain **BAR** stores `FOO$`.
- **Stores:** the trust key as an **NT hash** and/or **[[Kerberos Keys|Kerberos keys]]** (depending on context).
- Behaves like a [[Computer Accounts|computer account]] in form.

## Why a red teamer cares
> [!danger]
> Compromising a trust account's secret enables forging an **[[Inter-realm TGT]]** — a referral ticket the trusting domain will accept — to move **across the trust** into the other domain/forest.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Trust accounts"
