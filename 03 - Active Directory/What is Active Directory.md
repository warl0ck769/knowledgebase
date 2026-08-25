---
title: What is Active Directory
aliases: ["Active Directory", "AD", "AD DS"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain]]", "[[Forest]]", "[[Domain Controller (DC)]]", "[[AD Database (NTDS.dit)]]"]
created: 2026-06-06
updated: 2026-06-06
---

# What is Active Directory

> [!summary] One-liner
> A system for managing a set of computers and users on the same network from a central server, backed by a centralized database.

## What it is
Active Directory (AD) lets an organization administer many users and computers **centrally** instead of machine-by-machine. A central database holds information about **users, computers, policies, and permissions**. The central servers that host and manage this database are [[Domain Controller (DC)|Domain Controllers]].

## How it works
- IT teams perform admin tasks **remotely** — installing software, resetting passwords, managing file permissions — without touching individual machines.
- When a user logs into a domain-joined computer, the machine consults the central database to **authenticate** the user.
- This enables **single sign-on (SSO)**: one identity grants access across organizational resources.

## Why a red teamer cares
AD centralizes identity and trust for an entire organization. Compromising the right object (a DC, the [[krbtgt account]], a privileged user) can cascade into **control of the whole domain or forest**. The central database and authentication flows are the prime targets — see [[MOC - Attacks & Techniques]].

## Key terms
- **Domain** — a set of computers sharing one AD database → [[Domain]].
- **Domain Controller** — server hosting/managing that database → [[Domain Controller (DC)]].
- **Forest** — the top-level security container of one or more domains → [[Forest]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "What is Active Directory?"
