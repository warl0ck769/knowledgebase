---
title: EDR Bypass (Kernel)
aliases: ["EDR bypass", "kernel EDR bypass", "kernel driver EDR", "EDR evasion kernel"]
type: technique
domain: [red-team, windows-internals]
attack_tactic: [defense-evasion]
tags: [type/technique, domain/red-team, domain/windows-internals, attack/defense-evasion]
source: "[[Source - hackndo blog]]"
source_url: "https://en.hackndo.com/write-and-bypass-kernel-edr-part-1/"
verified: true
related: ["[[Windows Services]]", "[[Process and Thread]]", "[[Access Token]]", "[[Integrity Levels]]", "[[UAC Bypass]]"]
created: 2026-06-07
updated: 2026-06-07
---

# EDR Bypass (Kernel)

> [!summary] One-liner
> Kernel-mode EDR components monitor system activity from ring 0 using callbacks and driver infrastructure — bypassing them requires understanding Windows driver architecture, the SSDT, and how kernel notifications work.

## Concept abused

### User mode vs kernel mode

Every Windows process runs in **user mode** (ring 3). User-mode code cannot access hardware, other processes' memory, or kernel structures directly. When an application needs a privileged operation (file I/O, process creation, network), it issues a **system call** that transitions to **kernel mode** (ring 0).

```
┌─────────────────────────────────────────┐
│  User Mode (Ring 3)                     │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ notepad  │ │ chrome   │ │ malware │ │
│  └────┬─────┘ └────┬─────┘ └────┬────┘ │
│       │ syscall     │ syscall    │       │
├───────┼─────────────┼────────────┼──────┤
│  Kernel Mode (Ring 0)                   │
│  ┌──────────────────────────────────┐   │
│  │  SSDT (syscall dispatch table)   │   │
│  │  Drivers · Callbacks · I/O Mgr   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### System Service Dispatch Table (SSDT)

The SSDT maps syscall numbers to kernel functions. When an application calls `NtCreateFile`, the stub in `ntdll.dll` places the syscall number in `EAX` and executes `syscall` — the kernel looks up that number in the SSDT and dispatches to the corresponding `Nt*` function.

**Red-team relevance**: User-mode EDR hooks (e.g., `ntdll.dll` inline hooks) can be bypassed by direct syscall invocation. Kernel-mode EDR components sit below this — they use official callback mechanisms and cannot be bypassed the same way.

### Why EDRs operate in kernel mode

| User-mode EDR | Kernel-mode EDR |
|---|---|
| Hooks `ntdll.dll` exports | Registers kernel callbacks |
| Lives in the target process | Lives in a driver (separate address space) |
| Can be unhooked by the process itself | Cannot be tampered with from user mode |
| Bypassed by direct syscalls, unhooking, loading a fresh ntdll | Requires a signed driver or kernel exploit to bypass |

Kernel-mode EDR advantages:
- **Visibility**: observes all processes, not just those it injects into
- **Tamper resistance**: user-mode malware cannot modify kernel memory
- **Behavioral enforcement**: can block operations before they complete (via pre-operation callbacks)

## Windows kernel driver architecture

### DriverEntry — the entry point

Every kernel driver implements `DriverEntry`, analogous to `main()`:

```c
extern "C"
NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, 
                     _In_ PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = EDRUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = EDRCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = EDRCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = EDRDeviceControl;
    return STATUS_SUCCESS;
}
```

- `PDRIVER_OBJECT`: kernel-initialized structure the driver completes with its dispatch routines
- `DriverUnload`: cleanup function — critical to prevent kernel memory leaks
- `MajorFunction[]`: array of IRP (I/O Request Packet) handlers

### IRP Major Functions

| Index | Purpose |
|---|---|
| `IRP_MJ_CREATE` | Handle open operations (user-mode `CreateFile`) |
| `IRP_MJ_CLOSE` | Handle close operations |
| `IRP_MJ_READ` | Read data from driver |
| `IRP_MJ_WRITE` | Write data to driver |
| `IRP_MJ_DEVICE_CONTROL` | Handle `DeviceIoControl` calls (custom control codes) |

`IRP_MJ_DEVICE_CONTROL` is the primary communication channel — user-mode components send IOCTLs to the driver to exchange data and trigger actions.

### Skeleton EDR driver

```c
#include <ntddk.h>

void EDRUnload(_In_ PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
    KdPrint(("Driver stopped\n"));
}

NTSTATUS EDRCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp) {
    UNREFERENCED_PARAMETER(DeviceObject);
    UNREFERENCED_PARAMETER(Irp);
    return STATUS_SUCCESS;
}

NTSTATUS EDRDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp) {
    UNREFERENCED_PARAMETER(DeviceObject);
    UNREFERENCED_PARAMETER(Irp);
    return STATUS_SUCCESS;
}

extern "C"
NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, 
                     _In_ PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);
    DriverObject->DriverUnload = EDRUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = EDRCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = EDRCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = EDRDeviceControl;
    return STATUS_SUCCESS;
}
```

## Commands & tools

### Development setup
- **Visual Studio** + **Windows 10 SDK** (with debugging tools) + **Windows Driver Kit (WDK)**
- Project type: Empty WDM Driver
- Disable Spectre mitigation in project settings
- Use `extern "C"` on `DriverEntry` to prevent C++ name mangling
- Output: `.sys` binary

### Loading a test driver

```cmd
:: Enable test signing (requires reboot)
bcdedit /set testsigning on

:: Register the driver as a kernel service
sc.exe create EDR type= kernel binPath= C:\path\to\EDR.sys

:: Start the driver
sc.exe start EDR

:: Stop the driver
sc.exe stop EDR

:: Delete the driver service
sc.exe delete EDR
```

> [!warning] Test in a VM
> A buggy kernel driver causes a BSOD. Always develop and test in a virtual machine.

### Debugging

Use **DbgView** (Sysinternals) to capture `KdPrint()` output from the driver in real time.

```cmd
:: Or attach WinDbg to a kernel debug session
windbg -k net:port=50000,key=1.2.3.4
```

### User-mode EDR bypass tools (for context)

| Tool | Technique | Level |
|---|---|---|
| **SysWhispers / SysWhispers2** | Direct syscall stubs (bypass ntdll hooks) | User-mode |
| **Hell's Gate / Halo's Gate** | Dynamic syscall number resolution | User-mode |
| **ntdll unhooking** | Load clean ntdll from disk, overwrite .text | User-mode |
| **PPLdump / PPLFault** | Dump protected processes via PPL bypass | Kernel-adjacent |
| **Vulnerable driver exploits (BYOVD)** | Load signed vulnerable driver → arbitrary kernel r/w | Kernel |

### Kernel-level EDR components to target

| Component | API | What it monitors |
|---|---|---|
| Process creation callbacks | `PsSetCreateProcessNotifyRoutineEx` | New process creation |
| Thread creation callbacks | `PsSetCreateThreadNotifyRoutine` | New thread injection |
| Image load callbacks | `PsSetLoadImageNotifyRoutine` | DLL/driver loading |
| Object callbacks | `ObRegisterCallbacks` | Handle operations (e.g., `OpenProcess`) |
| Registry callbacks | `CmRegisterCallbackEx` | Registry key operations |
| Minifilter | `FltRegisterFilter` | File system operations |

## Detection / artifacts

- **Driver signature enforcement (DSE)**: Windows requires kernel drivers to be signed. Disabling DSE (`bcdedit /set nointegritychecks on`) or test signing leaves forensic traces in BCD store.
- **Event 7045**: new service created (driver loading via `sc.exe create`).
- **Event 4697**: service installed (if audit policy enabled).
- **Sysmon Event 6**: driver loaded — captures hash and signature status.
- **BYOVD indicators**: loading known-vulnerable drivers (e.g., `RTCore64.sys`, `dbutil_2_3.sys`) triggers EDR/AV signatures.
- **Kernel callback enumeration**: defenders can list registered callbacks via WinDbg or tools like `ObjectCallbackScanner`.

## Mitigation

- **Hypervisor-Protected Code Integrity (HVCI)**: blocks unsigned/improperly signed drivers even if DSE is disabled.
- **Virtualization-Based Security (VBS)**: isolates critical kernel memory from even kernel-mode code.
- **Driver blocklist**: Microsoft maintains a list of known-vulnerable drivers (`DriverSiPolicy.p7b`).
- **WDAC (Windows Defender Application Control)**: restrict which drivers can load based on publisher/hash.
- **Credential Guard**: isolates LSASS in a VTL1 container — kernel drivers in VTL0 cannot access it.
- **Secure Boot**: prevents boot-time driver tampering.

## Related

- Process fundamentals: [[Process and Thread]], [[Access Token]], [[Integrity Levels]].
- User-mode evasion: [[UAC Bypass]], [[harmj0y Misc Tradecraft]].
- What EDRs protect: [[LSASS Dumping]], [[Credential Hunting]].
- Driver loading: [[Windows Services]] (kernel services).

## Sources
- [[Source - hackndo blog]] — "Writing and bypassing a kernel-side EDR - Part 1: Kernel & Drivers" — https://en.hackndo.com/write-and-bypass-kernel-edr-part-1/
