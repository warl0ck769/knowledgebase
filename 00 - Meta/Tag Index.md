---
title: Tag Index
type: moc
created: 2026-06-06
updated: 2026-06-06
---

# Tag Index

A controlled vocabulary. **Only use tags from this list** — consistency is what makes the vault trainable. If you need a new tag, add it here first.

## `type/` — what kind of note
- `#type/concept`
- `#type/technique`
- `#type/tool`
- `#type/cheatsheet`
- `#type/moc`
- `#type/source`

## `domain/` — subject area
- `#domain/active-directory`
- `#domain/windows-internals`
- `#domain/red-team`

## `attack/` — MITRE ATT&CK tactic (techniques only)
- `#attack/recon` — Reconnaissance / Discovery
- `#attack/initial-access`
- `#attack/credential-access`
- `#attack/privesc` — Privilege Escalation
- `#attack/lateral-movement`
- `#attack/persistence`
- `#attack/defense-evasion`
- `#attack/collection`
- `#attack/exfiltration`
- `#attack/impact`

## `proto/` — protocol / subsystem (optional, add as needed)
- `#proto/kerberos`
- `#proto/ntlm`
- `#proto/ldap`
- `#proto/smb`
- `#proto/dns`
- `#proto/dcom`
- `#proto/wmi`

## `status/` — note maturity
- `#status/complete`
- `#status/stub`
- `#status/unverified`

## Frontmatter schema (every note)
```yaml
---
title:                 # human title
aliases: []            # alternate names / acronyms for search + LLM
type:                  # concept | technique | tool | cheatsheet | moc | source
domain: []             # active-directory | windows-internals | red-team
attack_tactic: []      # techniques only: matches attack/ tags
tags: []               # from this index only
source:                # link to a note in 99 - Sources, e.g. "[[Source - ...]]"
source_url:            # canonical URL if applicable
verified: true         # false => also add #status/unverified
related: []            # [[links]] to sibling notes
created: 2026-06-06
updated: 2026-06-06
---
```
