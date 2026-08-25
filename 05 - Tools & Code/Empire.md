---
title: Empire
aliases: [PowerShell Empire, EmPyre, Empire C2]
type: tool
domain: [red-team, windows-internals, active-directory]
tags: [type/tool, domain/red-team, domain/windows-internals, tool/powershell, tool/python, proto/c2, attack/lateral-movement, attack/persistence, attack/credential-access]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/empire/empire-1-1/
verified: true
related: ["[[PowerView]]", "[[PowerUp]]", "[[GhostPack]]", "[[Rubeus]]", "[[Kerberoasting]]", "[[DCSync]]", "[[Golden Ticket]]", "[[Silver Ticket]]", "[[Pass-the-Hash]]", "[[Overpass-the-Hash]]", "[[LSASS Dumping]]", "[[GPO Abuse]]", "[[LAPS]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary] Empire is a pure-PowerShell post-exploitation C2 framework (later joined by EmPyre, its Python/OS X-Linux counterpart) built to solve the PowerShell "weaponization problem" — combining a cryptologically-secure agent, listeners/stagers, and a large module library covering recon, privesc, lateral movement, credential theft, and persistence.

## What it does
Empire is a command-and-control framework released at BSides Las Vegas 2015 by harmj0y and sixdub. The PowerShell agent runs without `powershell.exe`, communicates asynchronously over encrypted HTTP/HTTPS, and exposes a large module library. Its sibling **EmPyre** (Adaptive Threat Division) ports the same architecture to pure Python 2.7 (stdlib-only) for macOS/Linux; Empire 2.0 merged the two so PowerShell and Python agents share a single listener/port.

Core architecture:
- **Listeners** — server-side handlers that catch agent check-ins (HTTP/HTTPS; later Flask-based in 2.0).
- **Stagers** — payload generators that deliver the agent: launcher one-liner, macro, HTA, DLL, VBS, batch, Ducky, WAR (Tomcat/JBoss), PHP hop redirector; EmPyre/2.0 add AppleScript, Mach-O, dylib, Safari HTML, JAR, ELF PyInstaller, AppBundle, and `.pkg`.
- **Agents** — deployed implants; managed with check-in lifecycle (`DefaultLostLimit` self-terminate, default 60), autoruns on new check-in, and orphan restaging (2.0).
- **Modules** — categorized: `situational_awareness` (recon), `privesc`, `lateral_movement`, `credentials`, `collection`, `persistence`, `management`, `trollsploit`. PowerView 2.0, PowerUp, and Mimikatz functionality are folded in via dependency-tracing helpers that recursively pull required functions.

Crypto/comms: agent traffic is encrypted but was originally malleable; MD5-HMAC integrity was added (with dual server-side verification to prevent timing attacks). 2.0 added RC4 first-stage obfuscation (replacing XOR), staging HMAC/nonces, an RC4-metadata packet format enabling P2P routing + multi-language agents, and @mattifestation's AMSI bypass in the stage0 launcher. EmPyre uses Diffie-Hellman EKE rather than Empire's RSA key exchange.

Interfaces: interactive console, headless **CLI** (msfvenom-style stager generation without the server running), and a Flask **RESTful API** (Empire 1.5; token auth, default port 1337 over HTTPS). Note: Empire is **complementary to, not a replacement for, Meterpreter** — its value is rapid in-field agent/script modification.

## Key commands/functions
```text
# --- Interactive console ---
listeners                              # configure/list C2 listeners
usestager <stager> <listener>          # generate a payload (e.g. launcher, macro, hta, dll)
agents                                  # agent management menu
agents> list stale / remove stale      # prune dead agents
interact <agent_id>                    # control a compromised host
usemodule <module_path>                # run a post-ex module (tab-completes per agent language in 2.0)
set Agent autorun                      # auto-run a module on every new check-in
load /path/to/folder                   # load external modules at runtime

# --- Headless CLI (no running server needed) ---
./empire -h                            # help
./empire -s                            # list stagers
./empire -s <stager> -o OPT1=VAL1 OPT2=VAL2   # generate a stager
./empire -l [<listener>]               # view listener config
./empire --debug [LEVEL]               # debug output (--debug 2 = console); writes ./LastTask.ps1

# --- RESTful API (Empire 1.5+) ---
./empire --rest        # API only
./empire --headless    # full headless mode
./empire --restport <PORT>             # default 1337, HTTPS via ./data/empire.pem
# POST /api/admin/login  (default user empireadmin, random pass) -> token
# GET  /api/listeners?token=<TOKEN>
# POST /api/stagers?token=<TOKEN>      # JSON {stager, listener}

# --- EmPyre (Python / OS X / Linux) ---
./setup/install.sh ; ./empyre
usestager <stager> <listener>          # AppleScript, macro, dylib, Mach-O, etc.
# launcher pipes echoed Python to the python binary so it hides from `ps`
```

## Notable modules (folded across releases)
- **Recon** — `powerview/*` (get_forest, get_group(_member), get_gpo, get_object_acl, find_gpo_location, find_gpo_computer_admin, get_domain_sid, get_domain_policy, get_pathacl, find_foreign_user/group), `find_fruit`, `find_managed_security_groups`, `get_cached_rdpconnection`, `paranoia`, `bloodhound` (2.0).
- **Credentials** — `mimikatz/dcsync` & `dcsync_hashdump` (see [[DCSync]]), `mimikatz/golden_ticket` (with `sids` for trust hops — see [[Golden Ticket]]), `cache`/`sam` (MSCachev2/SAM hashes), `get_spn_tickets` ([[Kerberoasting]]), `mcafee_sitelist`, `ChromeDump`/`FoxDump`.
- **Privesc** — `bypassuac_wscript`, `bypassuac_eventvwr` (fileless), `ask` (RunAs), `getsystem`, `tater` (Hot Potato).
- **Lateral movement** — `invoke_psexec`, `invoke_wmi`, `invoke_psremoting`, `invoke_wmidebugger` (IFEO debugger on accessibility binaries), `inveigh_relay` (SMB relay), `new_gpo_immediate_task` (see [[GPO Abuse]]), `invoke_sshcommand`. Mimikatz `sekurlsa::pth` for [[Overpass-the-Hash]] (then token theft to beat the Kerberos double-hop).
- **Collection** — `netripper`, `packet_capture` (netsh), `inveigh`/`inveigh_bruteforce` (LLMNR/NBNS), `mailraider/*` (Outlook phishing).
- **Persistence** — userland/elevated `Run`-key & scheduled-task & WMI-subscription triggers (payload in RegPath/ADS/EventLogID), `backdoor_lnk`, `add_netuser`, `install_ssp`, Skeleton Key, `disable_machine_acct_change` (durable [[Silver Ticket]]), PowerBreach memory-only triggers (Eventlog/Resolver/Deaduser).

## Used in techniques
- [[Kerberoasting]] — `get_spn_tickets`
- [[DCSync]] / [[NTDS.dit Extraction]] — `mimikatz/dcsync`, `dcsync_hashdump`
- [[Golden Ticket]] / [[Silver Ticket]] — `golden_ticket`, `disable_machine_acct_change`
- [[Pass-the-Hash]] / [[Overpass-the-Hash]] — `mimikatz/pth`
- [[GPO Abuse]] — `new_gpo_immediate_task`, PowerView GPO modules
- Persistence — registry/schtask/WMI/PowerBreach modules
- Recon / [[LDAP Enumeration]] — embedded [[PowerView]]; privesc enum via [[PowerUp]]; C# successors in [[GhostPack]] / [[Rubeus]]

## Sources
- harmj0y, "Empire 1.1" — https://blog.harmj0y.net/empire/empire-1-1/
- harmj0y, "Empire 1.2" — https://blog.harmj0y.net/empire/empire-1-2/
- harmj0y, "Empire 1.3" — https://blog.harmj0y.net/empire/empire-1-3/
- harmj0y, "Empire 1.4" — https://blog.harmj0y.net/empire/empire-1-4/
- harmj0y, "Empire 1.5" — https://blog.harmj0y.net/empire/empire-1-5/
- harmj0y, "Expanding Your Empire" (lateral movement) — https://blog.harmj0y.net/empire/expanding-your-empire/
- harmj0y, "Empire's CLI" — https://blog.harmj0y.net/empire/empires-cli/
- harmj0y, "Empire's RESTful API" — https://blog.harmj0y.net/empire/empires-restful-api/
- harmj0y, "The Empire Strikes Back" (2.0) — https://blog.harmj0y.net/empire/the-empire-strikes-back/
- harmj0y, "Nothing Lasts Forever: Persistence with Empire" — https://blog.harmj0y.net/empire/nothing-lasts-forever-persistence-with-empire/
- harmj0y, "Empire Fails" (security fixes) — https://blog.harmj0y.net/empire/empire-fails/
- harmj0y, "Building an EmPyre with Python" — https://blog.harmj0y.net/empyre/building-an-empyre-with-python/
- harmj0y, "OS X Office Macros with EmPyre" — https://blog.harmj0y.net/empyre/os-x-office-macros-with-empyre/
- harmj0y, "Empire, Meterpreter, and Offensive Half-Life" — https://blog.harmj0y.net/informational/empire-meterpreter-and-offensive-half-life/
