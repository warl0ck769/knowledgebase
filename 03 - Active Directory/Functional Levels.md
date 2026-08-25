---
title: Functional Levels
aliases: ["Functional Modes", "Domain Functional Level", "Forest Functional Level"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain]]", "[[Forest]]", "[[Protected Users Group]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Functional Levels

> [!summary] One-liner
> A version setting on a domain/forest that gates which AD features are available, named after the minimum Windows Server version required.

## What it is
Domains and forests have **functional modes/levels**, each named after the **minimum Windows Server version** the DCs must run. Reported values include:

`Windows2000` · `Windows2000MixedDomains` · `Windows2003` · `Windows2008` · `Windows2008R2` · `Windows2012` · `Windows2012R2` · `Windows2016`

## Why it matters (feature gating)
The level determines which features exist. Example: the **[[Protected Users Group]]** (a credential-theft mitigation) requires at least **Windows2012R2** functional level.

## Why a red teamer cares
- The level tells you which **defenses may be present or absent** (e.g. Protected Users, newer Kerberos protections).
- A low functional level can mean legacy, weaker configurations are in play.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Functional Modes"
