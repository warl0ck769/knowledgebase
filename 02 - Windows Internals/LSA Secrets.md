---
title: LSA Secrets
aliases: ["LSA Secrets", "$MACHINE.ACC", "DefaultPassword autologon", "_SC_ service secrets"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Credential Storage in Windows]]", "[[SAM Database]]", "[[SAM and LSA Secrets Dump]]", "[[Domain Cached Credentials (DCC2)]]", "[[DPAPI]]"]
created: 2026-06-06
updated: 2026-06-06
---

# LSA Secrets

> [!summary] One-liner
> A protected registry store (`HKLM\SECURITY\Policy\Secrets`) holding the machine account password, service passwords, autologon creds, and DPAPI keys.

## What it is
**LSA Secrets** live at **`HKLM\SECURITY\Policy\Secrets`**, encrypted with the **BootKey/SysKey** (from the SYSTEM hive). Contents include:

| Secret | What it is |
|---|---|
| **`$MACHINE.ACC`** | The **computer's domain account password** (hex blob + NT hash) — auth as the machine account |
| **`_SC_<service>`** | A **service account's password** (cross-reference the service via WMI to get the username) |
| **`DefaultPassword`** | **Auto-logon** credential (also see `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`: `DefaultUserName`, `DefaultDomainName`) |
| **`DPAPI_SYSTEM`** | System **[[DPAPI]]** master key — decrypts user-protected data |
| **Domain cached creds** | MSCACHEv2 / **[[Domain Cached Credentials (DCC2)]]** from recent domain logons |

## Why a red teamer cares
- `$MACHINE.ACC` → authenticate as the computer ([[Silver Ticket]] / [[Resource-Based Constrained Delegation (RBCD)|RBCD]] leverage).
- `_SC_*` service passwords are frequently **domain accounts**, sometimes privileged → instant lateral movement / privesc.
- `DPAPI_SYSTEM` unlocks DPAPI-protected secrets (browser creds, Credential Manager).

Dumped via [[SAM and LSA Secrets Dump]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Registry credentials → LSA secrets"
