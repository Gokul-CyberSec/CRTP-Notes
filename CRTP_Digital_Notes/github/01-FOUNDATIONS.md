# 01 — AD + PowerShell Foundations

> [!NOTE]
> Active Directory (AD) serves as Microsoft's centralized Identity and Access Management (IAM) infrastructure. Understanding the underlying authentication flows (Kerberos and NTLM) is mandatory before attempting domain exploitation.

---

## Security Principals & Objects

- **Users:** Individual user identities authenticated by the Domain Controller (DC).
- **Computers/Machines:** Machine identities represented in AD with a trailing `$` (e.g., `DC01$`, `WEBSRV01$`). Machine account passwords rotate automatically every 30 days by default.
- **Groups:** Security groups used to assign rights and permissions to users/computers.
- **Service Accounts:** Accounts used to run services/daemon processes (Managed Service Accounts, gMSAs, or standard user accounts with SPNs).

---

## Active Directory Core Architecture

An Active Directory environment is structured hierarchically across Forests, Domains, and Organizational Units (OUs).

```mermaid
graph TD
    Forest["Active Directory Forest (Schema & Global Catalog Boundary)"]
    Domain1["Root Domain (e.g. moneycorp.local)"]
    Domain2["Child Domain (e.g. dcorp.moneycorp.local)"]
    OU1["OU: Servers"]
    OU2["OU: Admins"]
    OU3["OU: Users"]
    GPO1["Default Domain Policy GPO"]
    GPO2["Server Hardening GPO"]

    Forest --> Domain1
    Forest --> Domain2
    Domain2 --> OU1
    Domain2 --> OU2
    Domain2 --> OU3
    GPO1 -. Linked to .-> Domain2
    GPO2 -. Linked to .-> OU1
```

### Core Services
- **Schema:** Defines all object classes and attributes allowed in the directory.
- **Global Catalog (GC):** A distributed data repository containing a searchable subset of attributes for every object in the forest (Port 3268/3269).
- **Replication Service:** Synchronizes directory updates across all Domain Controllers using RPC (MS-DRSR).
- **Group Policy Objects (GPOs):** Configurations applied to Sites, Domains, or OUs to enforce security settings and user environment policies.

---

## Kerberos Authentication Architecture

Kerberos is the primary authentication protocol in Active Directory (Port 88). It relies on symmetric key cryptography and trusted third-party Key Distribution Centers (KDC, running on DCs).

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant KDC as KDC (Domain Controller)
    participant Service as Target Service / Server

    Note over Client,KDC: Phase 1: Ticket Granting Ticket (TGT) Request
    Client->>KDC: AS-REQ (Username + Timestamp encrypted with Client Hash)
    KDC-->>Client: AS-REP (TGT encrypted with krbtgt key + Session Key)

    Note over Client,KDC: Phase 2: Service Ticket (TGS) Request
    Client->>KDC: TGS-REQ (TGT + Authenticator + Target SPN)
    KDC-->>Client: TGS-REP (TGS Ticket encrypted with Service Hash + Service Session Key)

    Note over Client,Service: Phase 3: Service Access
    Client->>Service: AP-REQ (TGS Ticket + Authenticator)
    Service-->>Client: AP-REP (Access Granted / Mutual Auth)
```

> [!IMPORTANT]
> - **TGT Encryption:** Encrypted with the `krbtgt` account password hash/AES key. Anyone with the `krbtgt` hash can forge a TGT (**Golden Ticket**).
> - **TGS Encryption:** Encrypted with the service account password hash/AES key. Anyone with the service account hash can forge a TGS (**Silver Ticket**).

---

## NTLM Challenge/Response Authentication

NTLM is used when Kerberos is unavailable (e.g., connecting via IP address, cross-forest non-transitive trusts, or legacy applications).

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Server as Target Server
    participant DC as Domain Controller

    Client->>Server: 1. Negotiate / Type 1 Message
    Server-->>Client: 2. Challenge / Type 2 Message (8-byte random challenge)
    Client->>Server: 3. Authenticate / Type 3 Message (Response encrypted with NTLM hash)
    Server->>DC: 4. Netlogon Validate Request (Challenge + Response)
    DC-->>Server: 5. Validation Result (Success/Failure)
    Server-->>Client: 6. Access Granted / Denied
```

> [!WARNING]
> **Relay Risks:** NTLM Type 3 responses can be relayed to target hosts if SMB Signing or HTTP Extended Protection for Authentication (EPA) is not enforced.

---

## LDAP & Directory Operations

- **LDAP (Lightweight Directory Access Protocol):** Protocol used to query and update directory objects (Port 389, LDAPS Port 636).
- **LDAP Pass-Back Attack:** Threat vector where rogue network devices or misconfigured printers pass back credentials via clear-text LDAP authentication requests to an attacker-controlled listener.

---

## PowerShell Scripting & Module Loading

PowerShell is built on the `.NET Framework` (`System.Management.Automation`).

```powershell
# Dot-sourcing loads functions into the current session scope
. C:\AD\Tools\PowerView.ps1

# Import-Module loads binary/script modules
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1

# Enumerate exported module cmdlets
Get-Command -Module ActiveDirectory
```

---

## PowerShell Security Bypasses & AV Evasion

> [!TIP]
> Modern EDR and Windows Defender monitor PowerShell execution via **AMSI (Antimalware Scan Interface)** and **Script Block Logging**.

### Signature & AMSI Evasion Concepts
- **Invisi-Shell:** Hooks .NET assemblies to bypass AMSI and PowerShell logging.
  - Admin execution: `RunWithPathAsAdmin.bat`
  - Non-admin execution: `RunWithRegistryNonAdmin.bat`
- **DefenderCheck:** Utility to identify exact offset locations flagged by Defender signatures (`DefenderCheck.exe <File>`).
- **Byte Patching / Obfuscation:** Modifying flagged strings or byte strings in scripts like `PowerUp.ps1` before execution.

```text
C:\AD\Tools\DefenderCheck.exe
C:\AD\Tools\PowerUp.ps1
```
