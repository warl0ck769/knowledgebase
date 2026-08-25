---
title: SSH
aliases: ["Secure Shell", "OpenSSH", "sshd"]
type: concept
domain: [windows-internals]
tags: [type/concept, domain/windows-internals]
source: "Microsoft documentation"
source_url: "https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview"
verified: true
related: ["[[Linux in AD]]", "[[Remote Execution & Lateral Movement (Windows)]]", "[[Kerberos]]"]
created: 2026-06-07
updated: 2026-06-07
---

# SSH (Secure Shell)

> [!summary] One-liner
> An encrypted remote-access protocol (port 22) — natively available on Windows since Server 2019 / Windows 10 1809 via **OpenSSH**, and the standard management interface for [[Linux in AD]] machines.

## OpenSSH on Windows
- **Client**: `ssh.exe` — built into Windows 10+ (optional feature).
- **Server**: `sshd.exe` — installable as a Windows service; supports password and public-key auth.
- Config: `C:\ProgramData\ssh\sshd_config` and `%USERPROFILE%\.ssh\`.

## SSH + Kerberos
On domain-joined Linux/Windows hosts, SSH can authenticate via **GSSAPI-with-mic** (Kerberos):
- Client presents a Kerberos service ticket for `host/<target>`.
- No password transmitted; SSO experience.
- See [[Linux in AD]] for how Linux hosts join AD and accept Kerberos SSH auth.

## Red-team relevance
- **Lateral movement**: SSH provides an interactive shell on Linux hosts (and increasingly on Windows). [[Remote Execution & Lateral Movement (Windows)]] covers Windows-native options; SSH is the Linux equivalent.
- **Key-based persistence**: Planting a public key in `~/.ssh/authorized_keys` (or Windows equivalent) gives persistent, password-less access.
- **Port forwarding / tunneling**: `ssh -L` / `ssh -D` (SOCKS) for pivoting through compromised hosts.
- **Credential capture**: If an attacker controls a host running `sshd`, password-auth connections reveal plaintext passwords.

## Key commands
```bash
# Connect
ssh <USER>@<TARGET>

# SOCKS proxy (pivoting)
ssh -D 1080 <USER>@<PIVOT_HOST>

# Local port forward
ssh -L <LOCAL_PORT>:<TARGET>:<REMOTE_PORT> <USER>@<PIVOT_HOST>

# Copy SSH key for persistence
echo "<ATTACKER_PUBKEY>" >> ~/.ssh/authorized_keys
```

## Related
- Linux domain members: [[Linux in AD]].
- Windows lateral movement: [[Remote Execution & Lateral Movement (Windows)]].
- Kerberos integration: [[Kerberos]].
