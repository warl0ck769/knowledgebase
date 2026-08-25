---
title: Linux in AD
aliases: ["Linux computers", "Linux domain join", "Kerberos on Linux", "keytab", "ccache"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[Kerberos Keys]]", "[[Pass-the-Ticket]]", "[[Credential Hunting]]", "[[SSH]]"]
created: 2026-06-06
updated: 2026-06-06
---

# Linux in AD

> [!summary] One-liner
> Linux hosts can join AD and authenticate with Kerberos; their ticket caches and keytabs are credential-theft targets just like Windows.

## Discovery & connection
- **Discovery:** LDAP computer objects, DNS, `nmap` (SSH **22**), SMB enumeration (if Samba).
- **Connection:** **SSH** (primary), SSH keys in `~/.ssh/`, **Kerberos** (`kinit` → SSH with GSSAPI), or RPC/SMB if Samba is configured.

## Kerberos on Linux (the key part)
| Item | Path / command | Meaning |
|---|---|---|
| `kinit` / `klist` / `kdestroy` | — | get / list / destroy tickets |
| **ccache** | `/tmp/krb5cc_<uid>` (or `$KRB5CCNAME`) | a user's cached tickets — **steal → [[Pass-the-Ticket]]** |
| **keytab** | `/etc/krb5.keytab` | machine-account [[Kerberos Keys|keys]] — auth **without a password** |
| config | `/etc/krb5.conf` | realm settings |
| **SSSD** / `realm` | — | daemon/tool managing AD Kerberos integration on Linux |

> A readable **keytab** or another user's **ccache** is equivalent to stealing their credentials — extract and reuse on the attacker box (set `KRB5CCNAME`).

## Credential hunting on Linux
`~/.bash_history`, `~/.ssh/` keys, `env`, `/etc/sudoers`, cron, `/var/log`, `/tmp` — see [[Credential Hunting]].

## Why a red teamer cares
Domain-joined Linux is often overlooked but holds AD Kerberos material (keytabs/ccaches) that pivots straight back into the domain.

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Linux computers" (discovery / connection / credentials)
