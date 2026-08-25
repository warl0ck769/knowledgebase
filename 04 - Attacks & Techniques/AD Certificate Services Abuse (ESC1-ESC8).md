---
title: AD Certificate Services Abuse (ESC1-ESC8)
aliases: ["ADCS abuse", "ESC1", "ESC8", "Certified Pre-Owned", "Certipy", "Certify", "PKINIT abuse"]
type: technique
domain: [red-team]
attack_tactic: [privesc, persistence, credential-access]
tags: [type/technique, domain/red-team, attack/privesc, proto/kerberos]
source:
  - "[[Source - zer1t0 - Attacking Active Directory]]"
  - "[[Source - harmj0y blog]]"
source_url: "https://zer1t0.gitlab.io/posts/attacking_ad/"
verified: true
related: ["[[Kerberos]]", "[[NTLM Relay]]", "[[ACL Abuse]]", "[[Overpass-the-Hash]]", "[[Domain Controller (DC)]]"]
created: 2026-06-06
updated: 2026-06-07
---

# AD Certificate Services Abuse (ESC1-ESC8)

> [!summary] One-liner
> Misconfigured AD Certificate Services lets attackers enroll certificates that authenticate as privileged users (via PKINIT) — often a direct path to Domain Admin and durable persistence.

> [!note] Sourcing
> zer1t0 names ADCS, the ESC1–ESC8 taxonomy, and tools (Certify/Certipy). The detailed ESC mechanics below are the canonical taxonomy from **SpecterOps "Certified Pre-Owned"** (the authoritative source for these). Treat specifics as that body of work.

## Concept abused
ADCS issues certificates; AD supports **PKINIT** — using a certificate to obtain a Kerberos **TGT**. If a template/CA is misconfigured so a low-priv user can get a cert **as someone else**, that cert = that identity.

## The ESC catalog (summary)
| ID | Misconfiguration |
|---|---|
| **ESC1** | Template allows **requester-supplied SAN** + Client Auth EKU + low-priv enroll → request a cert *as Administrator* |
| **ESC2** | Template has **Any Purpose** / no EKU |
| **ESC3** | Vulnerable **Enrollment Agent** template |
| **ESC4** | **Write/ACL** over a template → make it ESC1 ([[ACL Abuse]]) |
| **ESC5** | ACL over **PKI objects** (CA, etc.) |
| **ESC6** | CA flag **`EDITF_ATTRIBUTESUBJECTALTNAME2`** → SAN on any request |
| **ESC7** | **ManageCA / ManageCertificates** rights on the CA |
| **ESC8** | **NTLM relay** to the CA's **HTTP web enrollment** ([[NTLM Relay]]) |

## Commands & tools
```bash
# Certipy (Python) - find misconfigs, request, authenticate
certipy find -u user@contoso.local -p pass -dc-ip 192.168.0.2 -vulnerable
certipy req -u user@contoso.local -p pass -ca CA-NAME -template VulnTemplate -upn administrator@contoso.local
certipy auth -pfx administrator.pfx -dc-ip 192.168.0.2     # -> TGT / NT hash
```
```powershell
# Certify (C#)
Certify.exe find /vulnerable
Certify.exe request /ca:CA\CA-NAME /template:VulnTemplate /altname:Administrator
```
The resulting cert → TGT via PKINIT → [[Overpass-the-Hash|use like any credential]].

## Detection / artifacts
Cert enrollment events (4886/4887), requests with SAN ≠ requester, web-enrollment auth from unexpected hosts.

## Mitigation
Audit templates (remove requester SAN, restrict enroll), disable web enrollment / enforce EPA (ESC8), restrict CA ACLs.

---

## More from harmj0y (Certified Pre-Owned — the canonical ADCS abuse paper)

This is the original "Certified Pre-Owned" research (Will Schroeder & Lee Christensen, SpecterOps). It defines the **THEFT** (credential theft), **PERSIST** (persistence), and **ESC** (domain escalation) families. Research authors note that nearly every environment they examined with AD CS installed was vulnerable to at least one escalation vector.

### Authentication-enabling EKUs
A certificate is usable for AD authentication only if it carries an EKU (Extended/Enhanced Key Usage) that enables authentication:

| EKU Type | OID |
|---|---|
| Client Authentication | `1.3.6.1.5.5.7.3.2` |
| PKINIT Client Auth (not default; must be manually added) | `1.3.6.1.5.2.3.4` |
| Smart Card Logon | `1.3.6.1.4.1.311.20.2.2` |
| Any Purpose | `2.5.29.37.0` |
| SubCA (no EKU at all) | N/A |
| Certificate Request Agent (enrollment agent — see ESC3) | `1.3.6.1.4.1.311.20.2.1` |

### ESC mechanics — additional detail vs the summary above
- **ESC1** — The full required condition set: Enterprise CA grants low-priv users enrollment rights; **manager approval disabled**; **no authorized signatures required**; overly permissive template security descriptor; template defines an authentication EKU; and the template lets the requester supply the SAN. The SAN-supply behavior is controlled by the `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` flag in the template's `mspki-certificate-name-flag` property. "If a requester can specify the SAN in a CSR, the requester can request a certificate as anyone."
- **ESC2** — Same conditions as ESC1 except the EKU is **Any Purpose** or **SubCA (no EKU)**. These authenticate without needing to specify a SAN. "Any Purpose" certs work for client auth directly; SubCA certs can forge new certificates (though not trusted by default for domain auth).
- **ESC3** — Template grants the **Certificate Request Agent EKU** (`1.3.6.1.4.1.311.20.2.1`), which lets a principal enroll for a certificate **on behalf of another user**. With no enrollment-agent restrictions, attackers co-sign requests. Note: all Version 1 templates are unprotected against this.
- **ESC4** — Overly permissive template ACEs (e.g. **Domain Computers with FullControl or WriteDacl**) let an unprivileged principal edit the template into a dangerous (ESC1-like) state. See [[ACL Abuse]].
- **ESC5** — Vulnerable ACLs on **PKI objects**: the CA computer object, its RPC/DCOM server, and descendant objects under `CN=Public Key Services,CN=Services,CN=Configuration`.
- **ESC6** — The CA-wide flag `EDITF_ATTRIBUTESUBJECTALTNAME2`. When set, **any request** (even when the subject is built from AD) can carry user-defined SAN values, turning every enrollable template into an ESC1.
- **ESC7** — **ManageCA** allows modifying persistent CA config (including flipping `EDITF_ATTRIBUTESUBJECTALTNAME2` → ESC6). **ManageCertificates** can approve pending requests, bypassing the manager-approval protection.
- **ESC8** — NTLM relay to HTTP enrollment endpoints (Certificate Authority Web Enrollment, Certificate Enrollment Web Service, NDES). Coerce machine-account auth (MS-RPRN / SpoolSample / Dementor) and relay to web enrollment to request a Machine/Computer template cert with client-auth EKU. Key finding: with AD CS + a vulnerable web enrollment endpoint + at least one published template allowing domain-computer enrollment & client auth, "an attacker can compromise ANY computer with the spooler service running."

### THEFT — certificate / credential theft
- **Active enrollment (THEFT, no LSASS touch):** Use `Certify.exe find /clientauth` to LDAP-query templates carrying authentication EKUs, then enroll. "This is an alternative method of long-term credential theft that doesn't touch LSASS and can be performed from a non-elevated context." Alternatives: `certreq.exe` or the `certmgr.msc` GUI.
- Exported user/machine certs can be harvested from the credential store; **SharpDPAPI** decrypts DPAPI-protected private keys.

### PKINIT / UnPAC-the-hash
- **Rubeus** implements PKINIT, so a stolen/forged certificate yields a Kerberos **TGT** without a physical smart card or the Windows Credential Store: "We don't need a physical smart card or the Windows Credential Store to perform this certificate-based Kerberos authentication." (Kekeo has supported PKINIT for years; LDAPS auth is also possible via Schannel.)
- A machine certificate + **S4U2Self** yields service tickets for any service (CIFS, HTTP, RPCSS) as any user.
- **UnPAC-the-hash:** the PKINIT TGT exchange can return the account's NT hash in the PAC, recovering the password hash from just a certificate.

### PERSIST — durable persistence
- **Certificate persistence property:** "Certificates will still be usable even if the user (or computer) resets their password." Validity can be 1+ years, independent of password resets.
- **CA private key extraction (PERSIST / Golden Certificate setup):** Mimikatz / **SharpDPAPI** extract the CA cert + private key from the CA server (protected by machine DPAPI when no TPM/HSM is used).
- **Golden Certificate (ForgeCert, released Black Hat 2021):** with the stolen CA private key, sign **arbitrary certificates for any user**. "These certs can't be revoked, since they were never actually issued by the CA itself." Default 5-year CA cert validity enables multi-year persistence.

### Detection / defensive tooling (PSPKIAudit)
**PSPKIAudit** (PowerShell, built on the PSPKI module):
```powershell
# Identify templates with authentication EKUs (dangerous candidates)
Get-AuditCertificateTemplate | ?{$_.HasAuthenticationEku}

# Comprehensive misconfiguration audit
Invoke-PKIAudit

# Triage certificates issued to (potentially) compromised accounts
Get-CertRequest
```
Incident response: when an account/machine is compromised, IR must **identify and invalidate any certificates** tied to it — otherwise the attacker can authenticate "for years – even after the account's password has been reset."

### Mitigation / hardening (harmj0y)
- **ESC6 / HTTP endpoints:** disable HTTP enrollment roles, disable NTLM via GPO, or enforce HTTPS-only with **Extended Protection for Authentication (EPA)**; configure IIS to accept only Kerberos or implement EPA.
- **Treat CA servers (including subordinate CAs) as Tier 0 assets** with the same protections as Domain Controllers.
- **Root CA key compromise** likely requires rebuilding the entire AD CS system, invalidating every issued certificate.

### OPSEC notes
- Certificate abuse is **non-destructive**, needs no elevation, and **does not touch LSASS**.
- Stolen/forged certs are **password-independent** and valid for 1+ years.
- There is a **detection gap**: limited public IR guidance means certificate-based persistence is frequently missed.

## Related
- [[Kerberos]], [[NTLM Relay]], [[ACL Abuse]], [[Overpass-the-Hash]], [[Domain Controller (DC)]]

## Sources
- [[Source - zer1t0 - Attacking Active Directory]] — "Microsoft extras → ADCS"
- ESC taxonomy: SpecterOps *Certified Pre-Owned* (Schroeder & Christensen).
- [[Source - harmj0y blog]] — "Certified Pre-Owned" — https://blog.harmj0y.net/activedirectory/certified-pre-owned/
