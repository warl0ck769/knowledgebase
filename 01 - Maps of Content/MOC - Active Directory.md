---
title: MOC - Active Directory
type: moc
domain: [active-directory]
tags: [type/moc, domain/active-directory]
created: 2026-06-06
updated: 2026-06-07
---

# MOC — Active Directory

> [!abstract] Scope
> Everything an attacker needs to understand AD: how it is built, how it authenticates, and the famous issues that break it. ✅ = note written.

## 1. Fundamentals (structure)
- [[What is Active Directory]] ✅
- [[Domain]] ✅ · [[Tree]] ✅ · [[Forest]] ✅ · [[Functional Levels]] ✅ · [[Organizational Unit (OU)]] ✅
- [[Domain Controller (DC)]] ✅ · [[AD Database (NTDS.dit)]] ✅ · [[Global Catalog]] ✅ · [[FSMO Roles]] ✅
- [[Domain and Forest Trusts]] ✅ · [[Trust Direction and Transitivity]] ✅
- [[Sites and Subnets]] ✅ · [[AD Replication]] ✅
- [[Linux in AD]] ✅

## 2. The directory data model
- [[AD Database]] ✅ (classes, properties, DN, partitions, GC) · [[LDAP]] ✅ (incl. ADWS)
- [[Security Principal]] ✅ · [[SID]] ✅ · [[RID]] ✅ · [[Well-known SIDs]] ✅ · [[GUID]] ✅
- [[Service Principal Name (SPN)]] ✅

## 3. Objects, identity & secrets
- **Users & accounts:** [[AD User Object]] ✅ · [[UserAccountControl]] ✅ · [[Important Users]] ✅ · [[krbtgt account]] ✅ · [[Computer Accounts]] ✅ · [[Trust Accounts]] ✅
- **Groups:** [[AD Groups]] ✅ · [[Privileged AD Groups]] ✅ · [[Protected Users Group]] ✅
- **Secrets:** [[LM and NT Hashes]] ✅ · [[Kerberos Keys]] ✅

## 4. Authentication
- **NTLM:** [[NTLM Authentication]] ✅ · [[NetNTLM]] ✅ (Net-NTLM v1/v2 vs NT hash) · [[SSPI and SSPs]] ✅ · [[SPNEGO]] ✅
- **Kerberos:** [[Kerberos]] ✅ · [[KDC]] ✅ · [[TGT vs TGS]] ✅ · [[Kerberos Authentication Flow]] ✅ · [[PAC]] ✅
- **Delegation:** [[Kerberos Delegation]] ✅ · [[Unconstrained Delegation]] ✅ · [[Constrained Delegation]] ✅ · [[Resource-Based Constrained Delegation (RBCD)]] ✅
- **Cached:** [[Domain Cached Credentials (DCC2)]] ✅

## 5. Authorization & policy
- [[ACL, ACE, DACL, SACL]] ✅ · [[AdminSDHolder]] ✅ · [[Privileges and Rights]] ✅ · [[Securable Objects]] ✅
- [[Group Policy Object (GPO)]] ✅ (incl. SYSVOL/GPC/GPT)

## 6. Name resolution & comms
- [[AD DNS]] ✅ (incl. ADIDNS) · [[NetBIOS]] ✅ · [[Communication Protocols]] ✅ (SMB/RPC/WinRM/RDP/SSH)

## 7. Famous issues & techniques
→ Full notes in [[MOC - Attacks & Techniques]].
