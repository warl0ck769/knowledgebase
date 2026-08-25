---
title: ldapsearch
aliases: ["ldapsearch", "LDAP search tool"]
type: tool
domain: [red-team]
tags: [type/tool, domain/red-team]
source: "OpenLDAP project"
source_url: "https://www.openldap.org/software/man.cgi?query=ldapsearch"
verified: true
related: ["[[LDAP]]", "[[LDAP Enumeration]]", "[[AD Database (NTDS.dit)]]", "[[Global Catalog]]"]
created: 2026-06-07
updated: 2026-06-07
---

# ldapsearch

> [!summary] One-liner
> A command-line LDAP client (from OpenLDAP) for querying Active Directory — useful for raw LDAP enumeration from Linux when PowerShell/PowerView aren't available.

## Common usage
```bash
# Enumerate all users
ldapsearch -H ldap://<DC_IP> -D "<USER>@<DOMAIN>" -w "<PASS>" -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Find Kerberoastable accounts (users with SPNs)
ldapsearch -H ldap://<DC_IP> -D "<USER>@<DOMAIN>" -w "<PASS>" -b "DC=corp,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName

# Find AS-REP Roastable accounts
ldapsearch -H ldap://<DC_IP> -D "<USER>@<DOMAIN>" -w "<PASS>" -b "DC=corp,DC=local" "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName

# Domain controllers
ldapsearch -H ldap://<DC_IP> -D "<USER>@<DOMAIN>" -w "<PASS>" -b "DC=corp,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=8192)" dNSHostName

# Query Global Catalog (forest-wide)
ldapsearch -H ldap://<DC_IP>:3268 -D "<USER>@<DOMAIN>" -w "<PASS>" -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Anonymous bind test
ldapsearch -H ldap://<DC_IP> -x -b "DC=corp,DC=local" -s base
```

## Key flags

| Flag | Purpose |
|---|---|
| `-H` | LDAP URI (ldap:// or ldaps://) |
| `-D` | Bind DN (user@domain or full DN) |
| `-w` | Password |
| `-b` | Search base (DN) |
| `-s` | Scope: base, one, sub |
| `-x` | Simple auth (vs. SASL) |

## Related
- Protocol: [[LDAP]].
- Enumeration: [[LDAP Enumeration]].
- GUI alternative: [[ADExplorer]].
- PowerShell alternative: [[PowerView]], `Get-ADUser`.
