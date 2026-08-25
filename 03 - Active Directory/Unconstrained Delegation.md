---
title: Unconstrained Delegation
aliases: ["Unconstrained Delegation", "TRUSTED_FOR_DELEGATION"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos, attack/privesc]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos Delegation]]", "[[Constrained Delegation]]", "[[Resource-Based Constrained Delegation (RBCD)]]", "[[UserAccountControl]]", "[[PetitPotam]]", "[[LSASS Dumping]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Unconstrained Delegation

> [!summary] One-liner
> A host flagged for unconstrained delegation receives every user's TGT inside their service ticket — compromise the host and you can impersonate anyone who connected to it, anywhere.

## How it works (concept)
When a service account has **`TRUSTED_FOR_DELEGATION`** ([[UserAccountControl]]):
1. A user requests an ST for that service.
2. The KDC **embeds the user's TGT inside the ST**.
3. The service caches that TGT and can reuse it to access **any** other service **as the user** — unrestricted.

## The attack
1. Compromise an unconstrained-delegation host.
2. **Extract cached TGTs** from [[LSASS]] (`sekurlsa::tickets` → [[LSASS Dumping]]) or monitor with `Rubeus.exe monitor`.
3. **Coerce a privileged target** (esp. a **DC**) to authenticate to your host (printerbug / [[PetitPotam]]) → capture the **DC's TGT** → [[DCSync]] → domain compromise.

```powershell
Rubeus.exe monitor /interval:5 /nowrap      # harvest incoming TGTs
# coerce a DC to connect (SpoolSample / PetitPotam), then use the captured TGT
```

## Across forests
With a forest trust, an unconstrained host in forest A can capture TGTs of forest B users that authenticate to it → lateral movement into forest B. (Modern mitigations restrict TGT delegation across forest trusts.)

## Anti-delegation measures
- **Protected Users** & **`NOT_DELEGATED`** accounts cannot be delegated (their TGTs won't be forwarded).

## Why a red teamer cares
Owning any unconstrained host + a coercion primitive is a classic, reliable path to **Domain Admin**.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Kerberos Unconstrained Delegation" (+ across forests, anti-delegation measures)
