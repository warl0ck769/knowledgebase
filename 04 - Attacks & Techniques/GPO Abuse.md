---
title: GPO Abuse
aliases: ["GPO abuse", "SharpGPOAbuse", "Group Policy attack"]
type: technique
domain: [red-team]
attack_tactic: [privesc, lateral-movement, persistence]
tags: [type/technique, domain/red-team, attack/privesc]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - harmj0y blog]]", "[[Source - hackndo blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Group Policy Object (GPO)]]", "[[ACL Abuse]]", "[[Privileged AD Groups]]", "[[Organizational Unit (OU)]]", "[[PowerView]]"]
created: 2026-06-06
updated: 2026-06-07
---

# GPO Abuse

> [!summary] One-liner
> With write access to a GPO, push a malicious policy (script, scheduled task, local-admin add) to every computer/user the GPO is linked to.

## Concept abused
A [[Group Policy Object (GPO)]] applies to all objects under its linked scope. **Write permission on the GPO** (often via [[ACL Abuse]] or membership in **Group Policy Creator Owners**) = code execution across that scope.

## Prerequisites
- Write rights to a GPO that is **linked** to an OU/domain/site containing targets.

## How the attack works
Modify the GPO to do one of:
- Add an **immediate scheduled task** running your payload.
- Add your account to the **local Administrators** group on linked machines.
- Deploy a **startup/logon script**.

## Commands & tools
```powershell
# SharpGPOAbuse - add a user to local admins on all linked computers
SharpGPOAbuse.exe --AddLocalAdmin --UserAccount eviluser --GPOName "Vulnerable GPO"

# or add an immediate scheduled task (command execution)
SharpGPOAbuse.exe --AddComputerTask --TaskName "update" --Author NT\System \
  --Command "cmd.exe" --Arguments "/c net user ..." --GPOName "Vulnerable GPO"
```
(PowerView's `Get-DomainGPO`/`Get-DomainGPOUserLocalGroupMapping` helps find writable, well-linked GPOs.)

## Detection / artifacts
GPO version bumps, changes to GPT files in SYSVOL, new scheduled tasks/startup scripts, 5136 on GPC.

## Mitigation
Restrict GPO edit rights, monitor SYSVOL/GPC changes, limit scope of high-impact GPOs.

## More from harmj0y (Abusing GPO Permissions)

### How GPOs are stored and applied
A GPO has two parts:
- **Group Policy Container (GPC)** — the AD object stored under `<domain> > System > Policies`, identified by a unique GUID. This object controls who has creation and modification rights over the GPO. The GPC's **`gPCFileSysPath`** attribute points to where the GPO's configuration files live on disk.
- **Group Policy Template (GPT)** — the actual policy files stored on every DC at `\\<DC>\SYSVOL\<domain>\Policies\<GUID>\`. All authenticated users can read this share by default.

An OU/site/domain links a GPO through its **`gPLink`** attribute, which holds the GPO GUID references and indicates exactly where the policy applies.

By default, **computer Group Policy refreshes in the background every 90 minutes with a random offset of 0-30 minutes**, so a pushed policy lands across all affected machines without manual intervention (relevant for timing both attack execution and cleanup).

### Enumerating GPO permissions with [[PowerView]]
- **`Get-NetGPO`** — enumerate all GPOs in the domain. Use `-ComputerName` to find which GPOs are applied to a specific machine.
- **`Get-ObjectAcl`** — examine the ACL on a GPO object to find who has modify rights. Combine to dump ACLs for every GPO:
```powershell
# resolve GUIDs and dump the ACL of every GPO in the domain
Get-NetGPO | %{ Get-ObjectAcl -ResolveGUIDs -Name $_.Name }
```
- **`Invoke-ACLScanner`** — automated ACL search across domain objects, surfacing entries where the `IdentityReference` RID is >= -1000 (i.e. non-default / interesting principals) that hold modification rights. Use this to spot a GPO you can edit.

Workflow: find a GPO you can write to (via `Get-ObjectAcl` / `Invoke-ACLScanner`), confirm its `gPLink`/scope with `Get-NetGPO -ComputerName <target>` so you know which machines it hits, then push a payload.

### Command execution via New-GPOImmediateTask
PowerView's **`New-GPOImmediateTask`** writes an Immediate Scheduled Task into a GPO so the next policy refresh runs your command on every machine in scope. The task XML is written to `<GPO_PATH>\Machine\Preferences\ScheduledTasks\ScheduledTasks.xml`.

Parameters:
- `-TaskName` — required task identifier.
- `-Command` — binary to execute (defaults to `powershell.exe`).
- `-CommandArguments` — arguments for the command.
- `-GPOname` / `-GPODisplayName` — target GPO.
- `-Force` — suppress confirmation prompts.
- `-Remove` — delete the task XML afterward (cleanup).

```powershell
# push an Immediate Scheduled Task running an encoded PowerShell payload to all machines under SecurePolicy
New-GPOImmediateTask -TaskName Debugging -GPODisplayName SecurePolicy `
  -CommandArguments '-NoP -NonI -W Hidden -Enc <BASE64_PAYLOAD>' -Force
```

> [!warning] OPSEC / reliability
> harmj0y notes `New-GPOImmediateTask` was eventually **removed from PowerView because of its inconsistencies** and worked mainly as a proof-of-concept. For reliable exploitation, **manually craft the `ScheduledTasks.xml` in SYSVOL** under the GPO's `Machine\Preferences\ScheduledTasks\` path (and bump the GPO version so the policy reapplies). Use `-Remove` (or delete the XML manually) to clean up the task after execution.

### Other GPO-based attack vectors (per harmj0y)
Beyond immediate scheduled tasks, edit rights on a GPO let you:
- Deploy **startup scripts**.
- **Backdoor Internet Explorer** settings.
- **Software installation** via `.MSI` deployment.
- Add accounts to **local administrator / RDP group membership** on linked hosts.
- Mount a **network share** to coerce/relay credentials.

## EditSettings permission gap (BloodHound blind spot)

GPO delegation offers three permission levels:
1. **Edit Settings** — can modify GPO content (policy settings, scheduled tasks, scripts).
2. **Delete** — can delete the GPO.
3. **Edit Settings + Delete + Modify Security** — full control, includes **WriteDacl** over the GPO.

> [!danger] BloodHound blind spot
> BloodHound only tracks the third level (WriteDacl / full control) when building GPO attack paths. A principal with **EditSettings only** can fully modify GPO content (push scheduled tasks, scripts, local admin changes) but will **not appear in BloodHound's attack graph**. This makes EditSettings-only permissions an invisible attack path during standard enumeration with BloodHound/SharpHound.

### Exploiting EditSettings via Immediate Scheduled Tasks (User Configuration)

With EditSettings, an attacker can create an **Immediate Scheduled Task** under **User Configuration** (rather than Computer Configuration). When the GPO applies to a user session on a target machine, the task executes **as the currently logged-in user**. If a **Domain Admin** is logged into a machine in the GPO's scope, the scheduled task runs as that DA — providing immediate privilege escalation without needing direct access to the DA account.

This makes EditSettings especially dangerous on GPOs linked to OUs containing privileged user accounts or workstations where privileged users routinely log in.

## Related
- [[Group Policy Object (GPO)]]
- [[ACL Abuse]]
- [[PowerView]]
- [[Privileged AD Groups]]
- [[Organizational Unit (OU)]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Group Policy" (GPO abuse)
- [[Source - harmj0y blog]] — "Abusing GPO Permissions" — https://blog.harmj0y.net/redteaming/abusing-gpo-permissions/
- [[Source - hackndo blog]] — "GPO Abuse with Edit Settings" — https://en.hackndo.com/gpo-abuse-with-edit-settings/
