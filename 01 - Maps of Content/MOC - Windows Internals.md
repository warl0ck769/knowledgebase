---
title: MOC - Windows Internals
type: moc
domain: [windows-internals]
tags: [type/moc, domain/windows-internals]
created: 2026-06-06
updated: 2026-06-07
---

# MOC — Windows Internals

> [!abstract] Scope
> The OS mechanisms AD attacks stand on: security tokens, authentication subsystem, credential storage, privileges, and core OS concepts.

## Security model
- [[Access Token]] ✅ · [[SID]] ✅ · [[Integrity Levels]] ✅ · [[Securable Objects]] ✅
- [[Privileges and Rights]] ✅ · [[Logon Types]] ✅ (credential caching per logon type)
- [[UAC]] ✅ (User Account Control — token splitting, auto-elevation)

## Process model
- [[Process and Thread]] ✅ (primary token, impersonation token, injection)

## Authentication subsystem
- [[SSPI and SSPs]] ✅ · [[Authentication Packages]] ✅ (MSV1_0, Kerberos AP, WDigest, CredSSP)
- [[SPNEGO]] ✅ · [[LSASS]] ✅

## Credential storage
- [[Credential Storage in Windows]] ✅ (overview) · [[SAM Database]] ✅ · [[LSA Secrets]] ✅
- [[DPAPI]] ✅ (Data Protection API — masterkeys, domain backup key)
- [[Domain Cached Credentials (DCC2)]] ✅

## Networking & protocols
- [[SMB]] ✅ (Server Message Block — port 445, signing, shares)
- [[SSH]] ✅ (OpenSSH on Windows, Kerberos GSSAPI)

## System components
- [[Windows Registry]] ✅ (hives, security-critical paths, credential stores)
- [[Windows Services]] ✅ (SCM, service accounts, privilege escalation vectors)
