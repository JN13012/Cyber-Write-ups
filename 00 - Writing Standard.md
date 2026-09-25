---
category: writeup
platform: THM
last_verified: 2026-09-25
---
## 1. Purpose

This document defines the standard used for penetration-testing and CTF write-ups in this repository.

A write-up must serve three purposes:

1. **Reproducibility** — the attack path should be reproducible from the documented steps.
    
2. **Learning and revision** — the techniques should be explained well enough to recognize and reuse them in another lab or engagement.
    
3. **Professional presentation** — the document should demonstrate technical reasoning and methodology to another technical reader or recruiter.
    

A write-up should therefore be more than a list of commands, but it should not become a transcript of everything performed during the lab.

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

These steps do not need to appear as explicit headings every time. They should be visible naturally in the explanation.

---

# 2. File Naming

## Standard

Use:

```
NN - Room - Main Focus - Main Technique 1 - Main Technique 2 [- Main Technique 3].md
```

The filename should identify:

- the sequence number;
    
- the room or machine;
    
- the main technical focus;
    
- the most important techniques, vulnerabilities or exploitation mechanisms.
    

Examples:

```
01 - Recruit - Web Exploitation - Brute Force - Arbitrary File Read - SQL Injection.md

05 - Jump - Linux Privilege Escalation - Writable Script - PATH Hijacking - Sudo Abuse.md

06 - Jump - Windows Privilege Escalation - Credential Disclosure - Service Hijacking - Scheduled Task.md

07 - Proxy - Active Directory - NTLM Capture - Constrained Delegation - S4U.md

08 - Forward - Active Directory - Password Reuse - RBCD - S4U.md

09 - Domino - Web & Linux - IDOR - JWT Forgery - RCE - Cron PrivEsc.md
```

## Main Focus

Useful focus categories include:

```
Web Exploitation
Password Attacks
Linux Privilege Escalation
Windows Privilege Escalation
Active Directory
Web & Linux
```

The focus is useful because it immediately indicates the environment or type of skills exercised in the room.

## Tools vs techniques

Tool names should **normally not appear in filenames**.

Prefer:

```
Brute Force
```

instead of:

```
Hydra
```

Prefer:

```
Hash Cracking
```

instead of:

```
Hashcat
```

Prefer:

```
NTLM Capture
```

instead of:

```
Responder
```

Tools belong in the YAML metadata.

An exception is acceptable when the tool itself is the subject of the exercise, for example:

```
Metasploit Payload Generation
```

Do not try to list every technique used during the room. Keep only the concepts that best identify the attack path.

---

# 3. YAML Frontmatter

Every write-up should start with YAML frontmatter.

Recommended structure:

```
---
type: writeup
platform: TryHackMe
room: Forward
os: Windows
environment: Active Directory
status: completed
last_verified: 2026-09-25

techniques:
  - password-reuse
  - rbcd
  - kerberos-delegation
  - privilege-escalation

tools:
  - nmap
  - netexec
  - impacket
  - hashcat
  - xfreerdp
---
```

## Recommended fields

```
type:
platform:
room:
status:
last_verified:
techniques:
tools:
```

Use when relevant:

```
os:
environment:
difficulty:
```

Do not invent information such as a difficulty if it is unknown.

### `techniques`

Contains vulnerabilities, attack techniques or exploitation mechanisms:

```
techniques:
  - idor
  - jwt-forgery
  - credential-reuse
  - path-hijacking
  - rbcd
```

### `tools`

Contains the software used:

```
tools:
  - hydra
  - hashcat
  - responder
  - impacket
  - pspy
```

This separation prevents tools and techniques from being mixed in the filename.

---

# 4. Write-up Structure

A complete machine should generally follow this structure:

```
YAML frontmatter

# Title

## TL;DR
## Objective / Initial Context

# 1. Reconnaissance
# 2. Enumeration
# 3. Initial Access
# 4. Lateral Movement / Post-Exploitation
# 5. Privilege Escalation
# 6. Final Objective

## Attack Chain
## Key Findings
## False Leads / Useful Failures
## Key Takeaways
## Cleanup
```

This is a guideline, not a rigid template.

The actual sections should follow the attack path of the machine.

Do not create empty sections simply to respect the structure.

## TL;DR

A `TL;DR` is recommended for full-machine or multi-stage write-ups.

It should summarize the compromise path in a few lines:

```
Initial credentials
    ↓
Credential exposure
    ↓
Password reuse
    ↓
Dangerous AD ACL
    ↓
RBCD
    ↓
Kerberos impersonation
    ↓
SYSTEM
```

For focused exercises such as a single password-attack lab or a Metasploit exercise, the TL;DR is optional.

## Objective / Initial Context

Clearly state:

- the starting position;
    
- any credentials supplied by the scenario;
    
- the target environment;
    
- the final objective.
    

This prevents scenario-provided information from being confused with information discovered during exploitation.

---

# 5. Writing Rules

## Explain why, not only what

Important commands should have a reason for being executed.

Weak:

```
getcap -r / 2>/dev/null
```

Better:

> `sudo -l` did not reveal a usable privilege-escalation path, so Linux capabilities were enumerated next. Capabilities can grant privileged operations to binaries without requiring the SUID bit.

```
getcap -r / 2>/dev/null
```

The reader should understand both:

```
what was tested
```

and:

```
why it was tested at that point
```

## Do not over-explain basic commands

Commands such as:

```
ls
cd
cat
pwd
```

do not normally require detailed explanations.

Explain an option when it materially affects the technique or teaches something reusable.

Example:

```
-m 5600
→ Hashcat mode for NetNTLMv2
```

or:

```
-k
→ use Kerberos authentication

-no-pass
→ use the existing Kerberos ticket instead of requesting a password
```

## Distinguish hypotheses from facts

Never present an assumption as a confirmed finding.

Example:

```
DB_PASS = <REDACTED>
system_user = devops
```

Correct reasoning:

> The naming of the database password suggests possible credential reuse with the Linux account `devops`. This is only a hypothesis at this stage.

Test:

```
ssh devops@<TARGET_IP>
```

Only after successful authentication:

> Credential reuse is confirmed.

Use the mental model:

```
Observation
→ Hypothesis
→ Test
→ Confirmation / Rejection
```

## Prefer minimal validation tests

Use the smallest test that proves the hypothesis.

For command injection:

```
date;id
```

is better as an initial proof than immediately deploying a reverse shell.

For RCE:

```
id
```

immediately identifies the execution context.

Minimal tests:

- reduce unnecessary target modification;
    
- make the reasoning clearer;
    
- separate validation from exploitation.
    

## Interpret important results

Do not leave important output unexplained.

Example:

```
-rwxrwxr-- root devops health_report.sh
```

Interpretation:

> The script belongs to `root`, but members of the `devops` group can modify it. This is only exploitable if a more privileged context later executes the script.

The important question is not only:

```
Can I modify this?
```

but also:

```
Who executes or trusts it?
```

## Keep useful output only

Do not paste hundreds of lines of scanner or process output.

Keep the evidence supporting the conclusion.

Example:

```
UID=0 CMD=/bin/sh -c /opt/monitoring/health_report.sh
UID=0 CMD=/bin/bash /opt/monitoring/health_report.sh
```

Then explain:

> `UID=0` confirms that the script is executed as `root`.

Keep detailed raw output when necessary for understanding:

- ACL entries;
    
- sudo rules;
    
- service permissions;
    
- Kerberos delegation attributes;
    
- authentication response differences;
    
- important token or hash structure.
    

## Keep useful failures

Failed attempts should remain when they teach something or influence the next decision.

Good examples:

```
Hydra returned a false positive because the failure matcher was incomplete.

Kerberoasting produced a valid hash, but RockYou was exhausted without recovering the password.

JWT secret cracking failed, which led to testing the server-side validation logic.

A suspected blind XSS callback was later shown to be a simple HTTP fetch rather than JavaScript execution.
```

Do not preserve every unsuccessful command.

The goal is to document **useful reasoning**, not terminal history.

## Explain concepts without breaking the narrative

Definitions are useful for revision, especially for Active Directory, Kerberos, Windows internals and privilege escalation.

When possible, use Obsidian callouts:

```
> [!info] RootDSE
> The RootDSE is the special LDAP entry at the root of the directory.
> It exposes information such as naming contexts and server capabilities.
```

This separates:

```
what happened
```

from:

```
why the underlying mechanism works
```

---

# 6. Technical Accuracy & Reproducibility

A write-up should be technically precise enough to reproduce the attack without relying on the original terminal history.

## Reproducibility

Every variable, file or artifact required later must be introduced first.

Bad:

```
curl -H "Authorization: Bearer $ADMIN_TOKEN" ...
```

if `$ADMIN_TOKEN` was never created.

The same rule applies to:

```
cookies.txt
hash.txt
wordlists
payload files
ccache files
KRB5CCNAME
machine accounts
SSH keys
```

A reader should not need to guess how an artifact appeared.

## Use accurate vulnerability names

Describe the observed mechanism, not only the label used by the room.

For example:

```
file_get_contents($path);
```

is generally better described as:

```
Arbitrary File Read
```

than classic LFI if no PHP inclusion occurs.

By contrast:

```
include($path);
```

may correspond to Local File Inclusion.

Likewise:

```
file_get_contents($url);
eval($content);
```

can result in remote code execution because attacker-controlled remote content is evaluated.

When the lab terminology differs from the actual implementation, mention both and explain the distinction.

## Authentication vs authorization

For access-control vulnerabilities, distinguish the two explicitly.

Example:

```
Robert is authenticated.

Changing:

id=4

to:

id=1

returns another user's profile.
```

The problem is therefore not authentication.

The missing control is:

```
Is Robert authorized to access object 1?
```

This confirms IDOR/BOLA.

## Privilege escalation

Focus on the trust boundary rather than memorizing the payload.

Generic model:

```
Low-privileged user controls X
        +
Privileged identity executes or trusts X
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
service running as another user
```

```
Writable directory earlier in PATH
+
privileged process calls a command without an absolute path
```

## Identity transitions

Whenever the attack moves to a new account or privilege level, document:

```
Current identity
→ discovered credential / trust
→ validation
→ new identity
```

Verify the new context:

Linux:

```
whoami
id
```

Windows:

```
whoami
whoami /groups
```

Active Directory write-ups should also explain:

- who requests a ticket;
    
- which user is impersonated;
    
- which SPN is targeted;
    
- why delegation permits it;
    
- which ticket is produced;
    
- how that ticket is later used.
    

---

# 7. Redaction & Publication

The public repository should demonstrate methodology, not publish answers or reusable secrets.

## Flags

Do not publish real flags.

Use:

```
THM{REDACTED}
```

or:

```
EPI{REDACTED}
```

Keep:

- the file location;
    
- the command used to retrieve it;
    
- the privilege level required.
    

Example:

```
cat /root/root.txt
```

Result:

```
THM{REDACTED}
```

## Credentials and secrets

Discovered credentials should normally be redacted:

```
Username: devops
Password: <REDACTED>
```

Do not publish:

- private keys;
    
- API tokens;
    
- session cookies;
    
- JWT signing secrets;
    
- reusable access tokens;
    
- customer or real-environment information.
    

If part of a secret is important to the reasoning, describe only the relevant characteristic.

Example:

> The database password contained the string `D3v0ps`, which suggested possible reuse with the `devops` Linux account.

## IP addresses

Temporary lab IP addresses should normally be replaced by:

```
<TARGET_IP>
<ATTACKER_IP>
<DC_IP>
```

This keeps the write-up reusable after the machine is restarted.

## Screenshots

Use screenshots only when they add information that is difficult to reproduce as text, for example:

- graphical application behavior;
    
- KeePass configuration;
    
- RDP-only discoveries;
    
- unusual UI behavior.
    

Prefer text for:

- terminal commands;
    
- scanner output;
    
- flags;
    
- simple configuration files.
    

Sanitize screenshots before publication.

---

# 8. Final Checklist

Before considering a write-up complete:

```
[ ] The filename identifies the room, main focus and key techniques

[ ] Tools and techniques are separated correctly

[ ] YAML frontmatter is complete

[ ] A TL;DR is present for a full-machine or multi-stage lab

[ ] The starting context and objective are clear

[ ] Important commands explain why they were used

[ ] Hypotheses are distinguished from confirmed facts

[ ] Important results are interpreted

[ ] Required variables, files and artifacts are introduced before use

[ ] Vulnerabilities are classified according to the actual mechanism

[ ] Useful false leads and failed attempts are preserved

[ ] Identity / privilege transitions are clearly demonstrated

[ ] The full attack path can be reproduced from the write-up

[ ] Real flags, credentials and sensitive material are redacted

[ ] Cleanup is documented when the target was modified

[ ] Markdown renders correctly and code fences are closed

[ ] The final Key Takeaways contain reusable lessons, not only a summary
```

The final test is:

```
Could I reproduce and explain this attack several months later
using only this document?
```

If the answer is no, the write-up is not finished.