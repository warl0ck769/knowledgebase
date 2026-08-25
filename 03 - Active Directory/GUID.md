---
title: GUID
aliases: ["GUID", "objectGUID", "Globally Unique Identifier", "UUID"]
type: concept
domain: [active-directory]
tags: [type/concept, domain/active-directory]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows/win32/ad/using-objectguid-to-bind-to-an-object"
verified: true
related: ["[[SID]]", "[[AD Database (NTDS.dit)]]", "[[LDAP]]", "[[ACL, ACE, DACL, SACL]]"]
created: 2026-06-07
updated: 2026-06-07
---

# GUID (Globally Unique Identifier)

> [!summary] One-liner
> A 128-bit identifier assigned to every AD object at creation — unlike [[SID]]s it **never changes**, even if the object is renamed or moved, making it the most stable way to reference an AD object.

## GUID vs. SID

| Property | objectGUID | [[SID]] |
|---|---|---|
| **Assigned to** | Every AD object (users, groups, OUs, GPOs, schema classes) | Security principals only (users, groups, computers) |
| **Changes on rename/move?** | **No** | No (SID also stable, but GUID covers non-security objects too) |
| **Survives domain migration?** | No (new GUID in new domain) | No (new SID, old kept in SIDHistory) |
| **Format** | `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}` (128 bits) | `S-1-5-21-...-RID` |

## Where GUIDs appear in AD
- **`objectGUID`**: Every AD object's unique identifier.
- **Schema**: Each attribute and class has a `schemaIDGUID` — used in ACEs to grant rights on specific attributes (e.g., "write to `userAccountControl`").
- **Rights GUIDs**: Extended rights (like `Replicating Directory Changes`) are identified by GUID in ACEs — see [[ACL, ACE, DACL, SACL]].
- **GPOs**: Each [[Group Policy Object (GPO)]] has a GUID (`{...}`) as its folder name in SYSVOL.

## Red-team relevance
- **ACL Abuse**: When analyzing ACEs with tools like [[PowerView]] or [[BloodHound]], the `ObjectType` field is a GUID referencing a specific attribute or extended right. Resolving GUIDs is essential for understanding what an ACE actually grants.
- **GPO identification**: GPO folders in `\\<domain>\SYSVOL\<domain>\Policies\{GUID}\` — knowing the GUID maps to the GPO name.

```powershell
# Get an object's GUID
Get-ADUser <USER> -Properties objectGUID | Select objectGUID

# Resolve a schemaIDGUID to its attribute name
Get-ADObject -SearchBase "CN=Schema,CN=Configuration,DC=corp,DC=local" -Filter {schemaIDGUID -eq "<GUID>"} -Properties lDAPDisplayName
```

## Related
- Compared with: [[SID]] (for security principals).
- Used in ACEs: [[ACL, ACE, DACL, SACL]].
- GPO naming: [[Group Policy Object (GPO)]].
