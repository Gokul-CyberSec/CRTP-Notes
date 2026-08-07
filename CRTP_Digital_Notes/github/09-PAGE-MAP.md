# 09 — Page-by-Page Topic Map

> [!NOTE]
> This document maps every page from the original **59-page handwritten CRTP note set** to its corresponding digital module file in this repository.

---

## Page Mapping Matrix

| Page # | Core Topics & Attack Vectors | Digital Module Link |
| :---: | :--- | :--- |
| **1** | Security principals, machine accounts, password reset, GPO intro | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **2** | Kerberos TGT request architecture (AS-REQ / AS-REP) | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **3** | Kerberos TGS / Service ticket flow (TGS-REQ / TGS-REP) | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **4** | NTLM challenge/response authentication & relay concept | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **5** | LDAP protocol & LDAP pass-back attack vector | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **6** | Active Directory services (Schema, Global Catalog, Replication) | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **7** | Forest/domain/OU structure, PowerShell fundamentals, dot sourcing | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **8** | Active Directory module import, PowerShell security bypass notes | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) |
| **9** | DefenderCheck, byte patching, PowerUp, initial domain enumeration | [01-FOUNDATIONS.md](01-FOUNDATIONS.md) / [02-ENUMERATION.md](02-ENUMERATION.md) |
| **10** | Core PowerView enumeration (Users, Computers, DCs, Shares, Local Admin) | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **11** | Attribute expansion (`samaccountname`, `dnshostname`), ACL enumeration | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **12** | Organizational Unit (OU) & Group Policy Object (GPO) enumeration | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **13** | Domain trust enumeration (PowerView & AD Module) | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **14** | Forest trusts, PowerUp `Invoke-AllChecks`, service abuse local privesc | [02-ENUMERATION.md](02-ENUMERATION.md) / [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) |
| **15** | Domain group & local administrators group membership enumeration | [02-ENUMERATION.md](02-ENUMERATION.md) |
| **16** | Network share discovery, local admin hunting, PSRemoting, WinRS | [02-ENUMERATION.md](02-ENUMERATION.md) / [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **17** | Jenkins CI/CD pipeline exploitation notes, AppLocker policy registry queries | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) |
| **18** | Jenkins reverse shell execution (`Invoke-PowerShellTcp.ps1`), GPO coercion | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) |
| **19** | NTLM relaying, GPO DACL modification (`write_gpo_dacl`), malicious GPO update | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) |
| **20** | Share staging, PsExec service execution, WinRM remote execution | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) / [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **21** | `Invoke-Command` multi-host execution, `Invoke-SessionHunter`, LSASS overview | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) / [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **22** | Credential extraction sources (SAM, LSA, LSASS), Mimikatz execution | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **23** | Overpass-the-Hash (OPTH) via Mimikatz (`sekurlsa::pth`) and Rubeus | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **24** | Rubeus elevated Opsec NetOnly TGT request, DCSync overview | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **25** | Domain Admin session hunting (`Find-DomainUserLocation`), Jenkins AMSI workflow | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **26** | Memory loader to Jenkins shell, Windows `netsh portproxy` pivoting | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) / [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **27** | SafetyKatz execution through network pivot, OPTH ticket reuse | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **28** | KeePass database discovery, memory loader & portproxy integration | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) / [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **29** | In-memory SafetyKatz execution, service account OPTH, GPMC update | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **30** | Domain escalation chain: app-admin to DC, SafetyKatz on DC | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **31** | DCSync `krbtgt` account hash, Golden Ticket forgery via Rubeus | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) / [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **32** | Silver Ticket forgery via Rubeus (HTTP, HOST, RPCSS services) | [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **33** | WMI architecture and lateral movement overview | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **34** | PsExec, WinRS, Service Control Manager (SCM) remote execution | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **35** | `sc.exe` remote service creation with `msfvenom` service payloads | [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **36** | WMI execution cmdlets, ACL permission delegation attack primitives list | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) / [04-LATERAL-MOVEMENT.md](04-LATERAL-MOVEMENT.md) |
| **37** | Privilege delegation: `AddMember`, `ForceChangePassword` execution | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) |
| **38** | Password reset via `Set-ADAccountPassword`, Kerberos delegation types | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) / [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **39** | Constrained delegation enumeration (`Get-NetUser -TrustedToAuth`), Kekeo | [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **40** | S4U2self / S4U2proxy flow, ticket import, remote `Enter-PSSession` | [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **41** | S4U protocol extension details, user credential hunting overview | [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) / [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **42** | KeePass database exploitation path via Meterpreter & keylogging | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **43** | Process migration into user session context via Meterpreter | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **44** | Web-hosted PowerShell payload execution & process migration | [03-ACL-GPO-JENKINS.md](03-ACL-GPO-JENKINS.md) / [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **45** | AD CS PKI architecture, CA, CSR, and EKU definitions | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **46** | Vulnerable certificate template conditions (ESC1 checklist) | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **47** | SAN / UPN certificate request abuse & Rubeus PKINIT TGT request | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **48** | Golden Ticket vs. Silver Ticket comparison summary | [06-KERBEROS-DELEGATION.md](06-KERBEROS-DELEGATION.md) |
| **49** | Persistence via CA private key extraction (`crypto::certificates /export`) | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **50** | Offline certificate forgery (`ForgeCert.exe`) & Rubeus TGT request | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **51** | `SIDHistory` persistence injection via `DSInternals` (`Add-ADDBSidHistory`) | [07-ADCS-PERSISTENCE.md](07-ADCS-PERSISTENCE.md) |
| **52** | Credential hunting locations, PSReadLine history, registry search | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **53** | Password managers, memory dumping overview, SAM database path | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **54** | Meterpreter hash dumping, Volume Shadow Copy (VSS) snapshot creation | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **55** | Offline SAM & SYSTEM extraction via Impacket `secretsdump.py`, LSASS | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) / [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **56** | ProcDump LSASS dump, Mimikatz live memory extraction, PPL overview | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) |
| **57** | Mimikatz driver (`mimidrv.sys`) PPL removal, Windows Credential Manager | [05-CREDENTIAL-ACCESS.md](05-CREDENTIAL-ACCESS.md) / [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **58** | Windows `vaultcmd` execution, `Get-WebCredentials.ps1` helper script | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
| **59** | `NTDS.dit` Active Directory database architecture (Data, Link, Schema tables) | [08-CREDENTIAL-HUNTING.md](08-CREDENTIAL-HUNTING.md) |
