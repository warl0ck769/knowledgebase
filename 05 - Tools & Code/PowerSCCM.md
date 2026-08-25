---
title: PowerSCCM
aliases: [Power-SCCM, SCCM PowerShell toolkit]
type: tool
domain: [active-directory, red-team, windows-internals]
tags: [type/tool, domain/red-team, domain/windows-internals, tool/powersccm]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/defense/powersccm/
verified: true
related: ["[[PowerView]]", "[[Empire]]", "[[Credential Hunting]]", "[[User Hunting]]", "[[LSASS Dumping]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary] PowerSCCM is a PowerShell 2.0-compatible toolkit (by @harmj0y, @jaredcatkinson, @mattifestation, @enigma0x3) to connect to, query, and manipulate a Microsoft System Center Configuration Manager (SCCM) installation. SCCM's enterprise host-inventory data is treated as a dual-use intelligence source for both blue-team hunting/IR and red-team reconnaissance — leveraging existing infrastructure rather than deploying new agents.

## What it does

- Self-contained `.ps1` (PowerShell v2 compatible) that abstracts SCCM's backend so you can pull rich endpoint inventory across all enrolled hosts.
- Two connection paths to an SCCM site:
  - **SQL (database) access** — connects directly to the SCCM MSSQL DB. Supports complex queries and server-side JOINs (efficient post-processing). May need fewer privileges depending on DB permissions.
  - **WMI access** — uses the SMS WMI provider classes. Typically requires admin rights on the remote system; WQL has no JOIN equivalent so filtering is basic and results can differ from SQL.
- Session model mirrors `CimSession` cmdlets: build a session object, then pipe it to query functions.
- Verb scheme: **Get-** (raw dataset from an SCCM view/class), **Find-** (hunting meta-functions that flag suspicious patterns).
- Defensive value depends on what the SCCM **Client Settings** are configured to collect (see Mitigation/config below).

## Key commands/functions

```powershell
# --- Session setup ---
Find-SccmSiteCode -ComputerName <SCCM_SERVER>        # enumerate site code(s)
Find-LocalSccmInfo                                    # discover local SCCM config
New-SccmSession -ComputerName <SCCM_SERVER> -SiteCode <CODE> -ConnectionType [WMI|Database] -Credential <PSCred>
Get-SccmSession                                       # list registered sessions (pipe into queries)
Get-SccmSession | Get-Sccm<...>                       # typical usage pattern
```

```powershell
# --- Inventory query cmdlets (defensive enumeration) ---
Get-SccmService                  # current services        (v_GS_SERVICE)
Get-SccmServiceHistory           # historical services     (v_HS_SERVICE)
Get-SccmAutoStart                # autostart programs      (v_GS_AUTOSTART_SOFTWARE)
Get-SccmProcess                  # running processes       (v_GS_PROCESS)
Get-SccmProcessHistory           # process exec history    (v_HS_PROCESS)
Get-SccmRecentlyUsedApplication  # recently launched apps  (v_GS_CCM_RECENTLY_USED_APPS)
Get-SccmDriver                   # installed drivers       (v_GS_SYSTEM_DRIVER)
Get-SccmConsoleUsage             # console/login sessions  (v_GS_SYSTEM_CONSOLE_USER)
Get-SccmSoftwareFile             # inventoried .exe files  (v_GS_SoftwareFile)
Get-SccmBrowserHelperObject      # BHOs / browser plugins  (v_GS_BROWSER_HELPER_OBJECT)
```

```powershell
# --- Hunting meta-functions (Find-*) ---
Find-SccmRenamedCMD              # renamed command shells
Find-SccmUnusualEXE             # non-.exe-extension files launched
Find-SccmRareApplication        # infrequently-seen apps across the estate
Find-SccmPostExploitation       # known post-exploitation tool names (RecentlyUsedApps)
Find-SccmPostExploitationFile   # post-ex tools in indexed software files
Find-SccmMimikatz               # Mimikatz via RecentlyUsedApps metadata
Find-SccmMimikatzFile           # Mimikatz in the software-file inventory
```

### Key SCCM data sources tapped
`v_GS_SERVICE`/`v_HS_SERVICE`, `v_GS_AUTOSTART_SOFTWARE`, `v_GS_PROCESS`/`v_HS_PROCESS`, `v_GS_CCM_RECENTLY_USED_APPS` (CCM_RecentlyUsedApps — application launch telemetry), `v_GS_SYSTEM_DRIVER`, `v_GS_SYSTEM_CONSOLE_USER` (user/session attribution), `v_GS_SoftwareFile`, `v_GS_BROWSER_HELPER_OBJECT`, `vMDMUsersPrimaryMachines` (user-to-device mapping).

## Offensive use (red team)

- **Recon at scale:** an SCCM session is a domain-wide telemetry feed — process/app history, console-user-to-machine mapping (find where a target user logs in, akin to [[User Hunting]]), software inventory, and autostart data across every enrolled host without touching endpoints individually.
- **Lateral movement / code execution:** SCCM's native application/package deployment can push and run a payload as SYSTEM across enrolled endpoints, using legitimate, trusted infrastructure that endpoint defenses generally don't flag. [UNVERIFIED — the post references these capabilities but does not give deploy syntax in the read content.] #status/unverified

## Used in techniques

- [[User Hunting]] — console-usage / primary-machine mapping locates target logons.
- [[Credential Hunting]] — software-file and recently-used-app inventory across hosts.
- Complements PowerShell offensive tooling: [[PowerView]], [[Empire]].

## Detection / artifacts

- SQL connections to the SCCM site database from non-administrative hosts; anomalous queries against `v_GS_*` / `v_HS_*` views.
- WMI access to SMS provider classes on the site server.
- Defenders can use the same `Find-Sccm*` hunting functions proactively to surface Mimikatz, renamed cmd, and post-exploitation tooling across the estate.

## Mitigation / configuration

To get full hunting value, enable in SCCM **Client Settings** the inventory classes PowerSCCM relies on: AutoStart Software, Browser Helper Objects, Drivers (`Win32_DriverVXD`), Processes (`Win32_Process`), Recently Used Applications (`CCM_RecentlyUsedApps`), Share enumeration, System Console Usage, Software Metering (application execution), and Software Inventory for all `.exe` files. Restrict and monitor SCCM DB/WMI access; treat SCCM admin and DB read access as Tier-0 since it enables estate-wide deployment.

## Sources

- harmj0y — "PowerSCCM" — https://blog.harmj0y.net/defense/powersccm/
