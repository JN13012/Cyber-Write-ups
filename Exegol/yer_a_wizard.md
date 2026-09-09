# Yer a Wizard — Write-up

> État : accès **user** obtenu. La partie *privilege escalation* (root) sera ajoutée ultérieurement.

## 1. Reconnaissance

Scan des services exposés sur la cible :

```bash
nmap -sC -sV 10.10.0.5 -o notes.txt
```

Deux ports ouverts ont été identifiés :

```bash
21/tcp open  ftp
22/tcp open  ssh
```

Paramètres utilisés :

* `-sC` : lance les scripts NSE par défaut de Nmap.
* `-sV` : détecte les services et leurs versions.

Le port **21 (FTP)** est la porte d'entrée la plus prometteuse, car il autorise potentiellement une connexion anonyme.

## 2. Énumération FTP

Connexion au service FTP en tant qu'utilisateur anonyme :

```bash
ftp 10.10.0.5
Name: anonymous
Password: [vide]
```

Le login anonyme est accepté (code de retour `230`).

Listing du contenu, y compris les fichiers cachés :

```bash
ls -la
```

Des éléments cachés ont été découverts. Le dossier `...` (facile à rater, il ressemble à `..`) contient les fichiers cachés. On y entre puis on liste :

```bash
cd ...
ls -la
```

On y trouve `.hidden` et `.reallyHidden`.

Contenu de `.hidden` :

```text
I swear that my password is Il0veTheMalefoys, trust me, I'm Hagrid, I never lie!
```

Ce mot de passe est une **fausse piste** (le message se moque de la victime : « I never lie » est ironique).

Contenu de `.reallyHidden` :

```text
FINE! My password is IAlreadySaidTooMuch
```

Le vrai mot de passe de l'utilisateur `hagrid` est donc `IAlreadySaidTooMuch`.

## 3. Accès SSH

Connexion via SSH avec les identifiants récupérés :

```bash
ssh hagrid@10.10.0.5
Password: IAlreadySaidTooMuch
```

Le mot de passe est bien celui trouvé dans `.reallyHidden` : il est réutilisé tel quel pour le compte SSH `hagrid`.

Listing du répertoire personnel :

```bash
ls -la
```

```text
drwxr-xr-x 1 hagrid hagrid     4096 .
drwxr-xr-x 1 root   root       4096 ..
-rw-r--r-- 1 hagrid hagrid      220 .bash_logout
-rw-r--r-- 1 hagrid hagrid     3526 .bashrc
-rw-r--r-- 1 hagrid hagrid      807 .profile
drwxr-xr-x 1 hagrid headmaster 4096 .ssh
drwxr-xr-x 1 hagrid hagrid     4096 hut
---------- 1 hagrid hagrid     2097 riddle.txt
-rw-r--r-- 1 hagrid hagrid       97 user.txt
```

## 4. Récupération du user flag

Lecture du fichier `user.txt` :

```bash
cat user.txt
```

```text
VWxaQ1NtVjZRblZOTVRseVdWVTFabUpxVGpKTk1VcG1ZVWRHVjAweE9IcGlha0pXVDFWb1prNVVRa1JUZWtvNVEyYzlQUT09
```

Le contenu n'est pas le flag directement : c'est une chaîne encodée en **plusieurs couches de Base64 empilées** (on le reconnaît à l'alphabet Base64 et aux `=` de padding ; chaque décodage révèle encore du Base64).

Décodage réalisé avec l'outil en ligne **DCode** (dcode.fr), qui détecte le type d'encodage et permet de décoder les couches successives de Base64 les unes après les autres jusqu'à obtenir du texte clair.

> Équivalent en ligne de commande (pour vérifier / reproduire) : réinjecter la sortie dans `base64 -d` à chaque couche, par exemple
> `echo -n "VWxaQ1..." | base64 -d` puis re-décoder le résultat obtenu, autant de fois que nécessaire, jusqu'au flag.

Flag obtenu :

```text
EPI{0n3_kaN_n3v3R_haV3_3n0U9H_50CK2}
```

## Méthodologie

```text
Reconnaissance (nmap)
        ↓
Énumération FTP (login anonyme)
        ↓
Découverte de fichiers cachés (... / .hidden / .reallyHidden)
        ↓
Fausse piste écartée (.hidden) → vrai mot de passe (.reallyHidden)
        ↓
Accès SSH en tant que hagrid
        ↓
Lecture de user.txt (Base64 multi-couches)
        ↓
Décodage → user flag
```

## Vulnérabilités identifiées

**1. Connexion FTP anonyme autorisée**
* *Composant affecté :* service FTP (port 21).
* *Impact :* n'importe qui peut se connecter sans authentification et lister/télécharger des fichiers.
* *Risque :* exposition de données sensibles (ici, des identifiants). C'est souvent le premier point d'entrée d'une compromission.
* *Remédiation :* désactiver l'accès anonyme (`anonymous_enable=NO` dans `vsftpd.conf`), ou restreindre strictement les fichiers accessibles.

**2. Identifiants stockés en clair dans des fichiers accessibles**
* *Composant affecté :* fichiers `.hidden` / `.reallyHidden` sur le partage FTP.
* *Impact :* le mot de passe d'un compte système (`hagrid`) est lisible par un utilisateur anonyme.
* *Risque :* réutilisation directe du mot de passe pour un accès SSH → prise de contrôle du compte.
* *Remédiation :* ne jamais stocker de secrets en clair ; retirer ces fichiers du partage ; imposer une politique de mots de passe et éviter la réutilisation d'un mot de passe entre services.

**3. Réutilisation d'un mot de passe FTP → SSH**
* *Composant affecté :* compte utilisateur `hagrid`.
* *Impact :* le même secret ouvre plusieurs services.
* *Risque :* un seul secret exposé compromet plusieurs points d'accès.
* *Remédiation :* mots de passe distincts par service ; préférer l'authentification SSH par clé.

## Notes pour la soutenance

* La difficulté « pédagogique » ici est l'**attention au détail** : le dossier `...` imite `..` et les fichiers cachés (`.`) n'apparaissent qu'avec `ls -la`.
* Question probable de l'évaluateur : *pourquoi un login FTP anonyme est-il dangereux ?* → voir vulnérabilité 1.
* Point théorique : différence **encodage vs chiffrement vs hachage** — ici c'est de l'**encodage** (Base64), réversible sans clé, donc ce n'est pas une protection.
