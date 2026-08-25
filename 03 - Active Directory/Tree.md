---
title: Tree
aliases: ["Domain Tree", "AD Tree"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Domain]]", "[[Forest]]", "[[Domain and Forest Trusts]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Tree

> [!summary] One-liner
> A hierarchy of a parent domain and its subdomains sharing a contiguous DNS namespace.

## What it is
When a [[Domain]] has **subdomains** (for organizational purposes), the parent and its children form a **tree** — a contiguous DNS namespace (e.g. `corp.contoso.com` under `contoso.com`).

## How it works
- Parent and child domains are joined by **bidirectional, transitive** trusts automatically (see [[Trust Direction and Transitivity]]).
- One or more trees grouped together form a [[Forest]]; trees in the same forest can have **different** DNS namespaces but still trust each other.

## Why a red teamer cares
- Automatic parent↔child transitive trust means access can **traverse** the tree — privilege in one domain may reach others (see [[Domain and Forest Trusts]], [[SID History Abuse]]).

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Forests" (tree structure)
