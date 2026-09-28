---
category: writeup
platform: THM + Exegol
---
## Purpose

Write-ups in this repository document authorized labs, CTFs and penetration-testing exercises.

They should be useful for two audiences:

- a technical reader or recruiter who wants to understand the compromise quickly;
- later revision, where the reasoning and important technical mechanisms must still be understandable and reproducible.

A write-up is not a terminal transcript and should not become a complete theoretical course.

The objective is simple:

> Explain what happened, why each important decision was made, what the evidence means, and how the attack progressed.

Prefer the **shortest explanation that still makes the reasoning and technical mechanism clear**.

---

## Language

Each completed write-up should have:

- an **English version**, which is the canonical version;
- a **French version**, translated from the final English version.

Both versions should contain the same:

- structure;
- commands;
- technical evidence;
- attack path;
- redactions;
- level of detail.

Do not independently rewrite the French version.

Finalize and verify the English version first, then translate it.

---

## File Naming and Titles

Keep filenames and document titles concise.

Use:

```
NN - Room - Main Focus.md
```

Example:

```
02 - Support - Web Exploitation.md
```

The H1 should follow the same principle:

```
# Support — Web Exploitation
```

Do not list every vulnerability or technique in the filename or H1.

Detailed techniques belong in:

- YAML metadata;
- the optional `Attack path`;
- the write-up itself.

---

## General Rules

- One continuous machine or scenario should normally produce one complete write-up per language.
- Existing multi-part write-ups should be merged when they describe the same compromise.
- Use exactly one H1 title.
- Adapt the sections to the real attack path.
- Do not force every machine into the same structure.
- Follow the attack chronologically.
- Do not invent commands, outputs, credentials, discoveries or historical reasoning.
- Do not repeat the same information in several summary sections.
- Explain important reasoning once, where it matters.

---

## Metadata

Each write-up starts directly with YAML frontmatter.

Example:

```
---
type: writeup
platform: TryHackMe
room: Support
os: Linux
environment: Web Application
last_verified: YYYY-MM-DD

techniques:
  - brute-force
  - cookie-tampering
  - idor
  - path-traversal
  - arbitrary-file-read
  - command-injection
  - remote-code-execution

tools:
  - nmap
  - gobuster
  - curl
  - burp-suite
  - hydra
---
```

Keep metadata minimal and useful.

Typical fields:

```
type
platform
room
os
environment
last_verified
techniques
tools
```

Do not add metadata unless it provides a real classification or retrieval benefit.

---

## Structure

The structure is intentionally flexible.

A typical full-machine write-up may look like:

```
# Room / Machine — Main Focus

**Attack path:** step → step → step → final access

## 1. Reconnaissance

## 2. Initial Access

## 3. <Attack Phase>

## 4. Privilege Escalation

## Key Takeaways

## Cleanup
```

The `Attack path` line is optional, but useful for longer or multi-stage machines.

Keep it short.

Example:

```
Helpdesk brute force → cookie tampering → IDOR → arbitrary file read → admin access → command injection → www-data
```

Do not add several sections that summarize the same attack.

Sections such as the following are optional and should only exist when they genuinely improve the write-up:

```
TL;DR
Objective / Starting Context
Key Findings
Cleanup
```

For example, `Starting Context` is useful for an assumed-breach Active Directory lab with provided credentials, but unnecessary for a simple machine starting without credentials.

---

## Follow the Attack Chronologically

The write-up should follow the order in which the attack was understood and developed.

Important reasoning should appear where it happened.

Use this mental model:

```
Observation
→ Hypothesis
→ Minimal Test
→ Result
→ Interpretation
→ Next Decision
```

These are not mandatory headings.

They should appear naturally in the explanation.

Useful failures and false leads should normally remain **inside the chronological narrative**.

Example:

```
The API exposed numeric user IDs, so SQL injection was briefly tested.

The malformed request returned no useful behavior, and adding an admin=true
cookie also had no effect.

Attention therefore returned to the dashboard functionality.
```

Do not move failures to a separate end section if doing so breaks the attack flow.

---

## Explain Clearly, but Stay Direct

Important commands should answer three questions:

```
Why was this tested?
What important result was obtained?
What changed because of that result?
```

Do not explain basic commands such as:

```
ls
cd
cat
pwd
whoami
```

unless their use is technically important.

Do not explain every command-line option.

Explain options only when they matter to understanding or reproducing the technique.

Show only useful output.

Avoid large scanner dumps, complete process listings or repetitive responses when a few lines prove the conclusion.

---

## Code Blocks

Use a language identifier on fenced code blocks whenever possible.

Use `bash` as the default for:

- Linux shell commands;
- terminal output;
- paths;
- raw values;
- credentials or redacted credentials;
- hashes;
- service enumeration results;
- generic CLI-oriented content.

Use a more specific language when the content has a clear format, such as:

```
powershell
json
php
http
python
html
javascript
sql
yaml
```

For example:

```
{
  "email": "help@support.thm",
  "admin": false
}
```

```
$requested = realpath($webRoot . '/' . $skin . '.php');
```

```
POST /dashboard.php HTTP/1.1

sys=date +"%H:%M:%S"
```

Do not use `text` as the default when `bash` or a more precise language is appropriate.

---

## Technical Explanation

Explain the mechanism that matters to the current attack.

Example:

> The application normalizes the requested path with `realpath()`, but the following boundary check still allows traversal outside `skins`. Because the file is returned with `readfile()` rather than included as PHP code, the resulting primitive is **Arbitrary File Read**, not classic LFI.

This is preferable to several paragraphs explaining `realpath()`, path traversal and PHP inclusion in general.

The write-up should remain centered on the machine.

A useful rule is:

> Explain enough theory to understand this attack, but no more than necessary.

---

## Accuracy and Evidence

Distinguish clearly between:

```
observed evidence
hypothesis
test
confirmed result
```

Do not present an assumption as a fact.

Use vulnerability terminology that matches the mechanism actually observed.

Examples of distinctions that matter:

```
authentication vs authorization
path traversal vs arbitrary file read
file read vs file inclusion
credential discovery vs confirmed credential reuse
writable file vs exploitable privileged execution path
```

For privilege escalation, explain both:

```
What can the current user control?
Who executes or trusts it with greater privileges?
```

When moving to another identity or privilege level, make the transition clear and verify it.

---

## Historical Integrity

Never invent missing lab history.

If the exact historical command was not preserved, say so.

A reproducible equivalent may be provided:

> The exact historical enumeration command was not preserved. The following command reproduces the same discovery approach.

Do not present reconstructed commands as commands that were definitely executed during the original lab.

Important artifacts used later should be introduced before use, such as:

```
cookies
wordlists
hash files
payloads
Kerberos ticket caches
SSH keys
machine accounts
```

---

## Redaction

Public write-ups should demonstrate methodology without unnecessarily publishing challenge answers or reusable secrets.

Redact flags:

```
THM{REDACTED}
EPI{REDACTED}
```

Redact reusable secrets when appropriate:

```
Password: <REDACTED>
JWT secret: <REDACTED>
```

Do not publish private keys, tokens, session cookies or similar reusable secrets.

Temporary lab IP addresses should normally use:

```
<TARGET_IP>
<ATTACKER_IP>
<DC_IP>
```

Stable hostnames or domain names may remain when technically relevant.

---

## Key Takeaways

End with a few reusable lessons when the machine provides useful ones.

Keep them short and technical.

Good:

> A writable file becomes a privilege-escalation primitive only when a more privileged context executes or trusts it.

Avoid simply repeating the attack path.

---

## Final Check

Before publishing, verify:

```
[ ] The filename and H1 are concise.

[ ] The write-up follows the real attack chronologically.

[ ] The explanation is concise but sufficient to understand the reasoning.

[ ] Important commands have a clear purpose and interpreted result.

[ ] Code blocks use bash by default or a more specific language when appropriate.

[ ] Hypotheses are distinguished from confirmed facts.

[ ] Useful failures appear where they influenced the attack.

[ ] Technical terminology matches the observed mechanism.

[ ] No historical command or result was invented.

[ ] Repeated summaries and unnecessary theory were removed.

[ ] Flags and sensitive secrets are redacted.

[ ] The English and French versions contain the same technical content.

[ ] The document remains useful for both quick reading and later revision.
```

The final writing principle is:

> **Be direct, but not superficial. Explain the reasoning once, at the point where it matters.**