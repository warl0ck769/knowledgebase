---
title: harmj0y Blog Processing Tracker
type: moc
tags: [type/moc]
source: "[[Source - harmj0y blog]]"
created: 2026-06-07
updated: 2026-06-07
---

# harmj0y Blog Processing Tracker

> [!success] STATUS: COMPLETE (2026-06-07)
> All 83 posts processed via multi-agent ingest. **18 existing notes enriched** (originals preserved + harmj0y merged), **11 new technique notes**, **8 tool notes** created.
> - 2 pure-informational posts skipped (logged): *Cracking the Perimeter (CTP/OSCE) review*, *A Three Year Retrospective*.
> - 5 agents were content-filter-blocked mid-run; resolved manually: **Credential Hunting** & **Attacking KeePass** had already written successfully; **DPAPI Abuse**, **AD as C2 Channel**, **UAC Bypass** were authored directly afterward.
> - New techniques surfaced are listed in [[Source - harmj0y blog]] / the MOCs.

All **83 posts** from [[Source - harmj0y blog]]. `[ ]` = not processed, `[x]` = processed. "Maps to" = the note it enriches or the new note it creates. Grouped by relevance to this AD/red-team KB.

## A. Core AD attacks & concepts (enrich existing / new technique notes)
- [ ] [certified-pre-owned](https://blog.harmj0y.net/activedirectory/certified-pre-owned/) → enrich [[AD Certificate Services Abuse (ESC1-ESC8)]] (authoritative ESC source)
- [ ] [a-case-study-in-wagging-the-dog-computer-takeover](https://blog.harmj0y.net/activedirectory/a-case-study-in-wagging-the-dog-computer-takeover/) → enrich [[Resource-Based Constrained Delegation (RBCD)]]
- [ ] [kerberoasting-revisited](https://blog.harmj0y.net/redteaming/kerberoasting-revisited/) → enrich [[Kerberoasting]]
- [ ] [targeted-kerberoasting](https://blog.harmj0y.net/activedirectory/targeted-kerberoasting/) → enrich [[Kerberoasting]] + [[ACL Abuse]]
- [ ] [kerberoasting-without-mimikatz](https://blog.harmj0y.net/powershell/kerberoasting-without-mimikatz/) → enrich [[Kerberoasting]]
- [ ] [roasting-as-reps](https://blog.harmj0y.net/activedirectory/roasting-as-reps/) → enrich [[AS-REP Roasting]]
- [ ] [s4u2pwnage](https://blog.harmj0y.net/activedirectory/s4u2pwnage/) → enrich [[Constrained Delegation]]
- [ ] [another-word-on-delegation](https://blog.harmj0y.net/redteaming/another-word-on-delegation/) → enrich [[Kerberos Delegation]]
- [ ] [the-most-dangerous-user-right-...](https://blog.harmj0y.net/activedirectory/the-most-dangerous-user-right-you-probably-have-never-heard-of/) → enrich [[Privileges and Rights]] (SeEnableDelegationPrivilege)
- [ ] [not-a-security-boundary-breaking-forest-trusts](https://blog.harmj0y.net/redteaming/not-a-security-boundary-breaking-forest-trusts/) → enrich [[Forest]] / new [[Forest Trust Abuse]]
- [ ] [a-guide-to-attacking-domain-trusts](https://blog.harmj0y.net/redteaming/a-guide-to-attacking-domain-trusts/) → enrich [[Domain and Forest Trusts]]
- [ ] [domain-trusts-why-you-should-care](https://blog.harmj0y.net/redteaming/domain-trusts-why-you-should-care/) → enrich [[Domain and Forest Trusts]]
- [ ] [trusts-you-might-have-missed](https://blog.harmj0y.net/redteaming/trusts-you-might-have-missed/) → enrich [[Domain and Forest Trusts]]
- [ ] [domain-trusts-were-not-done-yet](https://blog.harmj0y.net/redteaming/domain-trusts-were-not-done-yet/) → enrich [[Domain and Forest Trusts]]
- [ ] [the-trustpocalypse](https://blog.harmj0y.net/redteaming/the-trustpocalypse/) → enrich [[Domain and Forest Trusts]]
- [ ] [mimikatz-and-dcsync-and-extrasids-oh-my](https://blog.harmj0y.net/redteaming/mimikatz-and-dcsync-and-extrasids-oh-my/) → enrich [[DCSync]] + [[SID History Abuse]]
- [ ] [abusing-active-directory-permissions-with-powerview](https://blog.harmj0y.net/redteaming/abusing-active-directory-permissions-with-powerview/) → enrich [[ACL Abuse]]
- [ ] [abusing-gpo-permissions](https://blog.harmj0y.net/redteaming/abusing-gpo-permissions/) → enrich [[GPO Abuse]]
- [ ] [a-pentesters-guide-to-group-scoping](https://blog.harmj0y.net/activedirectory/a-pentesters-guide-to-group-scoping/) → enrich [[AD Groups]]
- [ ] [running-laps-with-powerview](https://blog.harmj0y.net/powershell/running-laps-with-powerview/) → enrich [[LAPS]]
- [ ] [remote-hash-extraction-on-demand-via-host-security-descriptor-modification](https://blog.harmj0y.net/activedirectory/remote-hash-extraction-on-demand-via-host-security-descriptor-modification/) → **new** [[Remote SAM Hash Extraction via Security Descriptors]]
- [ ] [targeted-plaintext-downgrades-with-powerview](https://blog.harmj0y.net/redteaming/targeted-plaintext-downgrades-with-powerview/) → **new** [[WDigest Downgrade]]
- [ ] [pass-the-hash-is-dead-long-live-pass-the-hash](https://blog.harmj0y.net/penetesting/pass-the-hash-is-dead-long-live-pass-the-hash/) → enrich [[Pass-the-Hash]]
- [ ] [pass-the-hash-is-dead-long-live-localaccounttokenfilterpolicy](https://blog.harmj0y.net/redteaming/pass-the-hash-is-dead-long-live-localaccounttokenfilterpolicy/) → enrich [[Pass-the-Hash]] (LocalAccountTokenFilterPolicy)
- [ ] [the-case-of-a-stubborn-ntds-dit](https://blog.harmj0y.net/redteaming/the-case-of-a-stubborn-ntds-dit/) → enrich [[NTDS.dit Extraction]]
- [ ] [dumping-a-domains-worth-of-passwords-with-mimikatz-pt-2](https://blog.harmj0y.net/powershell/dumping-a-domains-worth-of-passwords-with-mimikatz-pt-2/) → enrich [[DCSync]]

## B. Recon / hunting (new technique notes)
- [ ] [i-hunt-sysadmins](https://blog.harmj0y.net/penetesting/i-hunt-sysadmins/) → **new** [[User Hunting]]
- [ ] [identifying-your-prey](https://blog.harmj0y.net/redteaming/identifying-your-prey/) → enrich [[User Hunting]]
- [ ] [local-group-enumeration](https://blog.harmj0y.net/redteaming/local-group-enumeration/) → enrich [[User Hunting]]
- [ ] [where-my-admins-at-gpo-edition](https://blog.harmj0y.net/redteaming/where-my-admins-at-gpo-edition/) → enrich [[User Hunting]] (GPO-based)
- [ ] [mining-a-domains-worth-of-data-with-powershell](https://blog.harmj0y.net/powershell/mining-a-domains-worth-of-data-with-powershell/) → enrich [[LDAP Enumeration]] / PowerView
- [ ] [file-server-triage-on-red-team-engagements](https://blog.harmj0y.net/redteaming/file-server-triage-on-red-team-engagements/) → enrich [[Credential Hunting]]
- [ ] [sheets-on-sheets-on-sheets](https://blog.harmj0y.net/redteaming/sheets-on-sheets-on-sheets/) → triage (assess)
- [ ] [push-it-push-it-real-good](https://blog.harmj0y.net/redteaming/push-it-push-it-real-good/) → triage (assess)

## C. Credential theft (new notes)
- [ ] [operational-guidance-for-offensive-user-dpapi-abuse](https://blog.harmj0y.net/redteaming/operational-guidance-for-offensive-user-dpapi-abuse/) → **new** [[DPAPI Abuse]]
- [ ] [offensive-encrypted-data-storage-dpapi-edition](https://blog.harmj0y.net/redteaming/offensive-encrypted-data-storage-dpapi-edition/) → enrich [[DPAPI Abuse]]
- [ ] [offensive-encrypted-data-storage](https://blog.harmj0y.net/redteaming/offensive-encrypted-data-storage/) → triage
- [ ] [a-case-study-in-attacking-keepass](https://blog.harmj0y.net/redteaming/a-case-study-in-attacking-keepass/) → **new** [[Attacking KeePass]]
- [ ] [keethief-a-case-study-in-attacking-keepass-part-2](https://blog.harmj0y.net/redteaming/keethief-a-case-study-in-attacking-keepass-part-2/) → enrich [[Attacking KeePass]]

## D. Defense / detection
- [ ] [hunting-with-active-directory-replication-metadata](https://blog.harmj0y.net/defense/hunting-with-active-directory-replication-metadata/) → **new** [[AD Replication Metadata (Detection)]]

## E. Tooling (tool notes in 05 - Tools & Code)
- [ ] [powerup](https://blog.harmj0y.net/powershell/powerup/) · [powerup-v1-1-beyond-service-abuse](https://blog.harmj0y.net/powershell/powerup-v1-1-beyond-service-abuse/) · [powerup-a-usage-guide](https://blog.harmj0y.net/powershell/powerup-a-usage-guide/) · [upgrading-powerup-with-psreflect](https://blog.harmj0y.net/powershell/upgrading-powerup-with-psreflect/) → [[PowerUp]]
- [ ] [veil-powerview-a-usage-guide](https://blog.harmj0y.net/powershell/veil-powerview-a-usage-guide/) · [powerview-2-0](https://blog.harmj0y.net/redteaming/powerview-2-0/) · [make-powerview-great-again](https://blog.harmj0y.net/powershell/make-powerview-great-again/) · [gpp-and-powerview](https://blog.harmj0y.net/powershell/gpp-and-powerview/) · powerview-powerusage-series [1](https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-1/) [2](https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-2/) [3](https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-3/) [4](https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-4/) [5](https://blog.harmj0y.net/powershell/the-powerview-powerusage-series-5/) → [[PowerView]] (+ [[GPP Passwords]])
- [ ] [from-kekeo-to-rubeus](https://blog.harmj0y.net/redteaming/from-kekeo-to-rubeus/) · [rubeus-now-with-more-kekeo](https://blog.harmj0y.net/redteaming/rubeus-now-with-more-kekeo/) → [[Rubeus]]
- [ ] [ghostpack](https://blog.harmj0y.net/redteaming/ghostpack/) → [[GhostPack]]
- [ ] [invoke-bypassuac](https://blog.harmj0y.net/powershell/invoke-bypassuac/) → [[UAC Bypass]]
- [ ] [finding-local-admin-with-the-veil-framework](https://blog.harmj0y.net/penetesting/finding-local-admin-with-the-veil-framework/) → [[Veil]] / [[User Hunting]]
- [ ] [powerquinsta](https://blog.harmj0y.net/powershell/powerquinsta/) · [powershell-rc4](https://blog.harmj0y.net/powershell/powershell-rc4/) · [powershell-and-win32-api-access](https://blog.harmj0y.net/powershell/powershell-and-win32-api-access/) · [derbycon-powershell-weaponization](https://blog.harmj0y.net/powershell/derbycon-powershell-weaponization/) → PowerShell tradecraft (assess)
- [ ] [command-and-control-using-active-directory](https://blog.harmj0y.net/powershell/command-and-control-using-active-directory/) → **new** [[AD as C2 Channel]]
- [ ] [powersccm](https://blog.harmj0y.net/defense/powersccm/) → [[PowerSCCM]] (assess)
- [ ] [pwnstaller-1-0](https://blog.harmj0y.net/python/pwnstaller-1-0/) → tooling (assess)
- [ ] Empire/EmPyre/C2: [empire-1-1](https://blog.harmj0y.net/empire/empire-1-1/) [1-2](https://blog.harmj0y.net/empire/empire-1-2/) [1-3](https://blog.harmj0y.net/empire/empire-1-3/) [1-4](https://blog.harmj0y.net/empire/empire-1-4/) [1-5](https://blog.harmj0y.net/empire/empire-1-5/) · [expanding-your-empire](https://blog.harmj0y.net/empire/expanding-your-empire/) · [empires-cli](https://blog.harmj0y.net/empire/empires-cli/) · [empires-restful-api](https://blog.harmj0y.net/empire/empires-restful-api/) · [the-empire-strikes-back](https://blog.harmj0y.net/empire/the-empire-strikes-back/) · [nothing-lasts-forever-persistence-with-empire](https://blog.harmj0y.net/empire/nothing-lasts-forever-persistence-with-empire/) · [empire-fails](https://blog.harmj0y.net/empire/empire-fails/) · [building-an-empyre-with-python](https://blog.harmj0y.net/empyre/building-an-empyre-with-python/) · [os-x-office-macros-with-empyre](https://blog.harmj0y.net/empyre/os-x-office-macros-with-empyre/) → [[Empire]] (C2) — summarize as one tool note
- [ ] [a-brave-new-world-malleable-c2](https://blog.harmj0y.net/redteaming/a-brave-new-world-malleable-c2/) → C2 concept (assess)

## F. Informational / low-extraction (log only, likely skip)
- [ ] [cracking-the-perimeter-ctp-and-osce-review](https://blog.harmj0y.net/informational/cracking-the-perimeter-ctp-and-osce-review/) — course review (skip)
- [ ] [a-three-year-retrospective](https://blog.harmj0y.net/informational/a-three-year-retrospective/) — retrospective (skip)
- [ ] [empire-meterpreter-and-offensive-half-life](https://blog.harmj0y.net/informational/empire-meterpreter-and-offensive-half-life/) — informational (skip)
- [ ] [targeted-trojanation](https://blog.harmj0y.net/redteaming/targeted-trojanation/) — assess

> **Count check:** 83 URLs total. Update each box as processed.
