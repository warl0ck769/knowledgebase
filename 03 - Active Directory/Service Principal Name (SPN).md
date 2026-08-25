---
title: Service Principal Name (SPN)
aliases: ["SPN", "ServicePrincipalName", "Host service"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory, proto/kerberos]
source: "[[Source - zer1t0 - Attacking Active Directory]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberoasting]]", "[[AD User Object]]", "[[Computer Accounts]]", "[[Kerberos]]", "[[Silver Ticket]]"]
created: 2026-06-06
updated: 2026-06-07
---

# Service Principal Name (SPN)

> [!summary] One-liner
> A string that names a service running under an account, so Kerberos clients can request a ticket for that specific service.

## What it is
A **service** = functionality offered by a computer; it is identified by a **Service Principal Name (SPN)** so Kerberos can authenticate to it.

**Format:** `service/host:port/name`
- **service** — the service class (e.g. `http`, `ldap`, `cifs`, `mssqlsvc`, `host`)
- **host** — hostname / FQDN where it runs
- **port** — optional
- **name** — optional instance name

## The `host` service
The **`host`** SPN represents the **computer account itself**. When a machine joins the domain it registers `host` SPNs mapped to its [[Computer Accounts|computer account]], enabling auth to the machine as a whole.

### HOST SPN alias behaviour
`HOST` is not just another service class — it is a **built-in alias** that maps to multiple service classes. The mapping is defined in the `sPNMappings` attribute of the Active Directory object at:

`CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration`

Having `HOST/<server>` registered effectively covers `CIFS`, `WWW`, and many other service classes without needing separate SPN entries for each. However, some **privileged operations** require an **explicit `CIFS` SPN** designation rather than relying on the HOST alias alone.

**Query the alias mappings:**
```powershell
Get-ADObject -Identity "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=<DOMAIN>,DC=<TLD>" -Properties sPNMappings
```

## Service → account mapping (the attack hook)
An SPN is an **attribute on the account** the service runs as ([[AD User Object|user]] or [[Computer Accounts|computer]]). That account's [[Kerberos Keys]] encrypt service tickets.

> [!danger]
> Because **any** domain user can request a service ticket for **any** SPN, and that ticket is encrypted with the service account's key, an attacker can request it and **crack it offline** → [[Kerberoasting]]. Knowing the service key also enables [[Silver Ticket]] forgery.

## Why a red teamer cares
SPNs reveal where services run and which accounts to target. **User accounts** with SPNs (service accounts) are prime Kerberoast targets — see [[SPN Scanning]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Services" / "Host service"
- [[Source - hackndo blog]]
