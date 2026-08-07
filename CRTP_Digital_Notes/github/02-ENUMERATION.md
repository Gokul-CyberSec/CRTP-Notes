# 02 — Domain / ACL / OU / GPO / Trust Enumeration

> [!TIP]
> Comprehensive enumeration is the backbone of Active Directory exploitation. Always collect information systematically across Users, Groups, Computers, OUs, GPOs, ACLs, and Trusts before attempting privilege escalation.

---

## Table of Contents

- [Domain Enumeration Execution Checklist](#domain-enumeration-execution-checklist)
- [Domain & Domain Controller Enumeration](#domain--domain-controller-enumeration)
- [User & Service Account Enumeration](#user--service-account-enumeration)
- [Computer & Operating System Discovery](#computer--operating-system-discovery)
- [Group & Membership Enumeration](#group--membership-enumeration)
- [Organizational Unit (OU) & GPO Enumeration](#organizational-unit-ou--gpo-enumeration)
- [Access Control List (ACL) Enumeration](#access-control-list-acl-enumeration)
- [Domain & Forest Trust Enumeration](#domain--forest-trust-enumeration)
- [Network Shares & Active Session Hunting](#network-shares--active-session-hunting)
- [Local Admin Access & Remote Execution Enumeration](#local-admin-access--remote-execution-enumeration)

---

## Domain Enumeration Execution Checklist

- [ ] **1. Domain Architecture:** Query FQDN, domain SID, Forest name, and DCs (`Get-Domain`, `Get-DomainController`).
- [ ] **2. User Enumeration:** Export list of all domain users (`Get-DomainUser`).
- [ ] **3. Service Accounts (Kerberoasting):** Identify accounts with non-null SPNs (`Get-DomainUser -SPN`).
- [ ] **4. AS-REP Roasting:** Check for users with Kerberos pre-authentication disabled (`Get-DomainUser -PreauthNotRequired`).
- [ ] **5. AdminCount Check:** Find accounts protected by AdminSDHolder (`Get-DomainUser -AdminCount`).
- [ ] **6. Computer Discovery:** List all domain workstations and servers with OS details (`Get-DomainComputer`).
- [ ] **7. Privileged Groups:** Enumerate Domain Admins, Enterprise Admins, Account Operators, DNSAdmins (`Get-DomainGroupMember`).
- [ ] **8. OUs & GPO Links:** Map OUs and inspect GPLinks (`Get-DomainOU`, `Get-DomainGPO`).
- [ ] **9. ACL Enumeration:** Scan object DACLs for interesting ACEs (`Find-InterestingDomainAcl -ResolveGUIDs`).
- [ ] **10. Trust Mapping:** Enumerate inbound/outbound domain & forest trusts (`Get-DomainTrust`, `Get-ForestTrust`).
- [ ] **11. Share Discovery:** Scan domain hosts for open network shares (`Find-DomainShare`).
- [ ] **12. Active Sessions:** Locate logged-in Domain Admins (`Get-NetSession`, `Invoke-SessionHunter`).
- [ ] **13. Local Admin Access:** Identify hosts where current credentials have administrative rights (`Find-LocalAdminAccess`).

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
> **Source:** handwritten pp. 9–10.

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
> **Source:** handwritten pp. 9–11.

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
> **Source:** handwritten pp. 10, 15.

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
> **Source:** handwritten pp. 11, 15.

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
> **Source:** handwritten p. 12.

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
> **Source:** handwritten p. 11.

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
> **Source:** handwritten pp. 13–14.

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
> **Source:** handwritten pp. 10, 16.

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
> **Source:** handwritten pp. 15–16.
