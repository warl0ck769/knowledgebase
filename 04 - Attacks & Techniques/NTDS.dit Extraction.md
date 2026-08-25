---
title: NTDS.dit Extraction
aliases: ["NTDS.dit dump", "ntdsutil IFM", "Domain database dumping"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[AD Database (NTDS.dit)]]", "[[Domain Controller (DC)]]", "[[DCSync]]", "[[krbtgt account]]", "[[Golden Ticket]]"]
created: 2026-06-06
updated: 2026-06-07
---

# NTDS.dit Extraction

> [!summary] One-liner
> Copy the domain database file (plus the SYSTEM hive) off a DC and parse it offline to recover every account's hashes.

## Concept abused
The whole domain's secrets sit in [[AD Database (NTDS.dit)|`C:\Windows\NTDS\ntds.dit`]] on each [[Domain Controller (DC)]], encrypted with the SYSTEM-hive BootKey.

## Prerequisites
- **Administrator / SYSTEM on a DC** (or Backup Operators-style rights, e.g. shadow-copy access).

## How the attack works
1. The live `ntds.dit` is locked, so use a method that snapshots it (IFM / Volume Shadow Copy).
2. Grab the SYSTEM hive too (needed to decrypt).
3. Parse offline.

## Commands & tools

### Local on the DC
```cmd
:: ntdsutil "Install From Media" - creates a copy of ntds.dit + SYSTEM
ntdsutil "ac i ntds" "ifm" "create full C:\temp" q q

:: or Volume Shadow Copy
vssadmin create shadow /for=C:
```

### Parse offline with [[Impacket]]
```bash
secretsdump.py -ntds ntds.dit -system system.bin LOCAL
```

## Detection / artifacts
Shadow-copy creation, `ntdsutil` execution, large file reads/exfil from a DC; EDR on `vssadmin`/`ntdsutil`.

## Mitigation
Restrict DC logon (tier-0), monitor shadow-copy/ntdsutil use, protect Backup Operators.

## More from harmj0y ("The Case of a Stubborn ntds.dit" — handling corrupt / dirty databases)

When an extracted `ntds.dit` was **not shut down cleanly** (a "dirty" / corrupt database), the standard offline-parse toolchain (`libesedb` + `ntdsxtract`/`creddump`) fails outright, and even Microsoft's own repair tool `esentutl` could not always recover the hashes. harmj0y walked through several workarounds.

### Getting the files off the DC (defender-friendly / agentless options)
The post emphasizes pulling `ntds.dit` + the SYSTEM hive via **Volume Shadow Copy** rather than running a live credential-dumping agent on the DC. Transport options noted: RDP session, PSEXEC, WMI, or PowerShell **`Invoke-NinjaCopy`** (copies locked files out of VSS without an active agent).

### Repairing a dirty / corrupt database
On the DC, create a copy with IFM, then run `esentutl` in **repair (`/p`)** mode against the copied database:

```cmd
:: 1) IFM copy (ntds.dit + SYSTEM) into a target path
ntdsutil.exe "activate instance ntds" "ifm" "Create Full <PATH>" quit quit

:: 2) Repair the copied database in place (NOT the live one)
esentutl.exe /p /o "<PATH>\Active Directory\ntds.dit"
```

> [!warning] OPSEC / safety
> Run `esentutl /p` only against the **copied** database (the IFM output), never the live `C:\Windows\NTDS\ntds.dit`. `/p` is a hard-repair that can discard data; it is for offline recovery of a snapshot.

### Offline parsing with [[Impacket]] (esentutl.py + ImpDump)
harmj0y found Impacket's `esentutl.py` more robust than `libesedb` for "wonky" databases. He wrote **ImpDump** to parse the `esentutl.py` table output and decrypt the hashes using the SYSTEM hive:

```bash
# Extract the datatable from a (possibly corrupt) ntds.dit
./extract.sh /path/to/ntds.dit > output

# Decrypt hashes using the SYSTEM hive
./impdump.py /path/to/SYSTEM output > hashes.txt

# Include password history
./impdump.py /path/to/SYSTEM output -history > hash_history.txt
```

As noted in the comments, modern **`secretsdump.py`** can parse `ntds.dit` locally or remotely directly (later Impacket versions), often removing the need for the multi-step pipeline above.

### Very large (30GB+) corrupt databases
For huge `ntds.dit` files that are also corrupt:
1. **On DC:** `ntdsutil.exe "activate instance ntds" "ifm" "Create Full <PATH>" quit quit`, then `esentutl.exe /p /o "<PATH>\Active Directory\ntds.dit"`.
2. **Locally:** use **`libesedb` version 20140406** specifically — newer builds were extremely slow on large files.
3. **Final extraction:** **`ntdsxtract` v1.3** `dsusers.py` together with the SYSTEM hive.

### OPSEC / detection notes from the post
- Avoid Meterpreter's `smart_hashdump` on a live DC — endpoint products (Symantec Endpoint Protection, Sophos Endpoint Protection) were observed killing the Meterpreter session during hashdump attempts.
- Prefer remote/VSS copies over live agent-based dumping on tier-0 hosts.
- Clean up the extracted `ntds.dit`/SYSTEM artifacts from the target after exfil.

## Related
- Remote alternative without file access: [[DCSync]]. Both yield [[krbtgt account|krbtgt]] → [[Golden Ticket]].

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Domain database dumping"
- [[Source - harmj0y blog]] — "The Case of a Stubborn ntds.dit" — https://blog.harmj0y.net/redteaming/the-case-of-a-stubborn-ntds-dit/
