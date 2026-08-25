---
title: Organizational Unit (OU)
aliases: ["OU", "organizational unit"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/creating-an-organizational-unit-design"
verified: true
related: ["[[Domain]]", "[[Group Policy Object (GPO)]]", "[[ACL, ACE, DACL, SACL]]", "[[AD Groups]]", "[[LDAP]]"]
created: 2026-06-07
updated: 2026-06-07
---

# Organizational Unit (OU)

> [!summary] One-liner
> A container within a [[Domain]] used to organize AD objects (users, computers, groups) into a hierarchy — the primary target for [[Group Policy Object (GPO)|GPO]] linkage and delegation of administrative control.

## Key properties
- **Hierarchical**: OUs can be nested (e.g., `OU=IT,OU=Departments,DC=corp,DC=local`).
- **Not a security boundary**: Unlike domains, OUs don't isolate security — an OU admin with the right delegation can affect objects in child OUs.
- **GPO linkage**: GPOs can be linked to a site, domain, or **OU** — OU-level is the most common and granular.
- **Delegation**: Admins can grant specific permissions on an OU via its [[ACL, ACE, DACL, SACL|DACL]] (e.g., "Help Desk can reset passwords for users in OU=HQ").

## OU vs. Container vs. Group

| Object | Purpose | GPO linkable? | Security principal? |
|---|---|---|---|
| **OU** | Organize + delegate + apply policy | Yes | No |
| **Container** (e.g., `CN=Users`) | Default AD containers | **No** | No |
| **[[AD Groups|Group]]** | Grant permissions | No (policies apply to OU, not group) | Yes (security groups) |

## GPO processing order (LSDOU)
1. **L**ocal policy
2. **S**ite-linked GPOs
3. **D**omain-linked GPOs
4. **OU**-linked GPOs (parent → child, most specific wins)

Last-applied wins for conflicting settings. `Enforced` on a higher GPO overrides child OU settings. `Block Inheritance` on an OU ignores parent GPOs (unless they're enforced).

## Red-team relevance
- **GPO abuse**: If you can modify a GPO linked to an OU with privileged users/computers, you can push code execution to those targets — see [[GPO Abuse]].
- **OU delegation abuse**: Weak DACLs on an OU can grant `GenericAll` or `WriteDACL` over all objects in it — see [[ACL Abuse]].
- **Enumeration**: OU structure reveals organizational layout, admin tiers, and where high-value targets sit.

```powershell
# Enumerate OUs
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName

# OUs with linked GPOs
Get-GPInheritance -Target "OU=Servers,DC=corp,DC=local"

# OU permissions (PowerView)
Get-DomainObjectAcl -Identity "OU=Admins,DC=corp,DC=local" -ResolveGUIDs
```

## Related
- GPO application: [[Group Policy Object (GPO)]].
- ACL delegation: [[ACL, ACE, DACL, SACL]].
- Container context: [[Domain]], [[LDAP]].
