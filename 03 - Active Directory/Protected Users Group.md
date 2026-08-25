---
title: Protected Users Group
aliases: ["Protected Users"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos, proto/ntlm]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Privileged AD Groups]]", "[[Functional Levels]]", "[[NTLM Relay]]", "[[Unconstrained Delegation]]", "[[Constrained Delegation]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Protected Users Group

> [!summary] One-liner
> A defensive group whose members get hardened authentication — no NTLM, no delegation — to frustrate credential-theft attacks.

## What it is
A built-in group (requires **Windows2012R2** [[Functional Levels|functional level]]) that **restricts how its members authenticate** to reduce credential exposure.

## Protections applied to members
- **Cannot authenticate with NTLM** — Kerberos only (defeats [[NTLM Relay]] / [[Pass-the-Hash]] against them).
- **Cannot be delegated** via unconstrained or constrained delegation (defeats [[Unconstrained Delegation]] / [[Constrained Delegation]] abuse of those identities).
- (Also: no RC4/DES for Kerberos, no long-lived TGT caching, no CredSSP/WDigest plaintext — standard hardening.)

## Why a red teamer cares
- If a target account is in Protected Users, your **NTLM-relay and delegation** plays against it won't work — pivot to other identities or techniques.
- Conversely, **privileged accounts NOT in Protected Users** are softer targets — check membership during recon.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Other important groups → Protected Users"
