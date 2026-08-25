---
title: AdminSDHolder
aliases: ["AdminSDHolder", "SDProp", "adminCount"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, attack/persistence]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[ACL, ACE, DACL, SACL]]", "[[Privileged AD Groups]]", "[[ACL Abuse]]", "[[AD User Object]]"]
created: 2026-06-06
updated: 2026-06-06
---

# AdminSDHolder

> [!summary] One-liner
> A template object whose ACL is force-copied onto privileged accounts every 60 minutes — a defense that doubles as a stealthy persistence mechanism.

## What it is
**AdminSDHolder** holds a reference security descriptor. A background process, **SDProp (Security Descriptor Propagator)**, runs **every ~60 minutes** and **re-applies that ACL** to members of protected groups ([[Privileged AD Groups]] like Domain Admins, Enterprise Admins). Protected objects get **`adminCount = 1`**.

## Two-edged
- **Defense:** ensures admins' ACLs aren't tampered with — resets unauthorized changes.
- **Attack/persistence:** if you can **modify AdminSDHolder's ACL** (e.g. add yourself with [[ACL, ACE, DACL, SACL|GenericAll]]), SDProp **propagates your backdoor ACE onto every protected admin** within an hour — durable, quiet domain persistence.

## Why a red teamer cares
- `adminCount=1` flags current/former privileged objects during recon (note: can be stale).
- Editing AdminSDHolder is a classic Tier-0 persistence — see [[ACL Abuse]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "ACL attacks → AdminSDHolder"
