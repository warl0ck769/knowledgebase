---
title: Forest
aliases:
  - AD Forest
  - Forest root domain
type: concept
domain:
  - active-directory
tags:
  - type/concept
  - domain/active-directory
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: https://zer1t0.gitlab.io/posts/attacking_ad/
verified: true
related:
  - "[[Domain]]"
  - "[[Tree]]"
  - "[[Functional Levels]]"
  - "[[Domain and Forest Trusts]]"
created: 2026-06-06
updated: 2026-06-06
---

# Forest

> [!summary] One-liner
> The top-level container of one or more domains/trees that trust each other — and AD's real security boundary.

## What it is
A **forest** is a collection of [[Domain|domains]] (organized into one or more [[Tree|trees]]). It is **named after its root domain** (the first domain created = the *forest root domain*).

## How it works
- Each domain keeps **its own database and its own [[Domain Controller (DC)|Domain Controllers]]**.
- Within a forest, domains are linked by **bidirectional transitive trusts**, so a user can access resources across domains by traversing trusts (see [[Trust Direction and Transitivity]]).
- **By default, users in one forest cannot access resources in another forest** — forests provide isolation.

## The forest is the security boundary
> [!important]
> The **forest**, not the domain, is the true security boundary in AD. Compromising one domain in a forest generally implies the whole forest can be compromised (e.g. via [[SID History Abuse]] / trust-key abuse). zer1t0 cites SpecterOps' *"Not A Security Boundary: Breaking Forest Trusts."*

## Functional modes
A forest (and each domain) runs at a **functional mode/level** that gates available features — see [[Functional Levels]].

## Why a red teamer cares
- Forest-wide trust + DC autonomy per domain means **domain compromise often → forest compromise**.
- Cross-forest trusts are a primary lateral-movement avenue ([[Domain and Forest Trusts]]).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Forests"
