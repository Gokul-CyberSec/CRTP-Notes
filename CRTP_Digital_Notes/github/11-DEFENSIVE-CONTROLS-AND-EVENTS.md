# 11 — Defensive Controls & Event Logs


> [!NOTE]
> Understanding defensive controls and security Event IDs helps red teamers maintain OPSEC, anticipate security restrictions, and understand how attacks are audited in Active Directory.

---

## Active Directory Defensive Controls Matrix

| Defensive Control | Core Function & Impact | Bypass / OPSEC Note |
| :--- | :--- | :--- |
| **LAPS (Local Administrator Password Solution)** | Rotates local Administrator account passwords automatically on domain hosts; stores password in AD attribute. | Query `ms-MCSF-UserPassword` (legacy LAPS) or `msLAPS-Password` if holding `GenericAll` / read rights over computer object. |
| **Credential Guard** | Uses virtualization-based security (VBS) to isolate LSASS secrets in a virtual container. | Prevents LSASS dumping (`sekurlsa::logonpasswords`). Pivot to token impersonation, DCSync, or Kerberoasting instead. |
| **Protected Users Group** | Restricts accounts in this group: disables NTLM authentication, DES/RC4 Kerberos encryption, and CredSSP caching. | Accounts cannot be targeted via NTLM relaying or WDigest dumping. Must use AES Kerberos authentication. |
| **SID Filtering** | Strips foreign SIDs (e.g. `ExtraSIDs`) on cross-forest trusts to prevent SID History spoofing across forests. | Enforced by default on external/forest trusts. Does NOT apply to intra-forest child-to-parent trusts unless explicitly enabled. |
| **Selective Authentication** | Requires explicit permission (`Allowed to authenticate`) on target computer objects across external trusts. | Standard domain accounts cannot access resources across the trust without explicit ACEs. |
| **PAW / Tiering Model** | Privileged Access Workstations & Tier 0/1/2 administrative isolation. | Prevents Tier 0 accounts (Domain Admins) from logging into Tier 2 workstations. Look for tiering violations to capture DA sessions. |

---

## Essential Active Directory Security Event IDs

### Logon & Authentication Logs

| Event ID | Event Description | Attack Vector Monitored |
| :---: | :--- | :--- |
| **4624** | An account was successfully logged on | Interactive logon, WinRM / network logon (`Logon Type 3 / 10`) |
| **4625** | An account failed to log on | Password spraying (`kerbrute`), brute-force attempts |
| **4648** | A logon was attempted using explicit credentials | Overpass-the-Hash (`sekurlsa::pth`), `runas` execution |
| **4672** | Special privileges assigned to new logon | Local Admin / SYSTEM logon |

### Kerberos Event Logs (DC Level)

| Event ID | Event Description | Attack Vector Monitored |
| :---: | :--- | :--- |
| **4768** | A Kerberos authentication ticket (TGT) was requested | AS-REP Roasting (`GetNPUsers`), Golden Ticket TGT request |
| **4769** | A Kerberos service ticket (TGS) was requested | Kerberoasting (`GetUserSPNs`), Silver Ticket, S4U delegation |
| **4771** | Kerberos pre-authentication failed | Failed AS-REQ attempts, bad password in Kerberos auth |
| **4776** | The DC attempted to validate credentials (NTLM) | NTLM authentication / NTLM relay attempts |

### Account & Group Modifications

| Event ID | Event Description | Attack Vector Monitored |
| :---: | :--- | :--- |
| **4720** | A user account was created | Rogue account persistence (`New-MachineAccount`) |
| **4724** | An attempt was made to reset an account's password | `ForceChangePassword` ACL abuse (`Set-ADAccountPassword`) |
| **4728** | A member was added to a security group | `AddMembers` ACL abuse (`Add-DomainGroupMember`) |
| **4738** | A user account was modified | Password reset, scriptpath modification, user account control change |
| **4764** | A group's type was changed | Group modification persistence |

### Directory Replication & AD CS

| Event ID | Event Description | Attack Vector Monitored |
| :---: | :--- | :--- |
| **4662** | An operation was performed on an object | DCSync execution (`DS-Replication-Get-Changes-All` GUID access) |
| **4886** | Certificate Services received a certificate request | AD CS ESC1 request (`Certify.exe`) |
| **4887** | Certificate Services approved & issued a certificate | Vulnerable certificate issuance |
