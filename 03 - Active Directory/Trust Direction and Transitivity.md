---
title: Trust Direction and Transitivity
aliases: ["Trust direction", "Trust transitivity", "Inbound trust", "Outbound trust"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain and Forest Trusts]]", "[[Forest]]", "[[Tree]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Trust Direction and Transitivity

> [!summary] One-liner
> Trust direction is the **opposite** of access direction; transitivity decides whether a trust can be "chained" through intermediate domains.

## Direction — the key mental model
> [!important]
> **Trust direction is opposite to access direction.** The trust arrow points from the **trusting** domain to the **trusted** domain; *access* flows the other way (trusted users → trusting resources).

| Term | Meaning |
|---|---|
| **Inbound / Incoming trust** | *Your* domain's users can access the other domain |
| **Outbound / Outgoing trust** | The *other* domain's users can access *your* domain |
| **Bidirectional** | Both directions at once |

## Transitivity
- **Transitive** trust = acts as a **bridge**: access can pass *through* intermediate domains (A trusts B, B trusts C ⇒ A effectively reachable from C's side via the chain).
- **Nontransitive** trust = access limited to the **two directly connected** domains only.
- **In a [[Forest]], parent↔child domains use bidirectional transitive trusts**, so any domain can reach any other by traversing the necessary trusts (see [[Tree]]).

## Enumeration
```
:: List all trusts of the current domain (type, direction)
nltest /domain_trusts
```
Output annotates each trust, e.g. `(Direct Outbound)`, `(Forest: 2)`.

## Why a red teamer cares
- **Transitive + bidirectional** intra-forest trusts mean compromise can **chain across the whole forest**.
- Knowing direction tells you **which way access flows** — i.e. where a stolen identity is actually usable.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Trust direction" / "Trust transitivity"
