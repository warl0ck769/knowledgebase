---
title: krbtgt account
aliases: ["krbtgt", "KRBTGT"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Important Users]]", "[[Golden Ticket]]", "[[DCSync]]", "[[NTDS.dit Extraction]]", "[[KDC]]", "[[TGT vs TGS]]"]
created: 2026-06-06
updated: 2026-06-06
---

# krbtgt account

> [!summary] One-liner
> The special domain account whose key the KDC uses to encrypt/sign every TGT — owning its hash means you can forge tickets for anyone (Golden Ticket).

## What it is
`krbtgt` is a built-in, disabled service account. The [[KDC]] uses **krbtgt's key** to **encrypt and sign every [[TGT vs TGS|TGT]]** it issues. So a valid TGT is, in effect, "anything encrypted with the krbtgt key."

## Why it's the crown jewel
- If you have **krbtgt's hash/AES key**, you can **mint your own TGTs** for any user, with any privileges — a **[[Golden Ticket]]**. The KDC trusts them because they're correctly encrypted.
- This is **domain-wide persistence**: valid until the krbtgt password is reset **twice** (the account keeps current+previous keys).

## How attackers get it
Requires dumping domain secrets — both need DC/admin-level access:
- **[[DCSync]]** — ask a DC to replicate the krbtgt secret over the directory-replication protocol.
- **[[NTDS.dit Extraction]]** — steal `C:\Windows\NTDS\ntds.dit` (the [[AD Database (NTDS.dit)|AD database]]).

## Why a red teamer cares
krbtgt is the single highest-value secret in a domain. Goal-of-goals for full, durable domain control.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Important Users → krbtgt"
