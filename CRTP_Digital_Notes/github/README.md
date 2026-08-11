# Certified Red Team Professional (CRTP) — Digital Exam Notes

---

## Navigation

| Module | Topic | Link |
| :--- | :--- | :--- |
| **00** | Quick Reference — Master Attack Flow, Ticket Matrix, Troubleshooting, Loot Tracker, Kerbrute, Impacket, Bloody-AD | [00-QUICK-REFERENCE.md](00-QUICK-REFERENCE.md) |
| **01** | AD Foundations, Kerberos, NTLM, Defense Evasion, Offensive .NET, MDE/MDI | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **02** | Initial SMB, Domain, User, Computer, Group, GPO, AdminSDHolder & Trust Enumeration | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **03** | LDAP Enumeration Commands — Bitmask Rules, UAC Flags, PowerView, AD Module, `[adsisearcher]`, `ldapsearch` | [03-LDAP-ENUMERATION.md](03-LDAP-ENUMERATION.md) |
| **04** | ACL Abuse, GPO Relay, Jenkins Reverse Shells, Delegation | [04-ACL-GPO-JENKINS.md](04-ACL-GPO-JENKINS.md) |
| **05** | Local Admin Hunting, WinRM, WMI, SCM, Netsh Portproxy | [05-LATERAL-MOVEMENT.md](05-LATERAL-MOVEMENT.md) |
| **06** | Kerberoasting, AS-REP Roasting, LSASS, OPTH, DCSync | [06-CREDENTIAL-ACCESS.md](06-CREDENTIAL-ACCESS.md) |
| **07** | Golden/Silver/Diamond Tickets, Unconstrained/Constrained Delegation, RBCD | [07-KERBEROS-DELEGATION.md](07-KERBEROS-DELEGATION.md) |
| **08** | Domain Dominance (Skeleton Key, Custom SSP, DSRM, AdminSDHolder, DCSync ACL), AD CS ESC1, SIDHistory | [08-ADCS-PERSISTENCE.md](08-ADCS-PERSISTENCE.md) |
| **09** | Credential Hunting — PSReadLine, KeePass, Vaults, VSS, NTDS | [09-CREDENTIAL-HUNTING.md](09-CREDENTIAL-HUNTING.md) |
| **10** | Cross-Trust Attacks (Child → Parent ExtraSIDs, AD CS across trusts) & SQL Server Database Links | [10-CROSS-TRUST-AND-SQL.md](10-CROSS-TRUST-AND-SQL.md) |
| **11** | Defensive Controls (LAPS, Cred Guard, Protected Users, SID Filtering) & Security Event IDs | [11-DEFENSIVE-CONTROLS-AND-EVENTS.md](11-DEFENSIVE-CONTROLS-AND-EVENTS.md) |


---

## Placeholder Matrix

Replace these across all commands with your lab/exam values:

| Placeholder | Example | Description |
| :--- | :--- | :--- |
| `<DOMAIN>` | `dcorp.moneycorp.local` | Target Domain FQDN |
| `<DOMAIN_SHORT>` | `dcorp` | NetBIOS Domain Name |
| `<USER>` | `studentx` | Current / Compromised Account |
| `<TARGET>` | `dcorp-adminsrv.dcorp.moneycorp.local` | Target Computer FQDN |
| `<DC>` | `dcorp-dc.dcorp.moneycorp.local` | Domain Controller Hostname |
| `<DC_IP>` | `172.16.2.1` | Domain Controller IP |
| `<AES256_KEY>` | `e61a...` | Kerberos AES256 Key |
| `<NTLM_HASH>` | `2b57...` | NTLM Password Hash |
| `<DOMAIN_SID>` | `S-1-5-21-3537233777-2717541362-1549646036` | Domain SID |
| `<KRBTGT_AES256>` | `c4d9...` | krbtgt AES256 Key |
| `<CERT_PFX>` | `C:\AD\Tools\cert.pfx` | PKCS#12 Certificate |
| `<ENTERPRISE_ADMIN_SID>` | `S-1-5-21-3537233777-2717541362-1549646036-519` | Enterprise Admins Group SID |

---

## Exam Attack Flow

1. **Initial Access & Recon** → Establish local footholds and bypass security controls (`whoami /all`, `hostname`, `ipconfig /all`).
2. **Domain Enumeration** → Discover users, computers, groups, trusts, ACLs, OUs, GPOs, AdminSDHolder, and SMB shares.
3. **Path Identification & Privilege Escalation:**
   - **Delegated Rights / ACLs:** AddMember / ForceChangePassword / GenericAll
   - **Local Admin Rights:** LSASS Dump / SafetyKatz / OPTH
   - **Unquoted / Weak Services:** PowerUp Service Abuse
   - **Vulnerable Certificate Templates:** AD CS ESC1 / SAN Impersonation
4. **Domain Controller Escalation** → DCSync krbtgt Key / Golden Ticket / Diamond Ticket Forgery.
5. **Domain Dominance & Persistence** → Skeleton Key, Custom SSP, DSRM Admin, AdminSDHolder ACLs, DCSync rights.
6. **Cross-Trust & Forest Escalation:**
   - **Intra-Forest:** Child → Parent Root via Trust Key / krbtgt + ExtraSIDs (`Enterprise Admins` `-519`).
   - **Database Links:** SQL Server link crawling across trust boundaries (`PowerUpSQL` / `EXEC AT`).

> [!TIP]
> Systematically enumerate at each hop before escalating. Preserve ticket artifacts (`.kirbi` / `.pfx`) in `C:\AD\Tools\` for session re-entry.

---

## Author

- **Author:** Gokul Amaran
- **LinkedIn:** [linkedin.com/in/gokulamaran](https://linkedin.com/in/gokulamaran)
- **Website:** [gokulamaran.me](https://www.gokulamaran.me)
