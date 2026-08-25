---
title: Important Users
aliases: ["Built-in Administrator", "Privileged accounts"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[krbtgt account]]", "[[AD Groups]]", "[[SID History Abuse]]", "[[Golden Ticket]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Important Users

> [!summary] One-liner
> Two built-in accounts dominate AD compromise: the **Administrator** (full control) and **krbtgt** (the key to ticket forgery).

## Built-in Administrator
- The most privileged user — **can do anything on any domain computer**.
- Compromise = **domain control**, and potentially **forest control** via [[SID History Abuse]].
- Well-known **RID 500** in its domain.

## krbtgt
- Stores the secret used to **encrypt/sign Kerberos TGTs**.
- Compromise enables **[[Golden Ticket]]** forgery (mint arbitrary TGTs as anyone).
- Its secret is obtained by **dumping the domain database** — via **[[DCSync]]** or stealing `C:\Windows\NTDS\ntds.dit` (see [[NTDS.dit Extraction]]) — both requiring admin/DC privileges.
- Full detail: [[krbtgt account]].

## Why a red teamer cares
These are the crown-jewel identities. The whole AD kill chain (recon → cred access → privesc) usually funnels toward owning one of these.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Important Users"
