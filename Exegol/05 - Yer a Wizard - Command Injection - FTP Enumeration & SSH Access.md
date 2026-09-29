---
type: writeup
platform: Exegol
room: Yer a Wizard
os: Linux
environment: FTP / SSH

techniques:
  - anonymous-ftp
  - hidden-file-enumeration
  - credential-discovery
  - credential-validation
  - ssh-access
  - data-decoding

tools:
  - nmap
  - ftp
  - ssh
---
**Attack path:** Anonymous FTP → hidden directory → credential disclosure → SSH as `hagrid` → multi-layer Base64 decoding → user flag

> The available source for this machine documents the compromise up to the user-level objective. No root privilege-escalation path is preserved in the current notes.

## 1. Reconnaissance

Initial service enumeration was performed with:

```bash
nmap -sC -sV <TARGET_IP> -o notes.txt
```

Two exposed services were identified:

```bash
21/tcp open  ftp
22/tcp open  ssh
```

With no credentials available yet, FTP was tested first for anonymous access.

---

## 2. Anonymous FTP Access

The FTP service accepted anonymous authentication:

```bash
ftp <TARGET_IP>
```

```bash
Name: anonymous
Password: [empty]
```

The server returned a successful login response.

A listing including hidden entries was performed:

```bash
ls -la
```

Among the entries was an unusual directory literally named:

```bash
...
```

This was easy to overlook because it visually resembled the standard parent-directory entry:

```bash
..
```

The directory was entered and enumerated:

```bash
cd ...
ls -la
```

Two hidden files were discovered:

```bash
.hidden
.reallyHidden
```

---

## 3. Credential False Lead

The first file contained a message claiming to disclose Hagrid's password:

```bash
I swear that my password is <REDACTED>, trust me, I'm Hagrid, I never lie!
```

At this stage, the value was only a **candidate credential**. The wording itself also suggested that it could be misleading.

The second file provided another candidate:

```bash
FINE! My password is <REDACTED>
```

Rather than assuming either value was valid, the second credential was tested against the SSH service discovered during reconnaissance.

---

## 4. SSH Access as `hagrid`

The credential from `.reallyHidden` was tested with:

```bash
ssh hagrid@<TARGET_IP>
```

Authentication succeeded.

This confirmed that the disclosed value was valid for the local account:

```bash
hagrid
```

The home directory was then enumerated:

```bash
ls -la
```

Relevant entries included:

```bash
drwxr-xr-x 1 hagrid headmaster ... .ssh
drwxr-xr-x 1 hagrid hagrid     ... hut
---------- 1 hagrid hagrid     ... riddle.txt
-rw-r--r-- 1 hagrid hagrid     ... user.txt
```

The successful SSH login established the transition from anonymous FTP access to an authenticated local user context.

---

## 5. Encoded User Flag

The contents of `user.txt` were inspected:

```bash
cat user.txt
```

Instead of directly containing an `EPI{...}` value, the file contained a Base64-looking string:

```bash
VWxaQ1NtVjZRblZOTVRseVdWVTFabUpxVGpKTk1VcG1ZVWRHVjAweE9IcGlha0pXVDFWb1prNVVRa1JUZWtvNVEyYzlQUT09
```

The preserved notes record that the value was decoded with **DCode**.

Each decoding layer produced another Base64 value, so the process was repeated until plaintext was recovered.

A command-line equivalent for reproducing the same approach is:

```bash
echo -n '<BASE64_DATA>' | base64 -d
```

The resulting output can then be passed through `base64 -d` again until the final plaintext is reached.

The decoded user flag was:

```bash
EPI{REDACTED}
```

---

## Key Takeaways

- Anonymous FTP access can expose sensitive files even when the initial directory contents appear harmless.
- Hidden-file enumeration matters; the useful artifacts were located behind a directory named `...` and files beginning with `.`.
- A discovered password should remain a candidate until it is validated against an actual authentication service.
- False leads should be tested rather than assumed to be valid or invalid solely from their wording.
- Base64 is reversible encoding, not encryption; multiple nested layers do not provide meaningful confidentiality.
- The documented source ends at user-level access, so no root compromise should be inferred from the available evidence.