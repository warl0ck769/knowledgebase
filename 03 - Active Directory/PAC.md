---
title: PAC
aliases: ["PAC", "Privileged Attribute Certificate"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[TGT vs TGS]]", "[[krbtgt account]]", "[[Golden Ticket]]", "[[Silver Ticket]]", "[[MS14-068]]", "[[SID History Abuse]]"]
created: 2026-06-06
updated: 2026-06-06
---

# PAC

> [!summary] One-liner
> The authorization blob inside a Kerberos ticket that says who you are and what groups/privileges you have — signed by the KDC and the service.

## What it is
The **Privileged Attribute Certificate (PAC)** is authorization data embedded in a ticket:
- The user's **SID**
- **Group memberships (SIDs)**
- **Privileges / rights**

## Signing
The PAC is **signed twice**:
- by the **KDC** using the **[[krbtgt account|krbtgt]] key**, and
- by the **service** using its own key.

This lets the service verify the PAC's authenticity/integrity.

## Why a red teamer cares
The PAC is **where privilege lives** in a ticket — control it and you control your effective access:
- **[[Golden Ticket]]** — forge a TGT with a PAC claiming Domain Admins (signed with the stolen krbtgt key, so it validates).
- **[[Silver Ticket]]** — forge an ST with a chosen PAC, signed with the service key (the service often doesn't re-validate the KDC signature).
- **[[MS14-068]]** — historic bug letting a normal user forge a valid PAC (privilege escalation).
- **[[SID History Abuse]]** — inject extra SIDs (e.g. Enterprise Admins) into the PAC to escalate across domains.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Tickets → PAC"
