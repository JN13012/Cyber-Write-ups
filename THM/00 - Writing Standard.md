## 1. Purpose

This document defines the standard used for all penetration-testing and CTF write-ups published in this repository.

A write-up has three objectives:

1. **Reproducibility** — allow the attack path to be reproduced from the documented steps.
    
2. **Learning and revision** — explain the techniques well enough to reuse them in another lab or engagement.
    
3. **Professional presentation** — demonstrate methodology, technical reasoning and understanding to a technical reader or recruiter.
    

A write-up must therefore be more than:

```
a list of commands
```

but should also avoid becoming:

```
a transcript of every action performed during the lab
```

The expected reasoning model is:

```
Observation
    ↓
Hypothesis
    ↓
Minimal test
    ↓
Result
    ↓
Interpretation
    ↓
Next decision
```

This reasoning does not need to appear as six explicit headings for every step, but it should be visible in the narrative.

---

# 2. File Naming

## Standard

```
NN - Room - Technique 1 - Technique 2 - Technique 3 [- Technique 4].md
```

Examples:

```
01 - Recruit - Hydra - LFI - SQLi.md

06 - Jump Windows - SMB - Winlogon - Service Hijacking - Scheduled Task.md

08 - Forward - KeePass - Password Spray - RBCD - S4U.md

09 - Domino - IDOR - JWT Forgery - RCE - Cron PrivEsc.md
```

## Rules

The filename should contain:

- the sequence number;
    
- the room or machine name;
    
- three to five important techniques, vulnerabilities or distinctive tools.
    

Do not attempt to list everything used during the lab.

For example, a room containing:

```
Nmap
SMB
LDAP
RDP
KeePass
John
Hashcat
Password Spray
ACL
RBCD
S4U
SMBExec
```

should not have all these elements in its title.

Prefer the elements that best identify the attack path:

```
Forward - KeePass - Password Spray - RBCD - S4U
```

Generic tools such as `Nmap`, `curl`, `grep` or `ls` normally do not belong in the title.

They should appear in the YAML metadata instead.

---

# 3. YAML Frontmatter

Every write-up must start with YAML frontmatter.

Recommended structure:

```
---
type: writeup
platform: TryHackMe
room: Forward
os: Windows
environment: Active Directory
difficulty: medium
status: completed
date: 2026-09-17
scope: authorized-lab

techniques:
  - credential-access
  - password-spraying
  - rbcd
  - kerberos-delegation
  - privilege-escalation

tools:
  - nmap
  - netexec
  - impacket
  - ldapsearch
  - hashcat
  - xfreerdp

tags:
  - pentest/writeup
  - tryhackme
  - active-directory
---
```

## Mandatory fields

```
type:
platform:
room:
status:
date:
scope:
techniques:
tools:
tags:
```

Recommended values:

```
type: writeup
status: completed
scope: authorized-lab
```

## Context-dependent fields

Use only when applicable:

```
os:
environment:
difficulty:
```

Examples:

```
os: Linux
```

```
environment: Active Directory
```

or:

```
environment: Web
```

Do not invent a difficulty if none is known.

---

# 4. Language and Terminology

The explanatory text may be written in French.

Technical terminology should generally retain the terminology used by the technology or security community.

Prefer:

```
Remote Code Execution
Privilege Escalation
Password Spraying
Credential Reuse
Path Traversal
Arbitrary File Read
Resource-Based Constrained Delegation
```

rather than forcing translations that make the concept harder to search or recognize.

Commands, protocol names, Windows/Linux objects and source-code identifiers must remain unchanged.

Examples:

```
S4U2Self
msDS-AllowedToActOnBehalfOfOtherIdentity
NT AUTHORITY\SYSTEM
KRB5CCNAME
```

---

# 5. Required High-Level Structure

A full write-up should normally use the following structure.

```
YAML frontmatter

Title

TL;DR

Objective / Initial Context

Attack phases
  Reconnaissance
  Enumeration
  Initial Access
  Lateral Movement
  Privilege Escalation
  Post-Exploitation
  ...

Attack Chain

Vulnerabilities Identified

False Leads / Useful Failures

Key Takeaways

Cleanup
```

Not every section is required for every lab.

For example, a password-attack room may have no privilege escalation.

Sections that do not apply should be omitted rather than kept empty.

---

# 6. TL;DR

Every full-machine write-up should begin with a short `TL;DR`.

Its purpose is to allow the reader to understand the complete compromise path without reading the entire document.

Example:

```
Anonymous SMB share
    ↓
Internal credentials
    ↓
Password reuse
    ↓
Dangerous AD ACL
    ↓
RBCD
    ↓
S4U2Self / S4U2Proxy
    ↓
Administrator service ticket
    ↓
SYSTEM
```

The TL;DR should normally fit on one screen.

It should describe the attack path, not all commands used.

---

# 7. Objective and Initial Context

Clearly state:

- the starting position;
    
- known credentials, if provided by the scenario;
    
- the target type;
    
- the objective.
    

Example:

```
The room follows an assumed-breach scenario.

Starting access:

Domain: ctf.local
User: j.smith
Password: <REDACTED>

Objective:

j.smith
→ lateral movement
→ privilege escalation
→ Administrator / SYSTEM
```

If credentials are part of the official starting scenario, they may be documented unless publication rules require otherwise.

Credentials discovered during exploitation should normally be redacted in the public repository.

---

# 8. Target Addresses

Avoid permanently embedding temporary lab addresses throughout the write-up.

Prefer placeholders:

```
<TARGET_IP>
<ATTACKER_IP>
<DC_IP>
```

Example:

```
nmap -sC -sV <TARGET_IP>
```

If the actual IP is useful during the lab, it can be defined once:

```
export TARGET_IP=10.10.10.10
export ATTACKER_IP=10.10.10.20
```

The write-up itself should remain reusable after the machine is restarted.

---

# 9. Reconnaissance

Reconnaissance should establish the attack surface.

Do not simply paste complete tool output.

Prefer:

```
nmap -sC -sV <TARGET_IP>
```

followed by the relevant findings:

```
22/tcp   SSH
80/tcp   HTTP
445/tcp  SMB
```

Then explain the consequence:

```
SMB is the most promising initial surface because it may expose shares,
users or authentication material. SSH will become relevant if credentials
are later recovered.
```

## Keep

- relevant ports;
    
- interesting service versions;
    
- domain/hostname information;
    
- unusual headers;
    
- paths influencing the next decision.
    

## Remove or shorten

- hundreds of irrelevant Nmap lines;
    
- repeated tool banners;
    
- progress percentages;
    
- output unrelated to the attack path.
    

---

# 10. Explain Why a Command Is Used

Important commands must not appear without context.

Weak:

```
getcap -r / 2>/dev/null
```

Better:

````
No sudo path was available, so Linux capabilities were enumerated next.
Capabilities may grant privileged operations to otherwise non-SUID
executables.

```bash
getcap -r / 2>/dev/null
````

````

The reader should understand both:

```text
what the command does
````

and:

```
why it was appropriate at that point
```

---

# 11. Command Explanations

Do not explain every universally obvious shell token.

For example, this is usually unnecessary:

```
cat = display a file
ls = list directory contents
cd = change directory
```

unless the command is part of a beginner-oriented concept being learned.

Explain parameters when:

- they materially affect the attack;
    
- they are easy to misunderstand;
    
- the command is unusual;
    
- the option teaches a reusable technique.
    

Good example:

```
-k
→ use Kerberos authentication

-no-pass
→ do not request a password; use the ticket referenced by KRB5CCNAME
```

Another example:

```
-m 5600
→ Hashcat mode for NetNTLMv2
```

---

# 12. Hypothesis vs Confirmed Fact

Never present a hypothesis as a confirmed fact.

Example:

A configuration file reveals:

```
DB_PASS = <REDACTED>
system_user = devops
```

Correct wording:

```
The naming of the database password suggests possible credential reuse
with the Linux account `devops`.

This is only a hypothesis at this stage.
```

Then test:

```
ssh devops@<TARGET_IP>
```

Only after successful authentication:

```
Credential reuse is confirmed.
```

The distinction should always be:

```
Observation
→ Hypothesis
→ Test
→ Confirmation / Rejection
```

---

# 13. Minimal Tests Before Exploitation

Prefer small tests that validate one assumption at a time.

Example for command injection:

```
Hypothesis:
The `sys` parameter may be passed to a shell.

Minimal test:

date;id
```

If the response contains:

```
uid=33(www-data)
```

then command execution is confirmed.

Do not immediately deploy a reverse shell when:

```
id
```

is enough to prove RCE.

This makes the reasoning easier to understand and reduces unnecessary changes to the target.

---

# 14. Results Must Be Interpreted

A command result should not be left unexplained.

Example:

```
-rwxrwxr-- root devops health_report.sh
```

Explain:

```
The script belongs to root but the `devops` group has write permission.

Since the current user belongs to `devops`, the script can be modified.

This becomes exploitable only if a more privileged context executes it.
```

Then verify execution separately.

This is more useful than simply labelling the file:

```
vulnerable
```

---

# 15. Authorization Vulnerabilities

When documenting IDOR/BOLA or access-control flaws, explicitly distinguish:

```
authentication
```

from:

```
authorization
```

Example:

```
Robert is legitimately authenticated.

Changing:

id=4

to:

id=1

returns Laura's profile.

The server therefore verifies that the requester is logged in but does
not verify whether that user may access the requested object.

IDOR/BOLA confirmed.
```

---

# 16. Technical Classification Must Match the Mechanism

Use the most technically accurate vulnerability name supported by the observed implementation.

For example:

```
file_get_contents($path);
```

is generally better described as:

```
Arbitrary File Read
Local File Disclosure
```

than classic:

```
LFI
```

if no PHP inclusion occurs.

Likewise:

```
file_get_contents($url);
eval($content);
```

may lead to:

```
Remote File Inclusion / Remote Code Inclusion
→ Remote Code Execution
```

The write-up may mention the terminology used by the lab, but should clarify the actual mechanism when different.

---

# 17. Secrets and Credentials

For a public repository, redact discovered secrets.

Use:

```
Username: devops
Password: <REDACTED>
```

or:

```
DB_PASS = <REDACTED>
```

Do not publish reusable:

- passwords;
    
- private keys;
    
- API tokens;
    
- session cookies;
    
- JWT secrets;
    
- hashes when disclosure is unnecessary.
    

If the secret itself is important to understanding the technique, describe its nature rather than its complete value.

Example:

```
The database password contained the string `D3v0ps`, which suggested
possible reuse for the `devops` Linux account.
```

---

# 18. Flags

Real flags must not be published in the public write-up.

Use:

```
THM{REDACTED}
```

or:

```
EPI{REDACTED}
```

However, keep:

- the location;
    
- the privilege level required;
    
- the command used to retrieve it.
    

Example:

```
cat /root/root.txt
```

Result:

```
THM{REDACTED}
```

This documents successful compromise without publishing the answer.

---

# 19. Session Material and Variables

No variable may appear in an exploitation command without being created earlier.

Bad:

```
curl -H "Authorization: Bearer $ADMIN_TOKEN" ...
```

if the write-up never showed where `$ADMIN_TOKEN` came from.

Correct:

```
TOKEN=$(...)
```

then:

```
ADMIN_TOKEN=$(...)
```

then later:

```
curl \
-H "Authorization: Bearer $ADMIN_TOKEN" \
...
```

The same applies to:

```
cookies.txt
hash.txt
KRB5CCNAME
ccache files
machine accounts
wordlists
payload files
```

A reader should not need missing information from the original terminal history.

---

# 20. Tool Output

Output should be reduced to the lines that support the conclusion.

Instead of:

```
500 lines of pspy output
```

keep:

```
UID=0 CMD=/bin/sh -c /opt/monitoring/health_report.sh
UID=0 CMD=/bin/bash /opt/monitoring/health_report.sh
```

Then explain:

```
UID 0 confirms that root executes the script.
```

Keep raw output when it is essential for understanding:

- ACL entries;
    
- sudo rules;
    
- service permissions;
    
- Kerberos delegation attributes;
    
- HTTP response differences;
    
- hashes or token structure when relevant.
    

---

# 21. False Leads and Failed Attempts

Useful failed attempts should be preserved.

Examples:

```
Kerberoasting succeeded but the hash was not crackable with the selected wordlist.

Hydra returned a false positive because the failure matcher was incomplete.

JWT secret cracking failed, leading to validation testing instead.

A suspected blind XSS callback was later proven to be only an HTTP URL fetcher.
```

These demonstrate reasoning and teach reusable lessons.

Do not preserve failures that provide no useful information.

Bad:

```
Tried command A.
Didn't work.

Tried command B.
Didn't work.

Tried command C.
Didn't work.
```

Good:

```
RockYou was exhausted without recovering the service password.

Given the lab time constraint and lack of evidence for a weak password,
further brute force was abandoned and enumeration resumed.
```

---

# 22. Automated Tools Must Be Verified

Never trust an automated result without understanding its success criteria.

For Hydra:

```
What indicates failure?
What indicates success?
Does the application return different status codes?
Does a redirect occur?
Is a session cookie created?
```

For scanners:

```
What exactly triggered the finding?
Can it be reproduced manually?
```

For privilege-escalation scripts:

```
Which exact permission or configuration creates the vulnerability?
```

The write-up should document the underlying condition, not just:

```
LinPEAS says vulnerable
```

---

# 23. Lateral Movement

When moving from one identity to another, clearly document:

```
Current identity
→ discovered trust/credential
→ validation
→ new identity
```

Example:

```
www-data
   ↓
database configuration disclosure
   ↓
credential-reuse hypothesis
   ↓
SSH authentication
   ↓
devops
```

Always verify the new context:

```
whoami
id
```

or on Windows:

```
whoami
whoami /groups
```

---

# 24. Privilege Escalation

Privilege escalation sections should emphasize the trust boundary.

Generic model:

```
Low-privileged user controls X
        +
Privileged identity executes/trusts X
        ↓
Privilege Escalation
```

Examples:

```
Writable script
+
root cron execution
```

```
Writable service executable
+
service runs as svcadmin
```

```
Writable directory earlier in PATH
+
privileged program calls command without absolute path
```

```
NOPASSWD interpreter
+
interpreter can execute arbitrary code
```

The mechanism is more important than memorizing a payload.

---

# 25. Linux Privilege-Escalation Baseline

When appropriate, useful initial checks include:

```
id
groups
sudo -l
```

then potentially:

```
find / -perm -4000 -type f 2>/dev/null
```

```
getcap -r / 2>/dev/null
```

and investigation of:

```
cron
systemd timers
custom services
writable privileged scripts
credentials
PATH usage
local-only network services
```

Do not blindly dump every possible enumeration command.

The write-up should explain why each branch was investigated.

---

# 26. Windows Privilege-Escalation Baseline

Useful areas include:

```
Identity and groups
Privileges
Stored credentials
Winlogon
Services
Service ACLs
Scheduled tasks
Writable executables/scripts
Registry
Local network services
```

The core reasoning should remain:

```
Who runs it?
+
What does it execute?
+
Can my current user modify any part of that chain?
```

---

# 27. Active Directory Write-ups

AD write-ups must explain identity and Kerberos transitions carefully.

Do not write only:

```
run getST.py
```

Explain:

```
who requests the ticket
which user is impersonated
which SPN is targeted
why delegation allows it
what ticket is produced
how the ticket is then consumed
```

Example:

```
ATTACKER$
    ↓ S4U2Self
Administrator identity
    ↓ S4U2Proxy
cifs/DC01.ctf.local
    ↓
Administrator CIFS TGS
```

For dangerous ACLs, show:

```
principal
→ right
→ target object
→ sensitive attribute
```

---

# 28. Theory and Definitions

Definitions are encouraged when they help revision.

However, they should not interrupt the attack narrative unnecessarily.

In Obsidian, prefer callouts:

```
> [!info] RootDSE
> The RootDSE is the special LDAP entry at the root of the directory.
> It exposes information such as naming contexts and server capabilities.
```

or:

```
> [!note] NetNTLMv2
> A captured NetNTLMv2 response is a challenge-response value, not the
> user's NT hash or plaintext password.
```

This visually separates:

```
what happened in the attack
```

from:

```
what the underlying concept means
```

---

# 29. Attack Chain

Every complete machine should include a final attack chain.

Example:

```
Anonymous SMB
    ↓
Credential disclosure
    ↓
User access
    ↓
Password reuse
    ↓
Dangerous AD ACL
    ↓
RBCD
    ↓
Kerberos impersonation
    ↓
Administrator ticket
    ↓
SYSTEM
```

This section should be concise.

Do not repeat the entire write-up.

---

# 30. Vulnerabilities Identified

When useful, summarize the vulnerabilities at the end.

Recommended format:

```
## 1. Broken Object-Level Authorization

Affected component:
`/api/users/profile.php?id=`

Cause:
The application verifies authentication but not ownership of the
requested user object.

Impact:
Authenticated users can retrieve other users' profiles.

Technique:
IDOR / BOLA
```

For larger write-ups, a table may be used:

|Stage|Finding|Impact|
|---|---|---|
|Web|IDOR/BOLA|access to other users|
|API|JWT signature not verified|admin authorization bypass|
|Host|credential reuse|SSH lateral movement|
|Linux|writable root cron script|root privilege escalation|

The purpose is technical review, not formal CVSS scoring unless explicitly required.

---

# 31. Key Takeaways

Every write-up should finish with reusable lessons.

Avoid conclusions such as:

```
The room was interesting and I learned a lot.
```

Prefer:

```
A writable root-owned file is not automatically exploitable.

The critical condition is:

attacker can modify the file
+
privileged context executes it
```

or:

```
A valid authentication session does not imply authorization to every
object exposed by an API.
```

These should be concepts applicable outside the room.

---

# 32. Cleanup

If the exploitation modified the target, include cleanup instructions.

Examples:

```
rm /tmp/rootbash
```

Restore modified scripts:

```
cp ~/health_report.sh.bak /opt/monitoring/health_report.sh
```

For Windows:

```
restore replaced service binary
restore modified scheduled-task script
remove uploaded payloads
```

For Active Directory:

```
remove attacker-created machine account
remove RBCD delegation entry
delete generated artifacts/tickets where applicable
```

Cleanup demonstrates understanding of real penetration-testing workflow.

---

# 33. Screenshots

Screenshots should be used only when they add information that is difficult to communicate as text.

Good candidates:

- graphical application functionality;
    
- RDP-only discoveries;
    
- KeePass configuration;
    
- unusual UI behavior.
    

Do not use screenshots for:

```
nmap output
simple terminal commands
one-line flags
basic text files
```

Text is easier to search, copy and review.

Any screenshot containing credentials, flags or personal information must be sanitized before publication.

---

# 34. Public Repository Safety

Before publishing, verify that the write-up contains no:

```
real reusable credentials
private keys
API secrets
JWT signing secrets
session cookies
access tokens
personal information
real engagement/customer data
unredacted flags
```

Lab-only credentials may still be redacted to keep the repository focused on methodology rather than answers.

---

# 35. One Machine, One Public Write-up

When a lab is split into multiple exercises such as:

```
machine_part_1.md
machine_part_2.md
```

prefer combining them for the public repository when they represent a single compromise path.

Public structure:

```
Reconnaissance
→ Initial Access
→ User
→ Lateral Movement
→ Privilege Escalation
→ Root
```

This gives the reader one complete attack narrative.

Private study notes may remain separated if useful.

---

# 36. Markdown Quality

Before publication verify:

- all code fences are closed;
    
- commands are syntactically valid;
    
- heading levels are consistent;
    
- numbering is sequential;
    
- tables render correctly;
    
- no accidental backslashes or shell syntax errors remain;
    
- placeholders are consistent;
    
- no duplicate sections remain.
    

Use language identifiers for code blocks when useful:

````
```bash
```

```powershell
```

```python
```

```php
```

```json
```

```text
```
````

---

# 37. Technical Accuracy Review

Before considering a write-up complete, answer these questions:

### Reproducibility

```
Could I reproduce the attack using only this document?
```

### Causality

```
Does every important step explain why the next one follows?
```

### Evidence

```
Is each claimed vulnerability actually demonstrated?
```

### Precision

```
Am I calling the vulnerability by the correct technical name?
```

### Assumptions

```
Have I distinguished hypotheses from confirmed facts?
```

### Secrets

```
Have all flags and sensitive values been redacted?
```

### Cleanup

```
Have attacker-created artifacts and target modifications been documented?
```

If one of these answers is no, the write-up is not finished.

---

# 38. Recommended Final Structure

A typical complete machine should therefore resemble:

```
---
YAML
---

# Room - Main Techniques

## TL;DR

## Objective

# 1. Reconnaissance

# 2. Enumeration

# 3. Initial Access

# 4. Post-Exploitation / Lateral Movement

# 5. Privilege Escalation

# 6. Root / Final Objective

# Attack Chain

# Vulnerabilities Identified

# False Leads / Useful Failures

# Key Takeaways

# Cleanup
```

This is a guideline, not a rigid form.

The structure should adapt to the actual attack path.

---

# 39. Quality Target

A finished write-up should demonstrate that the author can:

```
enumerate
→ analyze
→ form hypotheses
→ validate them
→ understand the vulnerability
→ exploit it
→ verify the new security context
→ re-enumerate
→ document the complete attack chain
→ clean up
```

The objective is not merely to prove that the room was completed.

The objective is to demonstrate that the technique was understood well enough to recognize and apply it again in another environment.