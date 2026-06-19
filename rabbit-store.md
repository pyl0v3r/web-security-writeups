# Rabbit Store — Penetration Test Report

## Objective and Scope

The objective of this assessment was to evaluate the security posture of the **Rabbit Store** target environment (TryHackMe), focusing on the web application's authorization model, server-side template handling, and the underlying RabbitMQ/Erlang messaging infrastructure.

## Scope of Work

**In-Scope Assets:**
- **Domain:** `cloudsite.thm` / `storage.cloudsite.thm`
- **Services:** Apache 2.4.52 (HTTP), OpenSSH 8.9p1, RabbitMQ (Erlang Port Mapper + Clustering)
- **Testing Perspective:** Authenticated low-privilege user (self-registered account)

---

## Enumeration

### Nmap Scan

```
Port    Service
22      SSH (OpenSSH 8.9p1)
80      Apache 2.4.52
4369    Erlang Port Mapper (epmd)
25672   RabbitMQ Clustering
```

The application redirected to `cloudsite.thm`, with a login portal at `storage.cloudsite.thm`. Self-registration was available.

### Tools Used
- Nmap
- Burp Suite
- Netcat
- Metasploit
- CyberChef
- JWT.io

---

## Vulnerabilities

| Vulnerability | Severity | Impact |
|---|---|---|
| Sensitive Authorization Data Exposed in Client-Side JWT | Medium | Disclosure of internal access-control logic |
| Mass Assignment — Subscription Privilege Escalation | High | Unauthorized access to premium/internal features |
| Server-Side Template Injection (SSTI) leading to RCE | Critical | Full remote code execution |
| Exposed Erlang Cookie Leading to RabbitMQ Cluster RCE | Critical | Root-equivalent access to messaging infrastructure |
| Recoverable Password Hash Disclosure (RabbitMQ Definitions Export) | High | Recovery of administrator credentials |

---

### Sensitive Authorization Data Exposed in Client-Side JWT

#### Severity
Medium

#### CWE
CWE-200: Exposure of Sensitive Information to an Unauthorized Actor

#### OWASP Category
OWASP Top 10 2021 – A04: Insecure Design

#### Description
The application issues a JSON Web Token (JWT) stored client-side that includes an authorization-relevant claim, `subscription`, indicating the account's access tier. Because this claim is both readable and (as shown in the next finding) influenceable by the client, it exposes internal authorization logic and creates an avenue for privilege manipulation.

#### Affected Functionality
- Session/authentication token issued at login

#### Steps to Reproduce
1. Authenticate to the application and extract the JWT from browser storage.
2. Decode the token using a JWT inspection tool (jwt.io).
3. Review claims for authorization-relevant fields.

#### Proof of Concept
Decoding the token revealed a `"subscription": "inactive"` claim, indicating that subscription/access-tier state is represented client-side within the token.

#### Impact
- Discloses internal access-control structure to any authenticated user
- Provides the reconnaissance needed to attempt the mass assignment attack described below

#### Root Cause
Authorization-relevant state is encoded into a client-readable token rather than being derived server-side from a trusted data store at request time.

#### Recommendation
- Do not encode authorization decisions (subscription tier, roles, entitlements) in client-readable tokens
- Treat JWT claims as identity assertions only; resolve authorization server-side against a trusted source
- Sign and validate tokens server-side; reject tokens with unexpected or tampered claims

#### References
- OWASP Top 10 2021 – A04: Insecure Design
- CWE-200

---

### Mass Assignment — Subscription Privilege Escalation

#### Severity
High

#### CWE
CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes

#### OWASP Category
OWASP Top 10 2021 – A01: Broken Access Control

#### Description
The user registration endpoint accepts a JSON body that is bound directly to internal account attributes without restricting which fields a client may set. By intercepting the registration request and adding a `"subscription": "active"` field, an attacker can self-assign a privileged subscription tier, gaining access to functionality (file upload) that should be restricted to paying/verified users.

#### Affected Functionality
- User registration endpoint

#### Steps to Reproduce
1. Register a new account with valid credentials, intercepting the request in Burp Suite.
2. Add an additional field to the JSON body: `"subscription": "active"`.
3. Forward the request and complete registration.
4. Log in with the new account and confirm access to subscription-gated features.

#### Proof of Concept
The newly registered account was granted `"subscription": "active"` status and was able to access the file upload functionality normally restricted to active subscribers — despite no payment or verification step being completed.

#### Impact
- Unauthorized escalation to a privileged account tier
- Grants access to functionality (file upload) that becomes the entry point for the SSTI/RCE finding below

#### Root Cause
The registration handler binds the full client-supplied JSON object to the user model without an allow-list of permitted fields, allowing clients to set attributes that should only be set server-side.

#### Recommendation
- Explicitly allow-list fields that may be set via user-controlled requests (e.g. username, email, password)
- Never bind request bodies directly to ORM/data models without a defined schema
- Set privilege/subscription attributes only via authenticated, server-side business logic (e.g. payment confirmation webhook)

#### References
- OWASP Top 10 2021 – A01: Broken Access Control
- CWE-915

---

### Server-Side Template Injection (SSTI) Leading to Remote Code Execution

#### Severity
Critical

#### CWE
CWE-1336: Improper Neutralization of Special Elements Used in a Template Engine

#### OWASP Category
OWASP Top 10 2021 – A03: Injection

#### Description
With privileged access gained via mass assignment, an internal API route (`/api/fetch_messages_from_chatbot`, discovered via API documentation reachable through the file upload feature) was found to render user-supplied input through a server-side template engine without sanitization. Submitting a template expression as the `username` field caused the expression to be evaluated server-side, confirming SSTI. This was escalated to full remote code execution by injecting a payload that invoked OS command execution via the template engine's underlying Python runtime.

#### Affected Functionality
- `/api/fetch_messages_from_chatbot` (POST, JSON body)

#### Steps to Reproduce
1. Send a POST request to `/api/fetch_messages_from_chatbot` with `Content-Type: application/json` and body `{"username":"{{7*7}}"}`.
2. Observe the response evaluates the expression (returns `49`), confirming SSTI.
3. Craft a payload using the template engine's introspection capabilities to reach `os.popen` via `__globals__`/`__builtins__`, executing a reverse shell one-liner.
4. Start a netcat listener on the attacker machine and submit the payload.

#### Proof of Concept
```json
{"username":"{{request.application.__globals__.__builtins__.eval('__import__(\"os\").popen(\"<reverse-shell-command>\").read()')}}"}
```
Submitting this payload to the chatbot endpoint resulted in an interactive reverse shell connecting back to the attacker-controlled listener, confirming arbitrary OS command execution.

#### Impact
- Full remote code execution in the context of the application's service account
- Direct access to the underlying host filesystem, enabling discovery of the Erlang cookie (see next finding)

#### Root Cause
User-controlled input is passed directly into a server-side template rendering call without sanitization, and the template engine's sandboxing (if any) does not prevent access to Python builtins via object introspection.

#### Recommendation
- Never render user-supplied input as a template; treat all user input as data, not template source
- If dynamic templating of user content is required, use a logic-less template language with no access to the host environment
- Apply strict input validation on API parameters and reject template-syntax characters (`{{`, `{%`, etc.)
- Run application services with least-privilege OS accounts to limit the blast radius of RCE

#### References
- OWASP Top 10 2021 – A03: Injection
- CWE-1336

---

### Exposed Erlang Cookie Leading to RabbitMQ Cluster RCE

#### Severity
Critical

#### CWE
CWE-522: Insufficiently Protected Credentials

#### OWASP Category
OWASP Top 10 2021 – A05: Security Misconfiguration

#### Description
With code execution on the host, the RabbitMQ data directory (`/var/lib/rabbitmq/`) was found to contain a `.erlang.cookie` file readable by the compromised account. This cookie is the shared-secret authentication mechanism for Erlang's distributed node protocol. Using this cookie, the Metasploit `erlang_cookie_rce` module was used to authenticate to the RabbitMQ/Erlang cluster interface and obtain a shell as the `rabbitmq` service user.

#### Affected Functionality
- `/var/lib/rabbitmq/.erlang.cookie`
- RabbitMQ clustering port (25672)

#### Steps to Reproduce
1. From the SSTI-derived shell, read `/var/lib/rabbitmq/.erlang.cookie`.
2. Configure the Metasploit module `exploit/multi/misc/erlang_cookie_rce` with the target host, RabbitMQ clustering port (25672), the recovered cookie value, and attacker `LHOST`.
3. Run the exploit to obtain a shell as `rabbitmq@forge`.

#### Proof of Concept
The Metasploit `erlang_cookie_rce` module successfully authenticated using the recovered cookie and returned an interactive shell as the `rabbitmq` service account.

#### Impact
- Full administrative access to the RabbitMQ cluster
- Code execution as the `rabbitmq` service account, with access to RabbitMQ configuration and credential stores

#### Root Cause
The Erlang cookie file was readable by an account other than the RabbitMQ service owner, and the RabbitMQ clustering interface was reachable and authenticatable using that cookie without additional network-level restriction.

#### Recommendation
- Restrict file permissions on `.erlang.cookie` to the RabbitMQ service account only (`chmod 400`, correct ownership)
- Restrict network access to the Erlang Port Mapper (4369) and RabbitMQ clustering port (25672) to trusted cluster nodes only
- Rotate the Erlang cookie if exposure is suspected
- Apply least-privilege OS permissions so that a compromised web application account cannot read other services' credential material

#### References
- OWASP Top 10 2021 – A05: Security Misconfiguration
- CWE-522

---

### Recoverable Password Hash Disclosure via RabbitMQ Definitions Export

#### Severity
High

#### CWE
CWE-916: Use of Password Hash With Insufficient Computational Effort

#### OWASP Category
OWASP Top 10 2021 – A02: Cryptographic Failures

#### Description
With access as the `rabbitmq` service account, the RabbitMQ configuration was exported via `rabbitmqctl export_definitions`, revealing the stored password hash for the root/administrator user. Analysis of RabbitMQ's hashing scheme (`rabbit_password_hashing_sha256`, structured as `Base64(SALT + SHA256(SALT + password))`) allowed the salt to be isolated and the original password to be recovered.

#### Affected Functionality
- RabbitMQ user definitions export
- RabbitMQ administrator account credentials

#### Steps to Reproduce
1. From the `rabbitmq` shell, export cluster definitions:
   ```bash
   rabbitmqctl export_definitions --node rabbit@forge thm.json
   ```
2. Locate the `password_hash` field for the administrator account in the exported JSON.
3. Base64-decode the hash and isolate the first 8 bytes as the salt (per RabbitMQ's documented hashing scheme).
4. Use the salt to reverse the SHA256 hashing process (assisted by CyberChef) and recover the plaintext password.

#### Proof of Concept
The exported definitions contained a `password_hash` value for the administrator account. Following RabbitMQ's documented `SALT + SHA256(SALT + password)` construction, the salt was isolated and the administrator's plaintext password was recovered, granting root-level credentials.

#### Impact
- Full recovery of administrator credentials
- Complete compromise of the RabbitMQ management plane and any systems sharing this credential

#### Root Cause
RabbitMQ's default password hashing scheme (single-round SHA256 with salt) is computationally cheap and, combined with an exposed export containing the hash, allows credential recovery without brute-forcing.

#### Recommendation
- Restrict access to `rabbitmqctl` and definitions exports to trusted administrative accounts only
- Enforce strong, unique passwords for administrative accounts to resist offline recovery even if hashes are disclosed
- Avoid credential reuse between the RabbitMQ administrator account and other system accounts
- Monitor and alert on `rabbitmqctl export_definitions` usage

#### References
- OWASP Top 10 2021 – A02: Cryptographic Failures
- CWE-916

---

## Attack Chain Summary

1. Registered a low-privilege account and identified a JWT exposing internal `subscription` state
2. Exploited mass assignment to self-escalate to an "active" subscription, unlocking file upload functionality
3. Discovered a hidden chatbot API endpoint via upload-related API documentation
4. Identified and exploited SSTI in the chatbot endpoint to achieve remote code execution
5. Recovered the RabbitMQ Erlang cookie from the compromised host
6. Used the Erlang cookie with Metasploit to gain a shell on the RabbitMQ cluster
7. Exported RabbitMQ definitions and recovered the administrator password hash
8. Reversed the salted SHA256 hash to recover administrator credentials and obtained root access

## Key Takeaways

- JWT claims should never encode authorization decisions that the client can influence
- Mass assignment vulnerabilities can turn low-privilege accounts into privileged ones with a single extra JSON field
- SSTI is a direct path to RCE when template engines expose Python builtins
- `.erlang.cookie` files must be tightly permissioned — they are equivalent to root credentials for the Erlang cluster
- RabbitMQ's default password hashing is reversible with knowledge of the salt structure and should not be relied upon if hashes may be exposed