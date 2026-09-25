---
category:
  - writeup
platform: THM
last_verified: 2026-09-25
---

# Room - Main Focus - Main Technique 1 - Main Technique 2

## TL;DR

> Required for full-machine or multi-stage labs.  
> Optional for focused exercises.

```
Initial access / starting point
    ↓
Technique / finding
    ↓
Technique / finding
    ↓
Privilege escalation / final objective
```

---

## Objective / Initial Context

Describe briefly:

- the starting point;
    
- any credentials or access supplied by the scenario;
    
- the target environment;
    
- the final objective.
    

Example:

```
Starting access:
User: <STARTING_USER>
Target: <TARGET_IP>

Objective:
Initial access
→ lateral movement
→ privilege escalation
→ root / SYSTEM
```

---

# 1. Reconnaissance

Explain what is being tested and why.

```
<COMMAND>
```

Relevant result:

```
<RELEVANT_OUTPUT>
```

Interpretation:

Explain what the result means and how it influences the next step.

---

# 2. Enumeration

Document the enumeration that actually contributes to the attack path.

```
<COMMAND>
```

Relevant result:

```
<RELEVANT_OUTPUT>
```

Explain:

- what was discovered;
    
- why it is interesting;
    
- what hypothesis it creates.
    

---

# 3. Initial Access

## Observation / Hypothesis

Explain the condition or behavior that may be exploitable.

## Validation

Use the smallest test that confirms or rejects the hypothesis.

```
<COMMAND>
```

Result:

```
<RELEVANT_OUTPUT>
```

## Interpretation

Explain:

- what has been confirmed;
    
- the vulnerability or technique involved;
    
- the security context obtained.
    

Verify the current identity when applicable:

```
whoami
id
```

Windows:

```
whoami
whoami /groups
```

---

# 4. Post-Exploitation / Lateral Movement

> Rename or remove this section if it does not apply.

For each identity transition, document:

```
Current identity
    ↓
Credential / trust / permission discovered
    ↓
Validation
    ↓
New identity
```

Command:

```
<COMMAND>
```

Relevant result:

```
<RELEVANT_OUTPUT>
```

Explain why the transition works.

---

# 5. Privilege Escalation

> Rename or remove this section if it does not apply.

## Enumeration

Document only the checks that contributed to the final privilege-escalation path.

```
<COMMAND>
```

Relevant result:

```
<RELEVANT_OUTPUT>
```

## Vulnerable Condition

Explain the trust boundary.

```
Low-privileged user controls X
        +
Privileged identity executes or trusts X
        ↓
Privilege Escalation
```

## Validation

Confirm that the privileged context really executes or trusts the controlled component.

```
<COMMAND>
```

Result:

```
<RELEVANT_OUTPUT>
```

## Exploitation

```
<COMMAND>
```

Verify:

```
whoami
id
```

Expected privileged context:

```
root / NT AUTHORITY\SYSTEM / privileged user
```

---

# 6. Final Objective

Document how the final objective was reached.

Example:

```
cat /root/root.txt
```

Result:

```
THM{REDACTED}
```

Do not publish real flags.

---

# Attack Chain

```
Reconnaissance
    ↓
Initial finding
    ↓
Initial access
    ↓
Lateral movement
    ↓
Privilege escalation
    ↓
Final objective
```

Keep this concise.

---

# Key Findings

Summarize the main vulnerabilities or weaknesses.

|Stage|Finding|Impact|
|---|---|---|
|Initial Access|||
|Lateral Movement|||
|Privilege Escalation|||

Remove rows that do not apply.

---

# False Leads / Useful Failures

Keep only failed attempts that influenced the investigation or taught something reusable.

Example:

```
A wordlist attack was exhausted without recovering the password.

This showed that further brute force was unlikely to be productive,
so enumeration resumed.
```

Delete this section if there are no useful failed paths.

---

# Key Takeaways

Document reusable lessons rather than only summarizing the room.

Examples:

```
Authentication does not imply authorization to every object exposed by an API.
```

```
A writable privileged script is only exploitable if a privileged context actually executes it.
```

```
A failed cracking attempt may be useful evidence that another attack path should be prioritized.
```

---

# Cleanup

> Include when exploitation modified the target.

Document:

- modified files restored;
    
- payloads removed;
    
- temporary accounts removed;
    
- altered ACLs / delegation restored;
    
- temporary tickets or artifacts removed.
    

Example:

```
rm -f /tmp/rootbash
cp ~/script.bak /opt/example/script.sh
```

If no cleanup was required, remove this section.