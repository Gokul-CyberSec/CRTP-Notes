# 02 — Domain / ACL / OU / GPO / Trust Enumeration

> [!TIP]
> Comprehensive enumeration is the backbone of Active Directory exploitation. Always collect information systematically across Users, Groups, Computers, OUs, GPOs, ACLs, and Trusts before attempting privilege escalation.

---

## Initial SMB Enumeration

Perform unauthenticated or authenticated SMB checks across domain targets.

### NetExec (nxe / cme)
```bash
# Check SMB signing across subnet / targets (find hosts with signing disabled)
nxe smb <TARGET_IP_OR_RANGE> --gen-relay-list hosts.txt

# Null session check
nxe smb <TARGET_IP> -u '' -p '' --shares

# Guest session check
nxe smb <TARGET_IP> -u 'Guest' -p '' --shares

# Authenticated share listing & permissions
nxe smb <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --shares

# Search shares for sensitive files (passwords, configs, keys)
nxe smb <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --spider "C$" --pattern "unattend*,*.kdbx,*.config,pass*"

# Enumerate logged-on users and active shares
nxe smb <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --loggedon-users --shares
```

### smbclient & smbmap
```bash
# Anonymous share listing
smbclient -L //<TARGET_IP> -N
smbmap -H <TARGET_IP> -u 'Guest' -p ''

# Authenticated share listing with smbmap
smbmap -H <TARGET_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN>

# Connect and download files via smbclient
smbclient //<TARGET_IP>/<SHARE> -U '<DOMAIN>\<USER>' -c 'recurse ON; prompt OFF; mget *'
```

### enum4linux-ng
```bash
enum4linux-ng -A <TARGET_IP>
```

### PowerView Shares Enumeration
```powershell
# Enumerate readable shares across domain hosts
Find-DomainShare

# Check share permissions for current user context
Find-DomainShare -CheckShareAccess
```

---

## Domain & Domain Controller Enumeration

```powershell
# PowerView: Get details for current domain
Get-Domain
Get-DomainController

# AD Module: Get domain & DC details
Get-ADDomain
Get-ADDomainController -Filter *
```

---

## User & Service Account Enumeration

```powershell
# PowerView: List all domain users or filter by specific identity
Get-DomainUser
Get-DomainUser -Identity <USER>
Get-DomainUser | Select-Object -ExpandProperty samaccountname

# PowerView: Find accounts protected by AdminSDHolder (AdminCount = 1)
Get-DomainUser -AdminCount

# PowerView: Find accounts with Service Principal Names (Kerberoastable)
Get-DomainUser -SPN | Select-Object samaccountname,serviceprincipalname

# AD Module: User enumeration equivalent
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity <USER> -Properties *
Get-ADUser -Filter {AdminCount -eq 1}
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

---

## Protected Groups & AdminSDHolder Concepts

The `AdminSDHolder` object (`CN=AdminSDHolder,CN=System,DC=domain,DC=local`) protects privileged users and groups (Domain Admins, Enterprise Admins, Schema Admins, Account Operators).

Every 60 minutes, the **SDProp (Security Descriptor Propagator)** process runs on the DC holding the PDC Emulator role:
1. Scans all accounts where `adminCount = 1`.
2. Overwrites their ACL with the ACL of `AdminSDHolder`.
3. Disables inheritance on those objects.

### Enumerate AdminSDHolder Objects
```powershell
# PowerView: Find protected accounts with adminCount = 1
Get-DomainUser -AdminCount
Get-DomainGroup -AdminCount

# PowerView: Inspect AdminSDHolder ACL
Get-DomainObjectAcl -Identity "CN=AdminSDHolder,CN=System,DC=dcorp,DC=moneycorp,DC=local" -ResolveGUIDs
```

---

## Computer & Operating System Discovery

```powershell
# PowerView: Enumerate domain computers & hostnames
Get-DomainComputer
Get-DomainComputer | Select-Object -ExpandProperty dnshostname
Get-DomainComputer | Select-Object samaccountname,logoncount,operatingsystem

# AD Module: Computer enumeration equivalent
Get-ADComputer -Filter * -Properties OperatingSystem,DNSHostName
```

---

## Group & Membership Enumeration

```powershell
# PowerView: Enumerate domain groups & privileged members
Get-DomainGroup
Get-DomainGroup *admin*
Get-DomainGroup -Identity 'Domain Admins'
Get-DomainGroupMember -Identity 'Domain Admins'

# AD Module: Group enumeration equivalent
Get-ADGroup -Filter *
Get-ADGroup -Filter {Name -like "*admin*"}
Get-ADGroupMember -Identity 'Domain Admins'
```

---

## Organizational Unit (OU) & GPO Enumeration

Group Policy Objects (GPOs) control security policies across OUs.

```powershell
# PowerView: Enumerate OUs and GPO links
Get-DomainOU
Get-DomainOU | Select-Object -ExpandProperty name
(Get-DomainOU -Identity <OU>).gplink

# PowerView: List computers residing inside a specific OU
(Get-DomainOU -Identity <OU>).distinguishedname |
  ForEach-Object { Get-DomainComputer -SearchBase $_ } |
  Select-Object name

# PowerView: Enumerate GPOs
Get-DomainGPO
Get-DomainGPO -Identity '<GPO_NAME_OR_GUID>'

# AD Module: Enumerate GPOs & OUs
Get-ADOrganizationalUnit -Filter * -Properties gPLink
Get-ADGPO -All
```

---

## Access Control List (ACL) Enumeration

> [!IMPORTANT]
> Always pass `-ResolveGUIDs` to PowerView ACL cmdlets to automatically translate ObjectGUIDs and RightsGUIDs into human-readable Active Directory permissions (e.g. `ExtendedRight`, `GenericAll`).

```powershell
# Inspect ACL permissions on high-value targets (e.g. Domain Admins group)
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs

# Find all ACL entries where a specific user/group holds explicit rights
Get-DomainObjectAcl -ResolveGUIDs |
  Where-Object { $_.IdentityReferenceName -match '<USER_OR_GROUP>' }

# Find anomalous / non-default ACL permissions across domain objects
Find-InterestingDomainAcl -ResolveGUIDs |
  Where-Object { $_.IdentityReferenceName -match '<USER_OR_GROUP>' }
```

---

## Domain & Forest Trust Enumeration

Trusts enable cross-domain or cross-forest access.

```powershell
# PowerView: Enumerate domain & forest trusts
Get-DomainTrust
Get-ForestDomain -Verbose
Get-ForestDomain | ForEach-Object { Get-DomainTrust -DomainName $_.Name }

# AD Module: Trust enumeration equivalent
Get-ADTrust -Filter *
Get-ADForest | ForEach-Object { Get-ADTrust -Filter * }
(Get-ADForest).Domains
```

---

## Network Shares & Active Session Hunting

```powershell
# PowerView: Enumerate open network shares across domain computers
Find-DomainShare

# PowerView: Enumerate active logged-on user sessions across domain hosts
Get-NetSession

# SharpHound / BloodHound collector (Alternative for automated graphing)
. C:\AD\Tools\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All
```

---

## Local Admin Access & Remote Execution Enumeration

```powershell
# PowerView: Scan domain hosts to find where current user has local admin access
Find-LocalAdminAccess

# Check local admin access via PSRemoting (WinRM Port 5985/5986)
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1

# Enumerate members of local Administrators group on a remote host
Get-NetLocalGroup -ComputerName <TARGET>
Get-NetLocalGroupMember -ComputerName <TARGET> -GroupName Administrators
```

```cmd
# Remote interactive command line via WinRS
winrs -r:<TARGET> cmd
```

```powershell
# Remote interactive PowerShell session
Enter-PSSession -ComputerName <TARGET>
```
