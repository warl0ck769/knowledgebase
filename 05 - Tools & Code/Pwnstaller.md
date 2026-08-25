---
title: Pwnstaller
aliases: [Pwnstaller 1.0]
type: tool
domain: [red-team, windows-internals]
tags: [type/tool, domain/red-team, domain/windows-internals, attack/defense-evasion]
source: "[[Source - harmj0y blog]]"
source_url: https://blog.harmj0y.net/python/pwnstaller-1-0/
verified: true
related: ["[[Empire]]", "[[GhostPack]]"]
created: 2026-06-07
updated: 2026-06-07
---

> [!summary]
> Pwnstaller is a tool that **dynamically recompiles the PyInstaller `runw.exe` loader with obfuscation** so that Python payloads packaged as standalone Windows EXEs evade the static antivirus signatures that began flagging the stock PyInstaller loader. It ships as a payload-generation option in [[Empire|Veil-Evasion]].

## What it does
[PyInstaller](https://www.pyinstaller.org/) packs a Python script into a standalone Windows executable: the output is a small **loader EXE with a CArchive appended at the end**, containing the compressed Python runtime, libraries, and the target script. At runtime the loader extracts those components, loads the libraries, and executes the script.

The problem: AV vendors started detecting **static signatures in the stock PyInstaller loader** (`runw.exe`), so any payload packed with PyInstaller — regardless of the payload itself — could be flagged. There were also DEP (Data Execution Prevention) compatibility issues that limited some shellcode-injection techniques in PyInstaller-packed EXEs.

Pwnstaller's fix is to **recompile the `runw.exe` loader from source on Kali Linux** (using `mingw32`), applying randomization/obfuscation on every run so no two loaders share the same static fingerprint. This makes writing a reliable static detection signature "a reasonable difficulty." It is not a full evasion solution — it specifically targets the *loader* signature problem, not the packed payload.

Each invocation produces:
- Obfuscated source files for the PyInstaller launcher
- A randomly selected executable **icon**
- A freshly compiled `runw.exe` built with `mingw32`
- Updated resource locations wired back into the PyInstaller build process

## Obfuscation / randomization applied
- **Stripped** all non-Windows code from the loader source
- **Randomized library imports**
- **Shuffled and randomized code sections**
- **Interspersed processing methods** to complicate the call tree / control flow
- **Randomized executable icon** per build

## Key commands/functions
Pwnstaller was incorporated into the development branch of **Veil-Evasion** rather than used as a standalone CLI in normal workflows.

```text
# In Veil-Evasion's Python compilation menu, choose:
2 - Pwnstaller

# Or via the command-line flag:
veil-evasion ... --pwnstaller
```

## Used in techniques
- Payload packaging / defense evasion for Python-based Windows implants delivered via [[Empire|Veil-Evasion]]
- Part of the broader offensive tooling family alongside [[GhostPack]] / [[Empire]]

## Sources
- Pwnstaller 1.0 — https://blog.harmj0y.net/python/pwnstaller-1-0/
