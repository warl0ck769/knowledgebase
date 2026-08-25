# README — How this vault works

This is a red-team **Active Directory + Windows internals** knowledge base. It is built to be (1) *correct*, (2) *fully interlinked*, and (3) *machine-readable*, so it can later be used as a training corpus for a custom LLM.

## Hard rules (never broken)
1. **Nothing skipped, nothing assumed.** Every concept in a source gets a note. Ambiguity is resolved by asking, never guessing.
2. **Every note has a source.** The `source` frontmatter field is mandatory. Anything not anchored to a solid source is tagged `#status/unverified` and marked inline with `[UNVERIFIED]`.

## Folder layout
| Folder | Holds |
|---|---|
| `00 - Meta` | This README, the [[Tag Index]], and `Templates/`. |
| `01 - Maps of Content` | The spine. Start at [[00 - Home]]. MOCs are index notes that link everything else. |
| `02 - Windows Internals` | Concept notes: processes, tokens, LSASS, SAM, memory, registry, services, etc. |
| `03 - Active Directory` | Concept notes: domains, forests, trusts, Kerberos, NTLM, LDAP, GPO, schema, replication, etc. |
| `04 - Attacks & Techniques` | Technique notes: each links to the concept it abuses + the tools/commands to run it. |
| `05 - Tools & Code` | Tool notes (BloodHound, Rubeus, Mimikatz, Impacket, PowerView…) and code snippets. |
| `06 - Cheat Sheets` | Condensed, interview/engagement-ready summaries that link back to concept + technique notes. |
| `99 - Sources` | One note per book/article/blog = the bibliography. Every note's `source` points here. |

## Note types (frontmatter `type:`)
- `concept` — a thing that exists (how AD/Windows works). Lives in `02`/`03`.
- `technique` — an attacker action. Lives in `04`. Always includes **Commands** + **Tools**.
- `tool` — a piece of software. Lives in `05`.
- `cheatsheet` — condensed reference. Lives in `06`.
- `moc` — map of content (index). Lives in `01`.
- `source` — a bibliography entry. Lives in `99`.

## Granularity rule (hybrid)
- **Big topic** (Kerberos, tokens, GPO) → split into atomic notes (one idea each) + a small MOC.
- **Small topic** → one self-contained note.

## Linking rules
- Link **liberally** with `[[wiki-links]]`. A link to a not-yet-written note is fine — it's a TODO marker.
- Every **technique** links to: the **concept** it abuses, the **tool(s)** used, and any **prerequisite/follow-on** techniques.
- Every note appears in at least one **MOC**.

## Status tags
- `#status/complete` — verified and done.
- `#status/stub` — placeholder, needs filling.
- `#status/unverified` — content not yet anchored to a confirmed source.

See [[Tag Index]] for the full taxonomy.
